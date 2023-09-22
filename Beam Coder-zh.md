当 [[Beam]] 运行程序执行您的 pipeline 时，它们通常需要具体化 `PCollection` 中的中间数据，这需要在元素与字节字符串之间进行转换。 Beam SDK 使用名为 `Coder` 的对象来描述如何对给定 `PCollection` 的元素进行编码和解码
>请注意，与外部数据源或接收器交互时，编码器与解析或格式化数据无关。此类解析或格式化通常应使用 `ParDo` 或 `MapElements` 等转换显式完成。

在 Beam SDK for Java 中，类型 `Coder` 提供了编码和解码数据所需的方法。 
SDK for Java 提供了许多可与各种标准 Java 类型配合使用的 Coder 子类，例如 Integer、Long、Double、StringUtf8 等。
>可以在 Coder 包中找到所有可用的 Coder 子类。

### Specifying coders 指定编码器
Beam SDK 需要为管道中的每个 `PCollection` 配备一个 Coder。
在大多数情况下，Beam SDK 能够根据其元素类型或生成它的转换自动推断 `Coder` 为 `PCollection` ，但是，在某些情况下，管道作者将需要显式指定 `Coder` ，或为其自定义类型开发 `Coder` 。

您可以使用方法 `PCollection.setCoder` 显式设置现有 `PCollection` 的 Coder。
>请注意，您不能对已完成的 `PCollection` 调用 `setCoder` （例如，通过对其调用 `.apply` ）。

您可以使用方法 `getCoder` 获取现有 `PCollection` 的 Coder。
>如果尚未设置编码器并且无法推断给定的 `PCollection` ，则此方法将失败并返回 `IllegalStateException` 。

Beam SDK 在尝试自动推断 `PCollection` 的 `Coder` 时使用各种机制。

每个管道对象都有一个 `CoderRegistry` 。 `CoderRegistry` 表示 Java 类型到默认编码器的映射，管道应将其用于每种类型的 `PCollection` 。

默认情况下，Beam SDK for Java 使用来自转换函数对象的类型参数自动推断由 `PTransform` 生成的 `PCollection` 元素的 `Coder` ，
例如 `DoFn` 。例如，在 `ParDo` 的情况下， `DoFn<Integer, String>` 函数对象接受 `Integer` 类型的输入元素并生成 `String` 的默认 `Coder` （在默认管道 `CoderRegistry` 中，这是 `StringUtf8Coder`

**使用 `Create` 时，确保拥有正确编码器的最简单方法是在应用 `Create` 转换时调用 `withCoder` 。**

#### Setting the default coder for a type 设置类型的默认编码器
要为特定管道的 Java 类型设置默认编码器，需要获取并修改管道的 `CoderRegistry` 。
使用方法 `Pipeline.getCoderRegistry` 获取 `CoderRegistry` 对象，然后使用方法 `CoderRegistry.registerCoder` 为目标类型注册一个新的 `Coder`:
```java
PipelineOptions options = PipelineOptionsFactory.create();
Pipeline p = Pipeline.create(options);

CoderRegistry cr = p.getCoderRegistry();
cr.registerCoder(Integer.class, BigEndianIntegerCoder.class);
```

#### Annotating a custom data type with a default coder 使用默认编码器注释自定义数据类型
如果管道程序定义了自定义数据类型，则可以使用 `@DefaultCoder` 注释来指定要与该类型一起使用的编码器。默认情况下，Beam 将使用 `SerializableCoder` ，它使用 Java 序列化，但它有缺点：
- 它在编码大小和速度方面效率低下。请参阅 [Java 序列化方法的比较](https://blog.softwaremill.com/the-best-serialization-strategy-for-event-sourcing-9321c299632b)
- 它是不确定的：它可能会为两个等效对象生成不同的二进制编码。
- 对于键/值对，基于键的操作（GroupByKey、Combine）和每个键状态的正确性取决于键的确定性编码器

可以使用 `@DefaultCoder` 注释设置新的默认值，如下所示：
```java
@DefaultCoder(AvroCoder.class)
public class MyCustomDataType {
  ...
}
```
如果创建了一个自定义编码器来匹配数据类型，并且想要使用 `@DefaultCoder` 注释，则Coder类必须实现静态 `Coder.of(Class<T>)` 工厂方法:
```java
public class MyCustomCoder implements Coder {
  public static Coder<T> of(Class<T> clazz) {...}
  ...
}

@DefaultCoder(MyCustomCoder.class)
public class MyCustomDataType {
  ...
}
```
