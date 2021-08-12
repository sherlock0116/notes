# 【Guice】01| Guice核心模型

了解 `Key`，`Provider` 以及为何 `Guice` 只是一个 `Map<Key, Provider>`

这里先贴一个示例代码, 下面会用到

```java
package cgs.guice.quickstart;

import com.google.inject.AbstractModule;
import com.google.inject.Guice;
import com.google.inject.Inject;
import com.google.inject.Injector;
import com.google.inject.Provides;

import javax.inject.Qualifier;
import java.lang.annotation.Retention;

import java.lang.annotation.RetentionPolicy;

/**
 * @Description TODO
 * @Author sherlock
 * @Date
 */
public class GuiceDemo {

	@Qualifier
  @Retention(RetentionPolicy.RUNTIME)
  @interface Message{}

  @Qualifier
  @Retention(RetentionPolicy.RUNTIME)
  @interface Count {}

  /**
    *  给 {@link Greeter} 内成员变量 message 和 count 提供变量值绑定
    */
  static class DemoModule extends AbstractModule {

		@Provides
    @Count
    static Integer provideCount() {
        return 3;
    }

    @Provides
    @Message
    static String provideMessage() {
      return "Hello Guice !";
    }
	}

  static class Greeter {
    private final String message;
    private final int count;

    // Greeter declares that it needs a string message and an integer
    // representing the number of time the message to be printed.
    // The @Inject annotation marks this constructor as eligible to be used by
    // Guice.
    @Inject
    Greeter(@Message String message, @Count int count) {
      this.message = message;
      this.count = count;
		}

    void sayHello() {
      for (int i=0; i < count; i++) {
        System.out.println(message);
      }
    }
	}

  public static void main(String[] args) {
    Module module = new DemoModule();
    Injector injector = Guice.createInjector(module);
    Greeter greeter = injector.getInstance(Greeter.class);
    greeter.sayHello();
  }
}
```



## Guice keys

`Guice`用 `Key` 标识可以解决的依赖性。
在 `cgs.guice.quickstart.GuiceDemo.Greeter` 的构造函数中声明了两个依赖关系，这些依赖关系在 `Guice` 中表示为：`Key`

- `@Message String -> Key<String>`
- `@Count int -> Key<Integer>`



最简单形式Key表示Java中的类型：

```java
// 标识了依赖是一个 String类型
Key<String> databaseKey = Key.get(String.class);
```

但是，应用程序通常具有相同类型的依赖项：

```javascript
final class MultilingualGreeter {
  private String englishGreeting;
  private String spanishGreeting;

  MultilingualGreeter(String englishGreeting, String spanishGreeting) {
    this.englishGreeting = englishGreeting;
    this.spanishGreeting = spanishGreeting;
  }
}
```

Guice 使用 绑定注释 来区分相同类型的依赖项，从而使类型更具体：

```java
final class MultilingualGreeter {
    private String englishGreeting;
    private String spanishGreeting;

    @Inject
    MultilingualGreeter( @English String englishGreeting, @Spanish String spanishGreeting) {
        this.englishGreeting = englishGreeting;
        this.spanishGreeting = spanishGreeting;
    }
}
```

具有绑定 `Annotation` 的 `Key` 可以创建为：

- `Key<String> englishGreetingKey = Key.get(String.class, English.class);`
- `Key<String> spanishGreetingKey = Key.get(String.class, Spanish.class);`

当应用程序调用 `injector.getInstance(MultilingualGreeter.class)`创建的实例 `MultilingualGreeter` 时。这等效于：

```java
// Guice internally does this for you so you don't have to wire up those
// dependencies manually.
String english = injector.getInstance(
  Key.get(String.class, English.class)
);

String spanish = injector.getInstance(
  Key.get(String.class, Spanish.class)
);

MultilingualGreeter greeter = new MultilingualGreeter(
  english, 
  spanish
);
```

总结：`Guice` 的 `Key` 是一种类型，它与可选的绑定注释结合在一起用于标识依赖关系。

## Guice Provider

`Guice` 用 `Provider` 表示能够创建满足依赖要求的对象的工厂。
`Provider` 是具有单个方法的接口：

```java
interface Provider<T> {
  // Provides an instance of T
  T get();
}
```

每个实现 `Provider` 类都是一些代码，这些代码知道如何提供的实例`T`。

它可以调用 `new T()`，也可以通过 `T` 其他方式构造，或者可以从缓存中返回一个预先计算的实例。

