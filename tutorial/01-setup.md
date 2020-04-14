---
layout: default
title: Tutorial setup
---

为了解释如何使用 Dagger, 我们将构建一个命令行
[ATM](https://en.wikipedia.org/wiki/Automated_teller_machine) 应用程序。
它将跟踪帐户余额，并在命令行上接受命令：

```
> deposit 20
> withdraw 10
```

让我们从构建此应用程序的外壳开始，刚开始不使用 Dagger。
如果某些方面看起来过于复杂，请耐心等待，因为一旦应用程序变大，Dagger 就开始展示其强大功能。

首先，我们将为 ATM 可以处理的每个可能的文本命令创建一个`命令`界面。

```java
/** Logic to process some user input. */
interface Command {
  /**
   * String token that signifies this command should be selected (e.g.:
   * "deposit", "withdraw")
   */
  String key();

  /** Process the rest of the command's words and do something. */
  Status handleInput(List<String> input);

  enum Status {
    INVALID,
    HANDLED
  }
}
```

然后我们将创建一个 `CommandRouter`，它可以收集多个 `Command`
并根据输入中的第一个单词将输入字符串路由到它们。
首先，我们将为它提供一个空的命令映射。

```java
final class CommandRouter {
  private final Map<String, Command> commands = Collections.emptyMap();

  Status route(String input) {
    List<String> splitInput = split(input);
    if (splitInput.isEmpty()) {
      return invalidCommand(input);
    }

    String commandKey = splitInput.get(0);
    Command command = commands.get(commandKey);
    if (command == null) {
      return invalidCommand(input);
    }

    Status status =
        command.handleInput(splitInput.subList(1, splitInput.size()));
    if (status == Status.INVALID) {
      System.out.println(commandKey + ": invalid arguments");
    }
    return status;
  }

  private Status invalidCommand(String input) {
    System.out.println(
        String.format("couldn't understand \"%s\". please try again.", input));
    return Status.INVALID;
  }

  // Split on whitespace
  private static List<String> split(String string) {  ...  }
}
```

最后，我们将创建一个 main 方法：

```java
class CommandLineAtm {
  public static void main(String[] args) {
    Scanner scanner = new Scanner(System.in);
    CommandRouter commandRouter = new CommandRouter();

    while (scanner.hasNextLine()) {
      commandRouter.route(scanner.nextLine());
    }
  }
}
```

恭喜你！现在，我们有了一个在命令行运行的 ATM！
它暂时什么都没做，但是很快我们将改变它。

<section style="text-align: center" markdown="1">
  [Previous](index) · [Next](02-initial-dagger)
</section>
