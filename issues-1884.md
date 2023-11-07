# 在运行时添加和删除 watche

为了提升我在这里的问题的可见性，我开启了一个新的议题。

我们有一个和链接中的议题非常类似的使用场景，我们想知道如何实际在运行时删除监视器。

---

@jcanseco，你能描述一下你需要动态移除观察器（informers）的使用案例吗？

---

@FillZpp 当然。

假设你有一个资源 A 依赖于资源 B。我们希望一旦资源 B 准备就绪，立即对资源 A 进行调节。

我们希望通过让资源 A 的 Controller 在资源 B 上创建一个监视器来实现这一点，这样我们可以在资源 B 准备就绪后立即将资源 A 加入队列进行调节。完成后，我们希望删除监视器，因为它不再需要。

如果有更好的解决这个问题的方法，也请告诉我们。

---

A 和 B 是资源种类还是对象名称？

我猜测它们是资源种类。所以你首先监视两种类型，但你只关心类型 B 的一个对象。在监视了类型 B 的唯一对象之后，你的 controller 触发对类型 A 的调和，不再需要监视类型 B。

我理解的对吗？

---

啊，是的，对不起，我没有表述得足够准确。你的理解是正确的。

要特别准确：资源 A 和 B 是两个特别的对象（即，由他们的 GVK、命名空间和名称的组合唯一标识）。

在我们的情况下，资源 A 和 B 的 GVKs 通常是不同的。

---

我明白了。另一个问题是，资源 B 只有一个特定的对象吗？还是资源 B 有多个对象，但你只需要监视其中的一个特定对象？

这样如何：

如果资源 B 只有一个对象，我认为你不必停止并移除它的通知器。
如果有多个对象，你可以使用 SelectorByObject 来确保你的缓存只监视特定的对象，然后就不必停止和移除它。

---

你说的资源 B，是指 GVK 吗？如果是的话，是的，可以有多个具有 GVK B 的对象，但我们只需要监视一个特定的对象（即，特定的 GVK + 命名空间 + 名称组合）。

我不太确定我理解了你的建议 -- 你能详细说明一下吗？

也许另一个重要的信息是：我们有一个带有多个 controller 的 controller 管理器。每个 controller 管理所有具有特定 GVK 的对象。
我们的 controller 管理器管理 GVK A 和 GVK B。所有的 controllers，据我理解，共享同一个缓存。

---

哦，所以你的操作器中有多个 controllers，不同的 controllers 可能关心资源 B 的不同对象。以这种方式，你并不打算停止或删除 B 的整个通知器，而只是想为一个 controller 删除一个监视器？

然而，通知器中没有办法移除监听器，你无法移除监视器。你唯一能做的就是在得到特定对象后，忽略监视器处理器中的所有事件。

---

我们的操作器中有多个 controllers，不同的 controllers 关心不同的资源（GVKs）。所以有一个 controller 管理资源 A 的所有对象，有一个 controller 管理资源 B 的所有对象，等等。

我们希望 Controller A 创建一个临时的对 Object B 的监视器，以便知道何时将 Object A 加入队列以立即进行调和。

-

我也得出了相同的结论，但是建议的解决方案会不会有问题，因为它实际上会导致内存泄漏？注意，我们将要创建大量的这些临时监视器。

你对这个想法怎么看：

让 Controller A 通过一个自定义 Source 监视 Object B。
这个自定义 Source 很像 source.Kind，但它创建了自己的新 SharedInformer，并且它包含一个可以从 Source 外部发出信号的停止通道。
一旦 Object B 准备好，事件处理器会发出信号让 Source 停止。然后，Source 停止它的 SharedInformer。
这是否可行？我不确定你是否可以创建一个全新的 SharedInformer，完全独立于 controllers 使用的那些，我也不确定如果只是临时的，创建大量独立的 SharedInformers 是否是个好主意。

---

@jcanseco 如果是这样，你可能只需要使用带有通用事件通道的通道源，从 Controller B 触发 Controller A。没有必要创建一个信号通知器。

