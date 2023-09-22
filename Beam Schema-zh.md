## What is Schema? 
Schema 提供了独立于任何特定编程语言类型的 [[Beam]] 记录类型系统。

可能有多个 Java 类都具有相同的Schema（例如 Protocol-Buffer 类或 POJO 类），Beam 允许在这些类型之间无缝转换。Schema 还提供了一种简单的方法来推理不同编程语言 API 的类型。

具有 `Schema` 的 `PCollection` 不需要指定 `Coder` ，因为 Beam 知道如何对 `Schema` 行进行编码和解码； Beam 使用特殊的编码器对 `Schema` 类型进行编码。

## Schema for pragramming language types
虽然 Schema 本身是独立于语言的，但它们被设计为自然嵌入到正在使用的 Beam SDK 的编程语言中。
这允许 Beam user 继续使用 Native 类型，同时获得 Beam 理解其元素模式的优势。

在 Java 中，您可以使用以下一组 `Class` 来表示 `Purchase Schema`。 Beam 将根据类的成员自动推断出正确的 `Schema`。
```java
@DefaultSchema(JavaBeanSchema.class)
public class Purchase {
  public String getUserId();  // Returns the id of the user who made the purchase.
  public long getItemId();  // Returns the identifier of the item that was purchased.
  public ShippingAddress getShippingAddress();  // Returns the shipping address, a nested type.
  public long getCostCents();  // Returns the cost of the item.
  public List<Transaction> getTransactions();  // Returns the transactions that paid for this purchase (returns a list, since the purchase might be spread out over multiple credit cards).

  @SchemaCreate
  public Purchase(String userId, long itemId, ShippingAddress shippingAddress, long costCents,List<Transaction> transactions) {
      ...
  }
}

@DefaultSchema(JavaBeanSchema.class)
public class ShippingAddress {
  public String getStreetAddress();
  @Nullable public String getState();
  @SchemaCreate
  public ShippingAddress(String streetAddress, @Nullable String state) {
     ...
  }
}

@DefaultSchema(JavaBeanSchema.class)
public class Transaction {
  public String getBank();
  public double getPurchaseAmount();

  @SchemaCreate
  public Transaction(String bank, double purchaseAmount) {
     ...
  }
}

```

如上所述使用 JavaBean Class 是将 Schema 映射到 Java 类的一种方法。
然而，多个 Java 类可能具有相同的 `Schema`，在这种情况下，不同的 Java 类型通常可以互换使用。 Beam 将在具有匹配模式的类型之间**添加隐式转换**。例如，上面的 `Transaction` 类与以下面这个`TransactionPojo`具有相同的Schema：
```java
@DefaultSchema(JavaFieldSchema.class)
public class TransactionPojo {
  public String bank;
  public double purchaseAmount;
}
```
因此，如果我们有两个 `PCollection` ，如下所示
```java
PCollection<Transaction> transactionBeans = readTransactionsAsJavaBean();
PCollection<TransactionPojos> transactionPojos = readTransactionsAsPojo();
```
那么这两个 `PCollection` 将具有相同的 Schema，即使它们的 Java 类型不同。这意味着以下两个代码片段是有效的：
```java
transactionBeans.apply(ParDo.of(new DoFn<...>() {
   @ProcessElement public void process(@Element TransactionPojo pojo) {
      ...
   }
}));
```
和
```java
transactionPojos.apply(ParDo.of(new DoFn<...>() {
   @ProcessElement public void process(@Element Transaction row) {
    }
}));
```
尽管在这两种情况下 `@Element` 参数与 `PCollection` 的 Java 类型不同，但由于 Schema 相同，Beam 会自动进行转换。内置的 `Convert` 转换还可用于在等效模式的 Java 类型之间进行转换。
## Schema Definition
`PCollection` 的架构将 `PCollection` 的元素定义为命名字段的有序列表。每个字段都有一个名称、一个类型，可能还有一组用户选项。字段的类型可以是原始类型或复合类型。以下是 Beam 目前支持的原始类型：

