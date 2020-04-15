---
layout: default
title: Dagger & Android
redirect_from:
  - /android
---

与大多数其他依赖项注入框架相比，
Dagger 2的主要优点之一是其严格生成的实现（无反射），
意味着它可以在 Android 应用程序中使用。
但是，在 Android 应用程序中使用 Dagger 时，仍然需要考虑一些注意事项。

## 原理

虽然为 Android 编写的代码是 Java 源代码，
但是在样式方面常常大不相同。
通常，存在这样的差异以适应移动平台独特的 [performance][android-performance] 注意事项。

但是通常应用于Android代码的许多模式与应用于其他Java代码的模式相反。
甚至 [Effective Java][effective-java]  中的很多建议都不适用于Android。

为了实现惯用代码且可移植代码的目标，
Dagger依靠[ProGuard]对已编译的字节码进行后处理。
这使Dagger可以生成在服务器和Android上看起来和感觉自然的源代码，
同时使用不同的工具链来生成在两种环境中都能高效执行的字节码。
此外，Dagger的明确目标是确保其生成的Java源代码始终与ProGuard优化兼容。

当然，并非所有问题都可以通过这种方式解决，
但这是提供Android特定兼容性的主要机制。

### tl;dr

Dagger假定Android上的用户将使用R8或ProGuard。

## 为什么在Android上使用Dagger很难

使用Dagger编写Android应用程序的主要困难之一是，
许多Android框架类都是由OS本身实例化的，
例如`Activity` 和 `Fragment`，但是Dagger如果能够创建所有注入的对象，则效果最佳。
相反，您必须在生命周期方法中执行成员注入。这意味着许多类最终看起来像：

```java
public class FrombulationActivity extends Activity {
  @Inject Frombulator frombulator;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    // DO THIS FIRST. Otherwise frombulator might be null!
    ((SomeApplicationBaseType) getContext().getApplicationContext())
        .getApplicationComponent()
        .newActivityComponentBuilder()
        .activity(this)
        .build()
        .inject(this);
    // ... now you can write the exciting code
  }
}
```

这有一些问题:

1. 复制粘贴代码使以后很难重构。随着越来越多的开发人员复制粘贴该块，越来越少的人会知道它的实际作用。

2. 从根本上讲，它需要请求注入的类型 (`FrombulationActivity`) 来了解其注入器。
即使这是通过接口而不是具体类型完成的，它也打破了依赖注入的核心原则：类不应该知道如何注入依赖。

## `dagger.android`

[`dagger.android`] 中的类提供了一种简化上述问题的方法。
这需要学习一些额外的API和概念，
但可以在生命周期的正确位置减少Android类中的样板代码和注入。