```go
// a global channel
var triggerChan = make(chan event.GenericEvent, 1)

// Controller A, watches the channel source and enqueue Resource A in the handler.
// Note that the event will be generic event and should be handled by Generic() of handler.
...Watches(&source.Channel{Source: triggerChan}, &handler)

// Controller B, once Object B is ready, put it into channel with Once
var once sync.Once
once.Do(func() { triggerChan <- event.GenericEvent{Object: ...} })
```

---

另外，一个不同的问题：

我知道你说我们应该考虑使用 source.Channel，而不是为每个临时监视器创建一个新的 SharedInformer -- 我们肯定会考虑这一点。

然而，作为备选选项，我仍然好奇，每次我们想要创建一个临时监视器时，是否有可能并且明智地创建新的 SharedInformers，就像我在上一条评论中描述的那样？我们已经有了一个这样做的原型，所以我想知道我们是否可以把它作为一个潜在的备选选项。

---

嗯... 所以只有 Controller A 或者多个 controller 都在等待资源 B 准备就绪？

如果有多个 controllers，你可以在每个 controller 中等待临时通道关闭，这意味着资源 B 已经准备就绪。

```go
// There is a global channel which will be closed when Resource B is Ready.
var tempChan = make(chan struct{})

// Controller B, close the tempChan only once when Resource B is Ready.
once.Do(func() { close(tempChan) })

// In each controller that wait for B Ready, use func channel and wait for tempChan to be closed
Watches(source.Func(func(ctx context.Context, _ handler.EventHandler, q workqueue.RateLimitingInterface, _ ...predicate.Predicate) error {
    go func() {
        select {
        case <-tempChan:  // wait for channel to be closed
        case <-ctx.Done():
            // cache has been canceled or closed
            return
        }
        q.Add(...)  // enqueue Resource A or any other resources for other controllers
    }()
    return nil
}), &handler.EnqueueRequestForObject{})
```

我并不真的建议你为每个临时监视器创建一个新的 SharedInformer，这会对 apiserver 和操作器带来压力。但这取决于你的场景，我对你的 controllers 和资源之间的关系了解不多。

---

jcanseco

对于这种混淆，我感到抱歉。也许用具体的例子来说明会更有帮助。我们是 Config Connector -- 一个用于 Google Cloud 资源的 operator。我们有多种资源类型 -- 每一种都由一个 controller 管理。所以例如，有一个 controller 用于 PubSubSubscription 对象，另一个用于 PubSubTopic 对象。

对象可以引用其他对象，形成一个有向无环图（DAG）。例如，一个 PubSubSusbcription 可以引用多个 PubSubTopics，多个 PubSubSubscriptions 可以引用同一个 PubSubTopic。

以这个例子为例：PubSubSubscription 需要等待两个 PubSubTopics 都准备就绪。但也可能有另一个 PubSubSubcription 需要等待那一个或两个 PubSubTopics。

-
回到建议的解决方案：我认为这不适用于我们，因为我们的 controllers（例如，controller A）需要在其生命周期中能够临时监视各种资源种类的各种对象（例如，资源种类 B 的各种对象），所以一旦被关闭的全局通道是不行的。我们需要一个更通用的解决方案。

实际上，现在想想，仅仅使用通道在 controllers 之间进行协调是不行的，因为我们允许用户创建多个在不同 pods 中运行的 controller managers 实例，用于管理不同的命名空间，所以我们需要能够跨进程边界进行协调。

我提出了一个可行的原型，其中：

- 每个 controller 都监视一个 source.Channel。
- 当 controller 遇到一个非就绪的依赖项时，它创建一个临时的 Watch（通过创建一个新的动态客户端，而不是一个新的 SharedInformer）。
- 然后，一旦依赖项准备就绪，监视 Watch 的 goroutine 将一个 GenericEvent 添加到 source.Channel，以触发引用资源的调和，然后停止 Watch。

