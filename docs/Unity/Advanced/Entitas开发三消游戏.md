# Entitas开发三消游戏

本章将开发一个相对复杂些的游戏示例，如果各位对于 ECS 的概念不是很了解，那么建议先看看我之前写的入门笔记。

## Entitas框架常见问题

---

### 标志组件的更新问题

标志组件在值相同的情况下是不会响应的：

```csharp
public bool isGameEliminate {
    get { return HasComponent(GameComponentsLookup.GameEliminate); }
    set {
        // 如果值相同则下面的代码将不会执行
        if (value != isGameEliminate) {
            var index = GameComponentsLookup.GameEliminate;
            if (value) {
                var componentPool = GetComponentPool(index);
                var component = componentPool.Count > 0
                        ? componentPool.Pop()
                        : gameEliminateComponent;

                AddComponent(index, component);
            } else {
                RemoveComponent(index);
            }
        }
    }
}
```

### 系统的调用顺序问题

在 Entitas 的开发过程中，我们经常会编写各类响应式系统以及监听方法，而这些系统的先后调用顺序将会对游戏的流程产生影响。

### Group遍历问题

直接遍历 Group 会导致死循环，必须新建一个 List 来进行遍历。