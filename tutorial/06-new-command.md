---
layout: default
title: 一个新的命令
---

让我们添加一个新命令从而登录 ATM：

```java
final class LoginCommand extends SingleArgCommand {
  private final Outputter outputter;

  @Inject
  LoginCommand(Outputter outputter) {
    this.outputter = outputter;
  }

  @Override
  public String key() {
    return "login";
  }

  @Override
  public Status handleArg(String username) {
    outputter.output(username + " is logged in.");
    return Status.HANDLED;
  }
}
```

(为简单起见，我们定义了抽象的`SingleArgCommand`
[here][SingleArgCommand])。

我们可以像 `HelloWorldModule` 一样创建一个 `LoginCommandModule` 来绑定
`LoginCommand` 作为`Command` 的实现：

```java
@Module
abstract class LoginCommandModule {
  @Binds
  abstract Command loginCommand(LoginCommand command);
}
```

为了开始在 `CommandRouter` 中使用 `LoginCommand`，
我们将使用`LoginCommandModule` 替代 [`@Component`] 注解中的`HelloWorldModule`。
运行该应用程序，然后尝试登录。

这开始显示出使用 Dagger 的一些好处。
一行 _声明_ 的改变，使得我们可以更改 `CommandRouter` 接收到的 `Command`。 
`CommandRouter` 没有任何变化，它可以正常工作。
使用这个方法，您可以编写许多不同版本的应用程序并重复使用
无需大量更改的代码。

<section style="text-align: center" markdown="1">

[Previous](05-abstraction-for-output) · [Next](07-two-for-the-price-of-one)

</section>

[SingleArgCommand]: https://github.com/google/dagger/tree/master/java/dagger/example/atm/SingleArgCommand.java

[`@Component`]: https://dagger.dev/api/latest/dagger/Component.html