| Type     | Description                                             |
|----------|---------------------------------------------------------|
| BYTE     | An 8-bit signed value                                   |
| INT16    | A 16-bit signed value                                   |
| INT32    | A 32-bit signed value                                   |
| INT64    | A 64-bit signed value                                   |
| DECIMAL  | An arbitrary-precision decimal type                     |
| FLOAT    | A 32-bit IEEE 754 floating point number                 |
| DOUBLE   | A 64-bit IEEE 754 floating point number                 |
| STRING   | A string                                                |
| DATETIME | A timestamp represented as milliseconds since the epoch |
| BOOLEAN  | A boolean value                                         |
| BYTES    | A raw byte array                                        |

字段还可以引用嵌套架构。在这种情况下，字段将具有 `ROW` 类型，并且嵌套架构将是此字段类型的属性。
支持三种集合类型作为字段类型：`ARRAY`、`ITERABLE` 和 `MAP`：
- ARRAY 表示可重复的
- ITERABLE 与数据非常类似
- MAP 表示从键到值的关联映射
### Logical types 逻辑类型
用户可以扩展 Schema 类型系统以添加可用作字段的自定义Logcial type。逻辑类型由唯一标识符和参数来标识。**Logical type 还指定用于存储的基础架构类型，以及与该类型之间的转换。**
#### Defining a logical type
要定义 Logical type，必须指定用于表示基础类型的Schema 类型以及该类型的唯一标识符。
Logical type 在 Schema 类型之上强加了额外的语义。
	例如，表示纳秒时间戳的 Logical type 表示为 INT64 和 INT32 的 Schema field，该 Schema 并未解释该类型，但 Logical type 含义中 INT64 表示秒，INT32 表示纳秒。

Logical type 也由参数指定，并允许创建相关类型的 classes。

Java 中 logical type 被指定为 `LogicalType` 的子类。并可以自定义 Java Class 类表示 logical type，但必须提供 converstion functions 来表示此 Java Class 和 Schema 类型之间的转换。
举例，表示纳秒时间戳的 logical type 可以如下实现：
```java
// A Logical type using java.time.Instant to represent the logical type.
public class TimestampNanos implements LogicalType<Instant, Row> {
  // The underlying schema used to represent rows.
  private final Schema SCHEMA = Schema.builder().addInt64Field("seconds").addInt32Field("nanos").build();
  @Override public String getIdentifier() { return "timestampNanos"; }
  @Override public FieldType getBaseType() { return schema; }

  // Convert the representation type to the underlying Row type. Called by Beam when necessary.
  @Override public Row toBaseType(Instant instant) {
    return Row.withSchema(schema).addValues(instant.getEpochSecond(), instant.getNano()).build();
  }

  // Convert the underlying Row type to an Instant. Called by Beam when necessary.
  @Override public Instant toInputType(Row base) {
    return Instant.of(row.getInt64("seconds"), row.getInt32("nanos"));
  }
     ...
}
```

#### Useful logical types
##### **EnumerationType 枚举类型**
此 logical type 允许创建由一组命名常量组成的枚举类型。
```java
Schema schema = Schema.builder()
               …
     .addLogicalTypeField("color", EnumerationType.create("RED", "GREEN", "BLUE"))
     .build();
```
该字段的值以 INT32 类型存储在行中，但是逻辑类型定义了一个值类型，允许以字符串或值的形式访问枚举。例如：
```java
EnumerationType.Value enumValue = enumType.valueOf("RED");
enumValue.getValue();  // Returns 0, the integer value of the constant.
enumValue.toString();  // Returns "RED", the string value of the constant
```
给定一个带有枚举字段的行对象，还可以提取该字段作为枚举值。
```java
EnumerationType.Value enumValue = row.getLogicalTypeValue("color", EnumerationType.Value.class);
```
来自 Java POJO 和 JavaBean 的**自动模式推断** 会自动将 Java 枚举转换为 EnumerationType 逻辑类型。

