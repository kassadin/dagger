---
layout: default
title: Two commands for the price of one
---

到目前为止，`CommandRouter` 一次仅支持一个命令，
但我们希望使它支持许多命令。

您会注意到，如果您尝试将 **两个** 模块一起添加
(`@Component(modules = {HelloWorldModule.class, LoginCommandModule.class})`),
Dagger 将报告错误。这两个模块发生冲突-它们各自告诉 Dagger
创建单个 `Command`，而 Dagger 不知道该选择哪个。

我们希望 `CommandRouter`依赖于 _多个_ 命令而不是一个。
由于我们的 `CommandRouter` 需要一个命令的映射，因此我们将使用 [`@IntoMap`]
将应用程序使用的每个 `Command` 映射到命令的前缀：

```java
@Module
abstract class LoginCommandModule {
  @Binds
  @IntoMap
  @StringKey("login")
  abstract Command loginCommand(LoginCommand command);
}
```

```java
@Module
abstract class HelloWorldModule {
  @Binds
  @IntoMap
  @StringKey("hello")
  abstract Command helloWorldCommand(HelloWorldCommand command);
}
```

[`@StringKey`] 注解与 [`@IntoMap`] 结合使用，告诉 Dagger 如何
填充一个 `Map<String, Command>`。请注意，我们的 `Command` 接口不再
需要一个 `key()` 方法，因为我们直接告诉 Dagger 键是什么。

要利用这个优势，我们可以切换 `CommandRouter` 的构造函数参数
到 `Map<String, Command>`。注意，`Command` 本身将不再起作用。

```java
final class CommandRouter {
  private final Map<String, Command> commands;

  @Inject
  CommandRouter(Map<String, Command> commands) {
    // This map contains:
    // "hello" -> HelloWorldCommand
    // "login" -> LoginCommand
    this.commands = commands;
  }

  ...
}
```

如果现在运行该应用程序，您将看到 `hello` 和 `login <your
name>` 都可以工作。确保更新 [`@Component`] 注解以包含这
两个模块。

> **概念**
>
> *   **[`@IntoMap`]** 允许创建具有特定类型值的 map，
>     并使用特殊注解（例如 [`@StringKey`] 或 [`@IntKey`] ）设置键。 
>     由于键是通过注解设置的，因此 Dagger 确保不会将多个值映射到同一键。
> *   **[`@IntoSet`]** 允许创建一组要收集在一起的类型。
>     它可以和 [`@Binds`] 和 [`@Provides`] 方法一起使用来提供 `Set<ReturnType>`。
> *   [`@IntoMap`] 和 [`@IntoSet`] 通常称为“multibindings”，
>     集合中包含来自几种不同绑定方法的元素。

<section style="text-align: center" markdown="1">

[Previous](06-new-command) · [Next](08-user-specific-types)

</section>

[`@Binds`]: https://dagger.dev/api/latest/dagger/Binds.html
[`@Component`]: https://dagger.dev/api/latest/dagger/Component.html
[`@IntKey`]: https://dagger.dev/api/latest/dagger/multibindings/IntKey.html
[`@IntoMap`]: https://dagger.dev/api/latest/dagger/multibindings/IntoMap.html
[`@IntoSet`]: https://dagger.dev/api/latest/dagger/multibindings/IntoSet.html
[`@Provides`]: https://dagger.dev/api/latest/dagger/Provides.html
[`@StringKey`]: https://dagger.dev/api/latest/dagger/multibindings/StringKey.html
