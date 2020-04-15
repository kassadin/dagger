---
layout: default
title: 添加第一个`Command`
---

如果您到目前为止已尝试运行该示例，则会看到它不知道如何
响应命令。让我们改变它！

首先，让我们实现一个 `Command`:

```java
final class HelloWorldCommand implements Command {
  @Inject
  HelloWorldCommand() {}

  @Override
  public String key() {
    return "hello";
  }

  @Override
  public Status handleInput(List<String> input) {
    if (!input.isEmpty()) {
      return Status.INVALID;
    }
    System.out.println("world!");
    return Status.HANDLED;
  }
}
```

现在，为这个实现给`CommandRouter` 的构造函数添加一个参数：

```java
final class CommandRouter {
  private final Map<String, Command> commands = new HashMap<>();

  @Inject
  CommandRouter(HelloWorldCommand helloWorldCommand) {
    commands.put(helloWorldCommand.key(), helloWorldCommand);
  }

  ...
}
```

这个参数告诉 Dagger，当它创建一个 CommandRouter 实例时，它
还应提供一个`HelloWorldCommand` 实例，并将其传递给
构造函数：`new CommandRouter(helloWorldCommand)`。Dagger 知道如何创造
一个 `HelloWorldCommand`，因为它具有 [`@Inject`] 构造函数，就像
`CommandRouter`。

如果尝试运行该应用程序，将会看到您现在可以输入 `hello` 并且
应用程序将响应 `world!`。我们正在进步！

> **概念**
>
> *   [`@Inject`] 构造函数的参数是这个类的依赖。
>     Dagger 将提供类的依赖关系以实例化该类
>     本身。请注意，这是递归的：依赖项可能是其
>     自身！
> *   **术语：**
>     *   在讨论这两种类型之间的关系时，
>         可以说`CommandRouter` _要求_ `HelloWorldCommand` 或 `CommandRouter`
>         _依赖于_ `HelloWorldCommand`。相反的，`HelloWorldCommand` 的
>         `@Inject` 构造函数 _提供_ `CommandRouter` _要求_ 的
>        `HelloWorldCommand` 实例。
>     *   有时人们会说 `CommandRouter` _注入_ `HelloWorldCommand`是
>         为了强调将一个`HelloWorldCommand`
>         对象放入 `CommandRouter` 中，而不是 `CommandRouter` 本身
>         获取或创建一个。但也常说 _Dagger_
>         注入`CommandRouter`，因为它是通过 [@Inject`] 构造函数实例化的。
>         `注入`一词的这些不同用法可能
>         令人困惑，因此我们在本教程中使用更明确的术语。

<section style="text-align: center" markdown="1">

[Previous](02-initial-dagger) · [Next](04-depending-on-interface)

</section>

[`@Inject`]: http://docs.oracle.com/javaee/7/api/javax/inject/Inject.html