... 其他参考 [Beam schema](https://beam.apache.org/documentation/programming-guide/#schemas)这一节。
### Creating Schemas
为了利用 Schema， `PCollection` 必须附加一个 Schema。通常，源本身会将 Schema 附加到 PCollection。
	例如，当使用 `AvroIO` 读取 Avro 文件时，源可以自动从 Avro 架构推断出 Beam Schema 并将其附加到 Beam `PCollection` 。然而，并非所有来源都会产生 Schema 。
	此外，Beam  Pipeline 通常具有中间阶段和类型，这些也可以从模式的表达能力中受益。
#### Inferring schemas Schema推断
Beam 能够从各种常见的 Java 类型推断 Schema。 `@DefaultSchema` 注释可用于告诉 Beam 从特定Class推断`Schema`。
该注释采用 `SchemaProvider` 作为参数，并且 `SchemaProvider` 类已针对常见 Java 类型**内置**。
	对于无法注释 Java 类型本身的情况，通过调用 `SchemaRegistry` 。

**Java POJOs**
POJO（普通 Java 对象）是不受 Java 语言规范以外的任何限制约束的 Java 对象。 POJO 可以包含基元成员变量、其他 POJO、集合映射或其数组。 
**POJO 不必扩展预先指定的类或扩展任何特定的接口**。

**如果 POJO 类用 `@DefaultSchema(JavaFieldSchema.class)` 注释，Beam 将自动推断此类的架构。支持嵌套类以及具有 `List` 、数组和 `Map` 字段的类。**

例如，注释以下类告诉 Beam 从该 POJO 类推断模式并将其应用于任何 `PCollection<TransactionPojo>` :
```java
@DefaultSchema(JavaFieldSchema.class)
public class TransactionPojo {
  public final String bank;
  public final double purchaseAmount;
  @SchemaCreate
  public TransactionPojo(String bank, double purchaseAmount) {
    this.bank = bank;
    this.purchaseAmount = purchaseAmount;
  }
}
// Beam will automatically infer the correct schema for this PCollection. No coder is needed as a result.
PCollection<TransactionPojo> pojos = readPojos();
```

`@SchemaCreate` 注释告诉 Beam 该**构造函数可用于创建 TransactionPojo 的实例**，假设构造函数参数与字段名称具有相同的名称。 

`@SchemaCreate` 还可以用于注释类上的**静态工厂方法**，**从而允许构造函数保持私有**。

如果没有 `@SchemaCreate` 注释，则所有字段都必须是非 `final` 字段，并且该类必须具有**零参数构造函数**。

还有一些其他有用的注释会影响 Beam 推断模式的方式。
默认情况下，推断的模式字段名称将与类字段名称匹配。但是 `@SchemaFieldName` 可用于指定用于架构字段的不同名称。 
`@SchemaIgnore` 可用于将特定类字段标记为从推断模式中排除。
	例如，类中通常存在不应包含在模式中的临时字段（例如缓存哈希值以防止昂贵的哈希重新计算），并且 `@SchemaIgnore` 可用于排除这些字段。请注意，忽略的字段不会包含在这些记录的编码中。

**Java Bean** 是在 Java 中创建可重用属性类的事实上的标准。虽然完整的标准有许多特征，但关键是所有属性都通过 getter 和 setter 类访问，并且这些 getter 和 setter 的名称格式是标准化的。
Java Bean 类可以使用 `@DefaultSchema(JavaBeanSchema.class)` 进行注释，Beam 将自动推断该类的模式。例如：
```java
@DefaultSchema(JavaBeanSchema.class)
public class TransactionBean {
  public TransactionBean() { … }
  public String getBank() { … }
  public void setBank(String bank) { … }
  public double getPurchaseAmount() { … }
  public void setPurchaseAmount(double purchaseAmount) { … }
}
// Beam will automatically infer the correct schema for this PCollection. No coder is needed as a result.
PCollection<TransactionBean> beans = readBeans();
```
`@SchemaCreate` 注释可用于指定构造函数或静态工厂方法，在这种情况下，可以省略 setter 和零参数构造函数。
```java
@DefaultSchema(JavaBeanSchema.class)
public class TransactionBean {
  @SchemaCreate
  Public TransactionBean(String bank, double purchaseAmount) { … }
  public String getBank() { … }
  public double getPurchaseAmount() { … }
}
```
`@SchemaFieldName` 和 `@SchemaIgnore` 可用于更改推断的Schema，就像 POJO 类一样。

**AutoValue**
众所周知，Java 值类很难正确生成。为了正确实现值类，必须创建大量样板文件。 AutoValue 是一个流行的lib，可通过实现简单的抽象基类轻松生成此类，Beam 可以从 AutoValue 类推断模式。例如：
```java
@DefaultSchema(AutoValueSchema.class)
@AutoValue
public abstract class TransactionValue {
  public abstract String getBank();
  public abstract double getPurchaseAmount();
}
```
这就是生成一个简单的 AutoValue 类所需的全部内容，上面的 `@DefaultSchema` 注释告诉 Beam 从中推断出Schema。这还允许在 `PCollection` 内部使用 AutoValue 元素。
### Using Schema Transforms
`PCollection` 上的 Schema 支持丰富多样的关系转换。
每条记录都由命名字段组成，这一事实允许通过名称引用字段的简单且可读的聚合，类似于 SQL 表达式中的聚合。
#### Field selection syntax
Schema 的优点是它们允许按名称引用元素字段。 Beam 提供了用于引用字段（包括嵌套字段和重复字段）的选择语法。
所有 Schema 转换在引用它们所操作的字段时都使用此语法。该语法还可以在 DoFn 内部使用来指定要处理的Schema 字段。

按名称寻址字段仍然保留类型安全性，因为 Beam 将在构造 Pipeline graph 时检查 Schema 是否匹配。
	如果指定的字段在 Schema 中不存在，Pipeline 将无法启动。
	如果指定字段的类型与 Schema 中该字段的类型不匹配，Pipeline 将无法启动。
	字段名称中不允许使用以下字符： `。 * [ ] { }`
##### **Top-level fields 顶级字段**
为了选择 Schema 顶层的字段，需要指定该字段的名称。例如，要仅从 `PCollection` 购买中选择用户 ID（使用 `Select` transform）
```java
purchases.apply(Select.fieldNames("userId"));
```
##### **Nested fields 嵌套字段**
可以使用点运算符`.`指定各个嵌套字段。例如，要仅从送货地址中选择邮政编码，可以写:
```java
purchases.apply(Select.fieldNames("shippingAddress.postCode"));
```
##### **Wildcards 通配符**
可以在任何嵌套级别指定 `*`运算符来表示该级别的所有字段。例如，要选择所有送货地址字段，可以写:
```java
purchases.apply(Select.fieldNames("shippingAddress.*"));
```
##### **Arrays 数组**
数组字段（其中数组元素类型是行）也可以具有所寻址的元素类型的子字段。选择后，结果是所选子字段类型的数组。例如:
```java
purchases.apply(Select.fieldNames("transactions[].bank"));
```
##### **Maps**
```java
purchasesByType.apply(Select.fieldNames("purchases{}.userId"));
```

#### Schema transforms 模式转换
Beam 提供了一系列可对 Schema 进行 Native 操作的 Transform。这些 Transform非常具有表现力，允许根据命名 Schema 字段进行选择和聚合。以下是一些有用的模式转换的示例。

通常情况下，计算只对输入 PCollection 中的部分字段感兴趣。通过选择转换，用户可以轻松地只选出感兴趣的字段。生成的 PCollection 有一个 Schema，其中包含作为顶层字段的每个选定字段。顶层字段和嵌套字段都可以选择。例如，在 "Purchase " Schema 中，可以只选择 userId 和 streetAddress 字段，如下所示:
```java
purchases.apply(Select.fieldNames("userId", "shippingAddress.streetAddress"));
```
生成的 `PCollection` 将具有以下 Schema

| **Field Name** | **Field Type** |
| ----------------------- | ----------------------- |
| userId         | STRING              |
| streetAddress  | STRING             |

对于通配符选择也是如此。
```java
purchases.apply(Select.fieldNames("userId", "shippingAddress.*"));
```

... 其他转换参考 [Beam schame transform](https://beam.apache.org/documentation/programming-guide/#schemas)

##### **Converting between types 类型之间的转换**
如前所述，Beam 可以在不同的 Java 类型之间自动转换，只要这些类型具有等效的Schema。实现此目的的一种方法是使用 `Convert` 转换，如下所示:
```java
PCollection<PurchaseBean> purchaseBeans = readPurchasesAsBeans();
PCollection<PurchasePojo> pojoPurchases =
    purchaseBeans.apply(Convert.to(PurchasePojo.class));
```
beam 将验证 `PurchasePojo` 的推断 Schema 是否与输入 `PCollection` 的 Schema 匹配，然后转换为 `PCollection<PurchasePojo>` 。

由于 `Row` 类可以支持任何Schema，因此具有 Schema 的任何 `PCollection` 都可以转换为 `PCollection` 行，如下所示:
```java
PCollection<Row> purchaseRows = purchaseBeans.apply(Convert.toRows());
```

如果源类型是单字段 Schema，则 Convert 还将根据要求转换为字段的类型，从而有效地取消unboxing。
例如，给定一个具有单个 INT64 字段的Schema，以下内容会将其转换为 `PCollection<Long>`:
```java
PCollection<Long> longs = rows.apply(Convert.to(TypeDescriptors.longs()));
```

#### Schemas in ParDo
具有 Schema 的 `PCollection` 可以应用 `ParDo` ，就像任何其他 `PCollection` 一样。
Beam runner 在应用 `ParDo` 时会意识到Schema，从而启用附加功能。
##### **Input conversion 输入转换**
由于 Beam 知道源 `PCollection` 的Schema，因此它可以自动将元素转换为已知匹配Schema的任何 Java 类型。例如，使用上述事务模式，假设我们有以下 `PCollection`:
```java
PCollection<PurchasePojo> purchases = readPurchases();
```

如果没有Schema，则应用的 `DoFn` 将必须接受 `TransactionPojo` 类型的元素。但由于存在Schema，可以应用以下 DoFn：
```java
purchases.apply(ParDo.of(new DoFn<PurchasePojo, PurchasePojo>() {
  @ProcessElement public void process(@Element PurchaseBean purchase) {
      ...
  }
}));
```
即使 `@Element` 参数与 `PCollection` 的 Java 类型不匹配，因为它具有匹配的Schema，Beam 也会自动转换元素。如果 Schema 不匹配，Beam 将在pipeline graph 构建时检测到这一点，会提示作业失败并出现类型错误。
	由于每个模式都可以用 Row 类型表示，因此这里也可以使用 Row：
```java
purchases.appy(ParDo.of(new DoFn<PurchasePojo, PurchasePojo>() {
  @ProcessElement public void process(@Element Row purchase) {
      ...
  }
}));
```
##### **Input selection 输入选择**
鉴于上述购买 `PCollection` ，假设您只想处理 userId 和 itemId 字段。您可以使用上述选择表达式来执行这些操作，如下所示：
```java
purchases.appy(ParDo.of(new DoFn<PurchasePojo, PurchasePojo>() {
  @ProcessElement public void process(
     @FieldAccess("userId") String userId, @FieldAccess("itemId") long itemId) {
      ...
  }
}));
```