# 在 runtime 支持动态添加新的 watch

这个特性在之前已经添加过（并且在 #246 中被请求），但是更新的 Manager & Builder 模式不再支持它，
因为它们负责创建 Controller 而不是由用户编写的代码来执行，而且 Controller 不对你的 reconciler 可访问。

关于这个问题在 Slack (https://kubernetes.slack.com/archives/CAR30FCJZ/p1561759175084100) 上有过讨论。
我正在考虑我们可以像这样修改 inject 包（名字待定 - 我知道它们并不 100% 遵循当前的模式，但我们需要缩小 Controller 接口的范围以避免导入循环）：

```go
type Watcher interface {
  Watch(src source.Source, eventhandler handler.EventHandler, predicates ...predicate.Predicate) error
}

type WatcherInjector interface {
  InjectWatcher(watcher Watcher)
}
```

你认为像这样的东西能行吗，@DirectXMan12？

---

我们趋向于在内部严格不需要时不再添加更多的注入器 -- 在大多数情况下，通过组织你的应用程序使你能够直接将它们传递给你的 reconcilers 来更明显地解决问题（就像我们在 KB v2 中所做的）。

因此，我更倾向于看到一种解决方案，使得可以以某种方式通过 builder 获取到底层 controller 的处理。如果我们无法清楚地找出如何做到这一点，一个注入器也是可以的，但我更倾向于更直接的方法。

---

好的，我会进行一些头脑风暴，看看我能想出什么来。

---

@DirectXMan12 如果我们修改 builder.Complete() 使其返回刚刚创建的 controller 呢？
然后用户可以对其进行任何他们想要的操作（包括将其传递给他们的 reconciler）。

---

或者在 Builder 上暴露一个 GetController() 方法。

---

+1 对于拥有获取 controller 的方法或者让 Complete() 返回创建的 controller 和错误。今天，kubebuilder 调用 ctrl.NewControllerManagedBy(...)，通常我认为它会返回一个我可以使用的新的 Controller 对象。

---

我更倾向于不使用 Complete（如果可以的话，只是为了避免常用函数的破坏），但我支持一个类似的方法。由于 Build 已经被废弃了几个版本，我们可以重新利用它（尽管这可能导致破坏）或者想出一个新方法。

---

我不认为我们想要两种不同的完成/构建方式 - 一种返回 controller，一种不返回 - 我们会这样想吗？

---

概括我们昨天在 Slack 上的讨论（https://kubernetes.slack.com/archives/CAR30FCJZ/p1564433195123400），我们将取消 Build 的废弃，并让它返回 Controller 而非 Manager。我们还将删除 builder 中的逻辑，该逻辑在你不传入 manager 时为你创建一个 manager。
