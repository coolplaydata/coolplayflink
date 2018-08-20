

# 案例简介flink CEP

随着无处不在的传感器网络和智能设备不断收集越来越多的数据，我们面临着以近实时的方式分析不断增长的数据流的挑战。 
能够快速响应不断变化的趋势或提供最新的商业智能可能是公司成功或失败的决定性因素。 实时处理中的关键问题是检测数据流中的事件模式。

复杂事件处理（CEP）恰好解决了对连续传入事件进行模式匹配的问题。 匹配的结果通常是从输入事件派生的复杂事件。
与对存储数据执行查询的传统DBMS相比，CEP在存储的查询上执行数据。 可以立即丢弃与查询无关的所有数据。 
考虑到CEP查询应用于潜在的无限数据流，这种方法的优势是显而易见的。 此外，输入立即处理。 一旦系统看到匹配序列的所有事件，结果就会立即发出。
这方面有效地带来了CEP的实时分析能力。

因此，CEP的处理范例引起了人们的极大兴趣，并在各种用例中得到了应用。 最值得注意的是，CEP现在用于诸如股票市场趋势和信用卡欺诈检测等金融应用。 此外，它用于基于RFID的跟踪和监控，例如，用于检测仓库中的物品未被正确检出的盗窃。 通过指定可疑用户行为的模式，CEP还可用于检测网络入侵。

Apache Flink具有真正的流处理特性以及低延迟和高吞吐量流处理功能，非常适合CEP工作负载。

## 栗子
案例是对数据中心进行监控告警。

![image](../pic/CEP/cep-monitoring.svg)

假设我们有一个带有多个机架的数据中心。 对于每个机架，都会监控功耗和温度。 无论何时发生这种测量，分别产生新的功耗或温度事件。 基于此监控事件流，我们希望检测即将过热的机架，并动态调整其工作负载和对其降温。

对于这种情况，我们使用两阶段方法。 首先，我们监测温度事件。 每当我们看到温度超过阈值的两个连续事件时，我们就会产生一个温度警告，其中包含当前的平均温度。 
温度警告不一定表示机架即将过热。 但是，每当我们看到连续两次警告温度升高时，我们就会发出此机架的警报。 然后，该警报可以触发对冷却机架的对策。


## 使用Apache Flink实现

首先，我们定义传入监视事件流的消息。 每条监控消息都包含其原始机架ID。 温度事件还包含当前温度，功耗事件包含当前电压。 我们将事件建模为POJO：
```java
public abstract class MonitoringEvent {
    private int rackID;
    ...
}

public class TemperatureEvent extends MonitoringEvent {
    private double temperature;
    ...
}

public class PowerEvent extends MonitoringEvent {
    private double voltage;
    ...
}
```
现在我们可以使用Flink的一个连接器（例如Kafka，RabbitMQ等）来摄取监视事件流。 这将为我们提供一个DataStream <MonitoringEvent> inputEventStream，我们将其用作Flink的CEP运算符的输入。
 但首先，我们必须定义事件模式以检测温度警告。 CEP库提供了一个直观的Pattern API，可以轻松定义这些复杂的模式。

每个模式都由一系列事件组成，这些事件可以分配可选的过滤条件。 模式始终以第一个事件开始，我们将为其指定名称“First Event”。

```java
Pattern.<MonitoringEvent>begin("First Event");
```

此模式将匹配每个监视事件。 由于我们只对温度高于阈值的TemperatureEvents感兴趣，因此我们必须添加一个额外的子类型约束和一个where子句：

```java
Pattern.<MonitoringEvent>begin("First Event")
    .subtype(TemperatureEvent.class)
    .where(evt -> evt.getTemperature() >= TEMPERATURE_THRESHOLD);
```

如前所述，当且仅当我们在温度过高的同一机架上看到两个连续的TemperatureEvent时，我们才想生成温度警告。 Pattern API提供next调用，允许我们向模式添加新事件。 
此事件必须直接跟随第一个匹配事件，以便整个模式匹配。

```java
Pattern<MonitoringEvent, ?> warningPattern = Pattern.<MonitoringEvent>begin("First Event")
    .subtype(TemperatureEvent.class)
    .where(evt -> evt.getTemperature() >= TEMPERATURE_THRESHOLD)
    .next("Second Event")
    .subtype(TemperatureEvent.class)
    .where(evt -> evt.getTemperature() >= TEMPERATURE_THRESHOLD)
    .within(Time.seconds(10));
```
最终模式定义还包含内部API调用，该调用定义了两个连续的TemperatureEvent必须在10秒的时间间隔内发生以使模式匹配。 根据时间特性设置，这可以是处理，注入或事件时间。

