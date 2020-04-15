---
layout: default
title: 接口依赖
---

但是等等: 为什么 `CommandRouter` 的构造函数明确的需要一个 `HelloWorldCommand`？
它不应该可以使用任何一种 `Command` 吗？

让我们将 `CommandRouter` 的依赖项指定为普通的 `Command`：

```java
@Inject
CommandRouter(Command command) {
  ...
}
```

但是现在 Dagger 不知道如何获取 `Command` 的实例。如果您尝试
编译，Dagger 报告错误。由于 `Command` 是 _接口_ 并且不能
有 [`@Inject`] 构造函数，我们需要给 Dagger 更多信息。

为此，我们可以编写一个带有 [`@Binds`] 注解的方法：

```java
@Module
abstract class HelloWorldModule {
  @Binds
  abstract Command helloWorldCommand(HelloWorldCommand command);
}
```

这个 [`@Binds`] 方法告诉 Dagger，当某些东西依赖于 `Command` 时，
Dagger 应在其位置提供一个 `HelloWorldCommand` 对象。请注意
方法的返回类型，`Command`，这是 Dagger 现在知道如何
提供的类型，并且参数类型也是 Dagger 知道
当某些东西依赖于`Command` 时使用的类型。

该方法是抽象的，因为仅它的声明就足以告诉Dagger
该怎么办。 Dagger 实际上不会调用此方法或提供
实现类。

请注意，[`@Binds`] 方法是声明在带 [`@Module`] 注解的类中。
Module 是绑定方法（带 [`@Binds`] 注解的方法 或其他一些注解，
我们将在后面看到）的集合，这些注解告诉 Dagger
如何提供实例。与 [`@Inject`] 直接用在类的构造函数上不同，
[`@Binds`] 方法必须在 Module 内部。

为了告诉 Dagger 在 `HelloWorldModule` 中寻找 [`@Binds`] 方法，我们将
它添加到 [`@Component`] 注解中。

```java
@Component(modules = HelloWorldModule.class)
interface CommandRouterFactory {
  CommandRouter router();
}
```

> **Aside:** 您可能想知道, 为什么我们不需要 [`@Module`]
> 告诉 Dagger 我们也需要 [`@Inject`] 注释的类。
> 答案是 Dagger 已经知道要查看这些类型，因为它们出现了
> Dagger 使用的 component 或 module 中的某处。比如
> `CommandRouter`，这是`CommandRouterFactory` 入口方法的返回类型
> 再比如 `HelloWorldCommand`，它是我们刚刚在 `HelloWorldModule`
> 中编写的 [`@Binds`] 方法的参数类型。在那之前
> 它作为 `CommandRouter` 的构造函数参数出现，因此 Dagger
> 在查看 `CommandRouter` 时可传递性的了解到它。

现在，当 Dagger 要创建`CommandRouter` 并看到它需要一个
`Command`，它将使用 `HelloWorldModule` 中的指令创建一个。

进行此更改后，我们的应用程序应像以前一样继续工作，
但是我们的 `CommandRouter` 类不再被迫仅使用一个
类型的 `Command`。

> **概念**
>
> *   **[`@Module`]** 是类或接口，充当
>     Dagger 如何构造依赖项的指令集和 。他们被称为
>     module，因为它们是 _modular模块化_ 的：您可以在不同的应用或上下文中
>     混合、匹配 module。
> *   **[`@binds`]** 方法是告诉 Dagger 如何构造
>     实例。它们是模块中的抽象方法，
>     将 Dagger 已经知道如何构造的类型（方法的参数） 与
>     Dagger 尚不知道如何构造的类型（该方法的返回值
>     类型）进行关联。

<section style="text-align: center" markdown="1">

[Previous](03-first-command) · [Next](05-abstraction-for-output)

</section>

[`@Binds`]: https://dagger.dev/api/latest/dagger/Binds.html
[`@Component`]: https://dagger.dev/api/latest/dagger/Component.html
[`@Inject`]: http://docs.oracle.com/javaee/7/api/javax/inject/Inject.html
[`@Module`]: https://dagger.dev/api/latest/dagger/Module.html