大多数应用程序不直接实现 `Provider` 接口，它们用于 `Module` 配置`Guice` 注入器，并且 `Guice` 注入器在内部为它知道如何创建的所有对象创建 Provider 。

例如，以下 `Guice` 模块创建两个 `Provider`：

```java
class DemoModule extends AbstractModule {
  @Provides
  @Count
  static Integer provideCount() {
    return 3;
  }

  @Provides
  @Message
  static String provideMessage() {
    return "hello world";
  }
}
```

- `Provider<String>` 返回 message "hello world"
- `Provider<Integer>` 调用 provideCount 方法 并返回 3



## Guice 核心模型

学习 `Guice` 一个好方法是将其想象为一个大`Map<Key<?>, Provider<?>>`(当然，这不是类型安全的，因为这两个通配符通用名将不匹配。这是简化的思维模型，而不是严格的实现。)

像 `Provider` 一样，大多数应用程序都不直接创建 `Key`，而是使用 `Modules` 配置从 `Key` 到 `Provider` 的映射。

`Module` 以配置逻辑束的形式存在，这些逻辑只是将事物添加到 `Guice` 映射中。

有两种方法可以执行此操作：使用方法注释（如 `@Provides`）或使用 Guice 域特定语言（DSL）

### Modules add things into the map

从概念上讲，这些 API 只是提供了操作 `Guice Map` 的方法。他们所做的操作非常简单。以下是一些示例翻译，为简洁明了起见，使用 Java 8语法显示了这些翻译：

| Guice DSL语法                      | 模型                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| `bind(key).toInstance(value)`      | `map.put(key, () -> value)`  （实例绑定）                    |
| `bind(key).toProvider(provider)`   | `map.put(key, provider)`  （提供者绑定）                     |
| `bind(key).to(anotherKey)`         | `map.put(key, map.get(anotherKey))`  （链接绑定）            |
| `@Provides Foo provideFoo() {...}` | `map.put(Key.get(Foo.class), module::provideFoo)`  （提供者方法绑定） |

`DemoModule` 在 `Guice Map` 中添加两个条目：

- `@Message String` -> `() -> "hello world"`
- `@Count Integer` -> `() -> DemoModule.provideCount()`

### You don't *pull* things out of a map, you *declare* that you need them

这是 `DI` 的本质。如果您需要某些东西，你不需要手动创建，甚至不需要 `class` 给你返回什么。相反，您只需声明没有它就无法完成工作，然后依靠其模块为您提供所需的东西。

该模型与大多数人对代码的看法相反：***它是一种更具说明性的模型，而不是命令性的模型。这就是为什么依赖项注入通常被描述为一种控制反转（IoC）的原因。***

声明您需要某些东西的一些方法：

1. ***申明 `@Inject` 的构造函数的参数***

   ```java
   class Foo {
     private Database database;
   
     @Inject
     Foo(Database database) {  // We need a database, from somewhere
       this.database = database;
     }
   }
   ```

2. ***申明 `@Provides` 方法的参数***

   ```java
   @Provides
   Database provideDatabase(
       // We need the @DatabasePath String before we can construct a Database
       @DatabasePath String databasePath) {
     return new Database(databasePath);
   }
   ```



### Dependencies form a graph

当注入具有依赖关系的事物时，`Guice` 递归注入依赖关系。您可以想象为了注入一个 如上所示的实例 `Foo` ，`Guice` 创建了如下所示的 `Provider` 实现：

```java
class FooProvider implements Provider<Foo> {
  @Override
  public Foo get() {
    Provider<Database> databaseProvider = guiceMap.get(Key.get(Database.class));
    Database database = databaseProvider.get();
    return new Foo(database);
  }
}

class ProvideDatabaseProvider implements Provider<Database> {
  @Override
  public Database get() {
    Provider<String> databasePathProvider =
        guiceMap.get(Key.get(String.class, DatabasePath.class));
    String databasePath = databasePathProvider.get();
    return module.provideDatabase(databasePath);
  }
}
```

依赖关系形成一个有*向图*，注入通过对对象的所有依赖关系进行深度优先遍历来实现。

`Guice Injector `对象代表整个依赖图。要创建 `Injector`，Guice 需要验证整个图形是否有效。不能有任何需要但未提供依赖项的“悬空”节点。如果图形由于任何原因而无效，Guice 会抛出`CreationException`描述问题所在。

通常，`CreationException` 义意味着您正在尝试使用某种东西（例如`Database`），但是您忘记了包括提供它的模块。其他时候，这可能意味着您输入了错误的内容。