定义了事件模式后，我们现在可以将它应用于inputEventStream。

```java
PatternStream<MonitoringEvent> tempPatternStream = CEP.pattern(
    inputEventStream.keyBy("rackID"),
    warningPattern);
```

由于我们希望单独为每个机架生成警告，因此我们通过“rackID”POJO字段 keyby 输入事件流。 这会强制我们模式的匹配事件都具有相同的机架ID。

PatternStream <MonitoringEvent>使我们能够访问成功匹配的事件序列。 可以使用select API调用访问它们。 select API采用PatternSelectFunction，为每个匹配的事件序列调用。
事件序列以Map <String，MonitoringEvent>的形式提供，其中每个MonitoringEvent由其指定的事件名称标识。 我们的模式选择函数为每个匹配模式生成一个TemperatureWarning事件。

```java
public class TemperatureWarning {
    private int rackID;
    private double averageTemperature;
    ...
}

DataStream<TemperatureWarning> warnings = tempPatternStream.select(
    (Map<String, MonitoringEvent> pattern) -> {
        TemperatureEvent first = (TemperatureEvent) pattern.get("First Event");
        TemperatureEvent second = (TemperatureEvent) pattern.get("Second Event");

        return new TemperatureWarning(
            first.getRackID(), 
            (first.getTemperature() + second.getTemperature()) / 2);
    }
);
```

现在我们从初始监视事件流生成了一个新的复杂事件流DataStream <TemperatureWarning>警告。 此复杂事件流可再次用作另一轮复杂事件处理的输入。
每当我们看到温度升高的同一机架连续两次温度警报时，我们就会使用TemperatureWarnings生成TemperatureAlerts。 TemperatureAlerts具有以下定义：

```java
public class TemperatureAlert {
    private int rackID;
    ...
}
```
首先，我们必须定义警报事件模式：

```java
Pattern<TemperatureWarning, ?> alertPattern = Pattern.<TemperatureWarning>begin("First Event")
    .next("Second Event")
    .within(Time.seconds(20));

```

这个定义说我们希望在20秒内看到两个温度警报。 第一个事件的名称为“First Event”，第二个连续的事件的名称为“Second Event”。 
单个事件没有分配where子句，因为我们需要访问这两个事件以确定温度是否在增加。 因此，我们在select子句中应用过滤条件。 但首先，我们再次获得一个PatternStream。

```java
PatternStream<TemperatureWarning> alertPatternStream = CEP.pattern(
    warnings.keyBy("rackID"),
    alertPattern);
```

同样，我们通过“rackID” keyby 警告输入流，以便我们单独为每个机架生成警报。 接下来，我们应用flatSelect方法，该方法将允许我们访问匹配的事件序列，并允许我们输出任意数量的复杂事件。
因此，当且仅当温度升高时，我们才会生成TemperatureAlert。

```java
DataStream<TemperatureAlert> alerts = alertPatternStream.flatSelect(
    (Map<String, TemperatureWarning> pattern, Collector<TemperatureAlert> out) -> {
        TemperatureWarning first = pattern.get("First Event");
        TemperatureWarning second = pattern.get("Second Event");

        if (first.getAverageTemperature() < second.getAverageTemperature()) {
            out.collect(new TemperatureAlert(first.getRackID()));
        }
    });
```

````java
DataStream <TemperatureAlert>警报是每个机架的温度警报的数据流。 基于这些警报，我们现在可以调整过热架的工作负载或冷却。
````

## 结论
在这篇博文中，我们已经看到使用Flink的CEP库推理事件流是多么容易。 使用数据中心监控和警报生成的示例，我们实施了一个简短的程序，当机架即将过热并可能发生故障时通知我们。

在未来，Flink社区将进一步扩展CEP库的功能和表现力。 路线图上的下一步是支持正则表达式模式规范，包括Kleene星 (Kleene star)，下限和上限( lower and upper bounds)以及否定(negation)。
此外，计划允许where子句访问先前匹配的事件的字段。 此功能将允许尽早修剪无意义的事件序列。

##  
[@完整代码链接](https://github.com/tillrohrmann/cep-monitoring)



































