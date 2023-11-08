* https://github.com/kubernetes-sigs/controller-runtime/issues/2513
  * 在 controller-runtime 0.15.0 中，调用 client.Get 后使用 errors.IsUnauthorized(err) 检查 Unauthorized 错误不再有效，可能是由于错误被包装导致的问题。

* https://github.com/kubernetes-sigs/controller-runtime/pull/2432
  * 高 CPU 消耗的原因是 mutating webhook 在生成 oldObj 和 newObj 之间的差异的 jsonpatch 数组时，无论它们是否相等，都会对它们的字段进行 reflect.DeepEqual 操作。这个操作会消耗大量的 CPU 时间。然而，在大多数情况下，oldObj 和 newObj 是相同的。因此，在生成 jsonpatch 数组之前，比较oldObj和newObj的字节数组可以节省大量的CPU周期。这是一个示例配置文件。
