---
layout: default
title: 初始化 Dagger 设置
---

让我们更改示例以实际使用 Dagger 创建一个 `CommandRouter` 实例。
我们将从创建 [`@Component`] 接口开始：

```java
@Component
interface CommandRouterFactory {
  CommandRouter router();
}
```

`CommandRouterFactory` 是 `CommandRouter` 的常规工厂类型。
它的实现将调用 `new CommandRouter()` 代替我们 main 方法中那样。
但是，我们没有编写 `CommandRouterFactory` 的实现类，
而是使用 [`@Component`] 对其进行注解，
使 Dagger 为我们 _生成_  实现类:`DaggerCommandRouterFactory`。
注意它有个给我们一个实例的静态方法 `create()`。

```java
class CommandLineAtm {
  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    CommandRouterFactory commandRouterFactory =
        DaggerCommandRouterFactory.create();
    CommandRouter commandRouter = commandRouterFactory.router();

    while (scanner.hasNextLine()) {
      commandRouter.route(scanner.nextLine());
    }
  }
}
```

为了使 Dagger 知道如何创建 `CommandRouter`，
我们还需要为其构造函数添加 [`@Inject`] 注解：

```java
final class CommandRouter {
  ...
  @Inject
  CommandRouter() {}
  ...
}
```

[`@Inject`] 注解向 Dagger 表示当我们需要
`CommandRouter` 时，Dagger 应该调用 `new CommandRouter()`。

> **Aside:** 有关如何将 Dagger 正确添加到您的构建，请参见[这些说明]

[这些说明]: https://github.com/google/dagger#installation

我们还没有做太多特别的事情，但是我们已经有一个满足 Dagger 最低要求的应用！
再次运行该应用程序以查看其运行情况。

> **概念**
>
> *   **[`@Component`] ** 告诉 Dagger 实现 `interface` 或 `abstract
>     class` 创建并返回一个或多个对象。
>     * Dagger 将生成一个实现 component 类型的类。
>     生成的类将命名为 `DaggerYourType`（或
>     `DaggerYourType_NestedType`用于嵌套类型）
> *   **[`@Inject`]** 告诉 Dagger 如何实例化
>    类。我们很快会看到更多。

<section style="text-align: center" markdown="1">

[Previous](01-setup) · [Next](03-first-command)

</section>

[`@Component`]: https://dagger.dev/api/latest/dagger/Component.html
[`@Inject`]: http://docs.oracle.com/javaee/7/api/javax/inject/Inject.html