这是可行的。我们担心的是，每当一个依赖项没有准备好时，我们都必须创建临时的监视器 -- 我认为这可能是可以的，因为我们的监视器应该是短期存在的，并且只在用户创建新资源时真正需要。我们也将节省大量的快速，重复的 Get 请求，这就是我们现在每次依赖项没有准备好时做的（由于失败的调和被以指数退避的方式重新加入队列）。

不过你怎么看呢？监视器是否那么昂贵以至于需要关心？理想情况下，我们希望依赖于使用 controller manager 的 SharedInformer 来共享监视器（至少在同一进程内），但是正如你所说，现在没有办法移除事件处理器。

---

FillZpp

实际上，我对 metacontroller 有类似的需求 - 它允许动态添加/删除用户想要监视的 kinds。
现在它正在使用一些 SharedInformers 的内部实现，如果这个功能被添加，
我会很乐意切换到一些常见库（client-go 或 controller runtime）的东西。

---

实际上（我是从我的手机上输入的），我希望得到的是一样的，在 metacontroller 中，RemoveEventHandler 是唯一添加到接口的方法 -
https://github.com/metacontroller/metacontroller/blob/master/pkg/dynamic/informer/informer.go#L51

---

好的，我明白了。这和 metacontroller 的 RemoveEventHandlers() 有一点不同，metacontroller 的 RemoveEventHandlers() 只会移除你自己的处理器，
而底层共享的 informer 中的其他处理器不会被移除。但在 controller-runtime 中，其 informer 只被同一进程中的控制器共享，一个控制器不能移除由多个控制器注册的所有处理器。
而且，一个控制器如何声明移除其自己的处理器，如果我们打算在 client-go 和 controller-runtime 中支持这个功能，这将是一个问题。

---

我有一个类似的需求，其中 Controller-A 监视某些事件并动态创建如下额外的控制器：
```go
ctrlruntime.NewControllerManagedBy(manager).Named("test").For(&Unstructured{}).Complete(reconciler) 
```
我如何删除这个控制器，以便我不再接收该对象的调和？我正在寻找一个像下面这样的函数：
```go
ctrlruntime.RemoveControllerManagedBy(manager, controller)
```
任何帮助将不胜感激。

---

允许从 SharedIndexInformer 中移除单个事件处理器的 PR 已在此处合并：

kubernetes/kubernetes#111122

抄送 @FillZpp

---

@alexzielenski 那太好了，谢谢！我本来打算帮忙看看那个 PR，但这几个月我太忙了...

所以我们只需要等待 client-go 的新版本标签，然后它就可以在 c-r 中使用了。

/remove-lifecycle stale

---

@FillZpp v0.25.2 应该包含 remove 方法

---

嘿 @FillZpp 👋 感谢你帮助解决这个问题！现在新的 client-go 发布了，c-r 的更改已经完成了吗？
我的使用场景和 @rnsv 一样，我们动态运行 ctrlruntime.NewControllerManagedBy(manager). ..... Complete(reconciler) 并希望在完成后能够从管理器中移除 reconciler。谢谢！

---

是的。我也遇到了这个问题。我也在等待这个特性。

---

@FillZpp 对这个有任何更新吗 😄 ? 如果你能指出我应该在哪里工作，我很乐意尝试实现这个 😄

---

https://github.com/kubernetes-sigs/controller-runtime/pull/2099

@aclevername 我已经发布了 #2099，但它仍然需要一些更多的测试，我稍后会添加它们。

各位 @jcanseco @grzesuav @rnsv @jnan806，K8s 1.26 已经发布，我们终于可以实现这个目标了。你们也可以帮忙审查 PR，看看它是否能满足你们的需求。

---

@FillZpp 谢谢！从控制器的角度看，它看起来是正常的。在 Source 的情况下，一个如何使用它的例子将会很有用，我对 controller runtime 的源代码不是很熟悉，无法评估它。

---

@FillZpp 感谢你的工作，我期待着 PR 的合并。
如果你认为我可能能帮助你，请联系我。

---


