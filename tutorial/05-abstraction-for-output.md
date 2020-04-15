---
layout: default
title: An abstraction for output
---

现在，`HelloWorldCommand` 使用 `System.out.println()` 来进行输出。
本着依赖注入的精神，让我们使用一个抽象，
以便我们可以删除对 `System.out` 的直接使用。
我们要做的是创建一个 `Outputter` 类型，它可以对写入的文本执行某些操作。
我们的默认实现仍然可以使用 `System.out.println()`，
但这给了我们更多灵活性，以后的更改无需修改 `HelloWorldCommand` 。
例如，我们的测试可能会使用一个将字符串添加到
`List<String>` 的实现，所以我们能检查输出了什么。

这是我们的 `Outputter` 类型：

```java
interface Outputter {
  void output(String output);
}
```

这是我们在 `HelloWorldCommand` 中使用它的方式：

```java
private final Outputter outputter;

@Inject
HelloWorldCommand(Outputter outputter) {
  this.outputter = outputter;
}

@Override
public Status handleInput(List<String> input) {
  outputter.output("world!");
  return Status.HANDLED;
}
```

`Outputter` 是一个接口。我们可以写一个它的实现，
给这个类一个 [`@Inject`] 构造函数，然后使用 [`@Binds`] 将 `Outputter` 绑定到
该实现。但是 `Outputter` 很简单…很简单，我们可以
甚至将其实现为 lambda 或方法引用。因此，与其做所有这些，
让我们编写一个 `static` 方法，该方法仅创建并返回
`Outputter` 的一个实例本身！

```java
@Module
abstract class SystemOutModule {
  @Provides
  static Outputter textOutputter() {
    return System.out::println;
  }
}
```

在这里，我们创建了另一个 [`@Module`]，但是我们使用的不是 [`@Binds`] 方法，是 [`@Provides`] 方法。 
[`@Provides`] 方法的工作原理与 [`@Inject`] 构造函数很像：
它在这里告诉 Dagger 当它需要一个 `Outputter` 实例时，
dagger 应该 _调用_ `SystemOutModule.textOutputter()` 从而获取一个。

同样，我们需要将新模块添加到组件中以告知
dagger 说它应该在我们的应用程序中使用该模块：

```java
class CommandLineAtm {
  ...

  @Component(modules = {HelloWorldModule.class, SystemOutModule.class})
  interface CommandRouterFactory {
    CommandRouter router();
  }
}
```

再次，我们的应用程序的 _行为_ 并没有改变，
但是现在很容易为我们的命令编写单元测试，
而无需实际执行写入`System.out`。

> **概念**
>
> *   **[`@Provides`]** 方法是模块中的具体方法，可以告诉
>     dagger，当某些东西请求该方法的返回类型的实例时，
>     ​​则应调用该方法以获取实例。
>     像 [`@Inject`] 构造函数，它们可以有参数：
>     这些参数是它们的依赖。
> *   [`@Provides`] 方法可以包含任意代码，只要它们返回一个
>     提供的类型的实例。他们不需要在每次调用创建新实例。
>     *   这突出了 Dagger （和整个依赖注入）的一个重要方面：
>         当请求类型时，是否输入新的
>         实例以满足该请求是实现细节
>         之后，我们将使用术语 `provided` 代替 `created`，
>         因为它对正在发生的情况描述的更准确。

<section style="text-align: center" markdown="1">

[Previous](04-depending-on-interface) · [Next](06-new-command)

</section>

[`@Binds`]: https://dagger.dev/api/latest/dagger/Binds.html
[`@Inject`]: http://docs.oracle.com/javaee/7/api/javax/inject/Inject.html
[`@Module`]: https://dagger.dev/api/latest/dagger/Module.html
[`@Provides`]: https://dagger.dev/api/latest/dagger/Provides.html