另一种方法是只使用常规的Dagger API并遵循 
[here](https://developer.android.com/training/dependency-injection/dagger-android) 的指南。
这可能更容易理解，但缺点是必须手动编写额外的样板。


Jetpack和Dagger团队正在共同为Android上的Dagger制定一项
[新计划](https://medium.com/androiddevelopers/dependency-injection-guidance-on-android-ads-2019-b0b56d774bc2)，
希望成为与目前的状况大相径庭。
不幸的是，虽然它还没有准备好，当您选择如何在现在的Android项目中使用Dagger时，
可能需要考虑一下这一点。

### 注入 `Activity` 对象

1.  在您的应用程序组件中安装[`AndroidInjectionModule`]，
    以确保这些基本类型所需的所有绑定均可用。

2.  从写一个实现
    [`AndroidInjector<YourActivity>`][AndroidInjector] 的`@Subcomponent`开始, 和一个
     扩展自[`AndroidInjector.Factory<YourActivity>`][AndroidInjector.Factory]
     的 `@Subcomponent.Factory`:

    ```java
    @Subcomponent(modules = ...)
    public interface YourActivitySubcomponent extends AndroidInjector<YourActivity> {
      @Subcomponent.Factory
      public interface Factory extends AndroidInjector.Factory<YourActivity> {}
    }
    ```

3.  定义子组件之后，通过以下方式将其添加到组件层次结构中，
    定义一个绑定子组件工厂的模块，并将其添加到
    注入您的`Application`:

    ```java
    @Module(subcomponents = YourActivitySubcomponent.class)
    abstract class YourActivityModule {
      @Binds
      @IntoMap
      @ClassKey(YourActivity.class)
      abstract AndroidInjector.Factory<?>
          bindYourAndroidInjectorFactory(YourActivitySubcomponent.Factory factory);
    }

    @Component(modules = {..., YourActivityModule.class})
    interface YourApplicationComponent {
      void inject(YourApplication application);
    }
    ```

    Pro-tip: If your subcomponent and its factory have no other methods or
    supertypes other than the ones mentioned in step #2, you can use
    [`@ContributesAndroidInjector`] to generate them for you. Instead of steps 2
    and 3, add an `abstract` module method that returns your activity, annotate
    it with `@ContributesAndroidInjector`, and specify the modules you want to
    install into the subcomponent. If the subcomponent needs scopes, apply the
    scope annotations to the method as well.

    ```java
    @ActivityScope
    @ContributesAndroidInjector(modules = { /* modules to install into the subcomponent */ })
    abstract YourActivity contributeYourAndroidInjector();
    ```

4.  接下来，让你的`Application` 实现 [`HasAndroidInjector`]
    并 `@Inject` a
    [`DispatchingAndroidInjector<Object>`][DispatchingAndroidInjector] to
    return from the `androidInjector()` method:

    ```java
    public class YourApplication extends Application implements HasAndroidInjector {
      @Inject DispatchingAndroidInjector<Object> dispatchingAndroidInjector;

      @Override
      public void onCreate() {
        super.onCreate();
        DaggerYourApplicationComponent.create()
            .inject(this);
      }

      @Override
      public AndroidInjector<Object> androidInjector() {
        return dispatchingAndroidInjector;
      }
    }
    ```

5.  最后, 在`Activity.onCreate()` 方法中, 
    在调用 `super.onCreate();`*之前* 调用 
    [`AndroidInjection.inject(this)`][AndroidInjection.inject(Activity)]:

    ```java
    public class YourActivity extends Activity {
      public void onCreate(Bundle savedInstanceState) {
        AndroidInjection.inject(this);
        super.onCreate(savedInstanceState);
      }
    }
    ```

6.  恭喜啦!

#### 那是怎么工作的？

`AndroidInjection.inject()` gets a `DispatchingAndroidInjector<Object>` from
the `Application` and passes your activity to `inject(Activity)`. The
`DispatchingAndroidInjector` looks up the `AndroidInjector.Factory` for your
activity’s class (which is `YourActivitySubcomponent.Factory`), creates the
`AndroidInjector` (which is `YourActivitySubcomponent`), and passes your
activity to `inject(YourActivity)`.

### 注入`Fragment`对象

Injecting a `Fragment` is just as simple as injecting an `Activity`. Define your
subcomponent in the same way.

Instead of injecting in `onCreate()` as is done for `Activity`
types, [inject `Fragment`s to in `onAttach()`](#when-to-inject).

Unlike the modules defined for `Activity`s, you have a choice of where to
install modules for `Fragment`s. You can make your `Fragment` component a
subcomponent of another `Fragment` component, an `Activity` component, or the
`Application` component — it all depends on which other bindings your `Fragment`
requires. After deciding on the component location, make the corresponding type
implement `HasAndroidInjector` (if it doesn't already). For example, if your `Fragment`
needs bindings from `YourActivitySubcomponent`, your code will look something
like this:

```java
public class YourActivity extends Activity
    implements HasAndroidInjector {
  @Inject DispatchingAndroidInjector<Object> androidInjector;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    AndroidInjection.inject(this);
    super.onCreate(savedInstanceState);
    // ...
  }

  @Override
  public AndroidInjector<Object> androidInjector() {
    return androidInjector;
  }
}

public class YourFragment extends Fragment {
  @Inject SomeDependency someDep;

  @Override
  public void onAttach(Activity activity) {
    AndroidInjection.inject(this);
    super.onAttach(activity);
    // ...
  }
}

@Subcomponent(modules = ...)
public interface YourFragmentSubcomponent extends AndroidInjector<YourFragment> {
  @Subcomponent.Factory
  public interface Factory extends AndroidInjector.Factory<YourFragment> {}
}

@Module(subcomponents = YourFragmentSubcomponent.class)
abstract class YourFragmentModule {
  @Binds
  @IntoMap
  @ClassKey(YourFragment.class)
  abstract AndroidInjector.Factory<?>
      bindYourFragmentInjectorFactory(YourFragmentSubcomponent.Factory factory);
}

@Subcomponent(modules = { YourFragmentModule.class, ... }
public interface YourActivityOrYourApplicationComponent { ... }
```

### 基本框架类型

Because `DispatchingAndroidInjector` looks up the appropriate
`AndroidInjector.Factory` by the class at runtime, a base class can implement
`HasAndroidInjector` as well as call `AndroidInjection.inject()`. All each
subclass needs to do is bind a corresponding `@Subcomponent`. Dagger provides a
few base types that do this, such as [`DaggerActivity`] and [`DaggerFragment`],
if you don't have a complicated class hierarchy. Dagger also provides a
[`DaggerApplication`] for the same purpose — all you need to do is to extend it
and override the `applicationInjector()` method to return the component that
should inject the `Application`.

The following types are also included:
  - [`DaggerService`] and [`DaggerIntentService`]
  - [`DaggerBroadcastReceiver`]
  - [`DaggerContentProvider`]

*Note:* [`DaggerBroadcastReceiver`] should only be used when the
`BroadcastReceiver` is registered in the `AndroidManifest.xml`. When the
`BroadcastReceiver` is created in your own code, prefer constructor injection
instead.

### Support libraries

For users of the Android support library, parallel types exist in the
`dagger.android.support` package.

> TODO(ronshapiro):我们应该开始通过androidx软件包对此进行拆分

### 我如何得到它？

将以下内容添加到您的build.gradle中：

```groovy
dependencies {
  compile 'com.google.dagger:dagger-android:2.x'
  compile 'com.google.dagger:dagger-android-support:2.x' // if you use the support libraries
  annotationProcessor 'com.google.dagger:dagger-android-processor:2.x'
}
```


<a name="when-to-inject"></a>

## 什么时候注入

Constructor injection is preferred whenever possible because `javac` will ensure
that no field is referenced before it has been set, which helps avoid
`NullPointerException`s. When members injection is required (as discussed
above), prefer to inject as early as possible. For this reason, `DaggerActivity`
calls `AndroidInjection.inject()` immediately in `onCreate()`, before calling
`super.onCreate()`, and `DaggerFragment` does the same in `onAttach()`, which
also prevents inconsistencies if the `Fragment` is reattached.

It is crucial to call `AndroidInjection.inject()` before `super.onCreate()` in
an `Activity`, since the call to `super` attaches `Fragment`s from the previous
activity instance during configuration change, which in turn injects the
`Fragment`s. In order for the `Fragment` injection to succeed, the `Activity`
must already be injected. For users of [ErrorProne], it is a
compiler error to call `AndroidInjection.inject()` after `super.onCreate()`.

## FAQ

### 定义 AndroidInjector.Factory

`AndroidInjector.Factory` is intended to be a stateless interface so that
implementors don't have to worry about managing state related to the object
which will be injected. When `DispatchingAndroidInjector` requests a
`AndroidInjector.Factory`, it does so through a `Provider` so that it doesn't
explicitly retain any instances of the factory. Because some implementations may
retain an instance of the `Activity`/`Fragment`/etc that is being injected, it
is a compile-time error to apply a scope to the methods which provide them. If
you are positive that your `AndroidInjector.Factory` does not retain an instance
to the injected object, you may suppress this error by applying
`@SuppressWarnings("dagger.android.ScopedInjectorFactory")` to your module
method.

<!-- References -->

[AndroidInjection.inject(Activity)]: https://dagger.dev/api/latest/dagger/android/AndroidInjection.html#inject-android.app.Activity-
[AndroidInjector]: https://dagger.dev/api/latest/dagger/android/AndroidInjector.html
[AndroidInjector.Factory]: https://dagger.dev/api/latest/dagger/android/AndroidInjector.Factory.html
[android-performance]: http://developer.android.com/training/best-performance.html
[`AndroidInjectionModule`]: https://dagger.dev/api/latest/dagger/android/AndroidInjectionModule.html
[`@ContributesAndroidInjector`]: https://dagger.dev/api/latest/dagger/android/ContributesAndroidInjector.html
[`dagger.android`]: https://dagger.dev/api/latest/dagger/android/package-summary.html
[`DaggerActivity`]: https://dagger.dev/api/latest/dagger/android/DaggerActivity.html
[`DaggerApplication`]: https://dagger.dev/api/latest/dagger/android/DaggerApplication.html
[`DaggerBroadcastReceiver`]: https://dagger.dev/api/latest/dagger/android/DaggerBroadcastReceiver.html
[`DaggerContentProvider`]: https://dagger.dev/api/latest/dagger/android/DaggerContentProvider.html
[`DaggerFragment`]: https://dagger.dev/api/latest/dagger/android/support/DaggerFragment.html
[`DaggerIntentService`]: https://dagger.dev/api/latest/dagger/android/DaggerIntentService.html
[`DaggerService`]: https://dagger.dev/api/latest/dagger/android/DaggerService.html
[DispatchingAndroidInjector]: https://dagger.dev/api/latest/dagger/android/DispatchingAndroidInjector.html
[effective-java]: https://books.google.com/books?id=ka2VUBqHiWkC
[ErrorProne]: https://github.com/google/error-prone
[`HasAndroidInjector`]: https://dagger.dev/api/latest/dagger/android/HasAndroidInjector.html
[ProGuard]: http://proguard.sourceforge.net/

