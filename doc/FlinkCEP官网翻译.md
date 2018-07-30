
# Flink CEP
## 1. 本文概述简介

FlinkCEP是在Flink之上实现的复杂事件处理（CEP）库。 它允许您在无休止的事件流中检测事件模式，让您有机会掌握数据中重要的事项。

本文描述了Flink CEP中可用的API调用。 我们首先介绍Pattern API，它允许您指定要在流中检测的模式，然后介绍如何检测匹配事件序列并对其进行操作。 
然后，我们将介绍CEP库在处理事件时间延迟时所做的假设，以及如何将您的工作从较旧的Flink版本迁移到Flink-1.3。

## 2.入门

首先是要在你的pom.xml文件中，引入CEP库。
```java
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep_2.11</artifactId>
  <version>1.5.0</version>
</dependency>
```
注意要应用模式匹配的DataStream中的事件必须实现正确的equals（）和hashCode（）方法，因为FlinkCEP使用它们来比较和匹配事件。

第一个demo如下：
```java
DataStream<Event> input = ...

Pattern<Event, ?> pattern = Pattern.<Event>begin("start").where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getId() == 42;
            }
        }
    ).next("middle").subtype(SubEvent.class).where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(SubEvent subEvent) {
                return subEvent.getVolume() >= 10.0;
            }
        }
    ).followedBy("end").where(
         new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getName().equals("end");
            }
         }
    );

PatternStream<Event> patternStream = CEP.pattern(input, pattern);

DataStream<Alert> result = patternStream.select(
    new PatternSelectFunction<Event, Alert> {
        @Override
        public Alert select(Map<String, List<Event>> pattern) throws Exception {
            return createAlertFrom(pattern);
        }
    }
});
```

## 2.Pattern API

Pattern API允许你定义要从输入流中提取的复杂模式序列。

每个复杂模式序列由多个简单模式组成，即寻找具有相同属性的单个事件的模式。我们可以先定义一些简单的模式，然后组合成复杂的模式序列。
可以将模式序列视为此类模式的结构图，其中从一个模式到下一个模式的转换基于用户指定的条件发生，例如， event.getName().equals("start")。
匹配是一系列输入事件，通过一系列有效的模式转换访问复杂模式图的所有模式。

注意每个模式必须具有唯一的名称，稍后可以使用该名称来标识匹配的事件。

注意模式名称不能包含字符“：”。

在本节接下来的部分，我们将首先介绍如何定义单个模式，然后如何将各个模式组合到复杂模式中。

### 2.1 单个模式
Pattern可以是单单个，也可以是循环模式。单个模式接受单个事件，而循环模式可以接受多个事件。在模式匹配符号中，模式“a b + c？d”（或“a”，后跟一个或多个“b”，可选地后跟“c”，后跟“d”），a，c ？，和d是单例模式，而b +是循环模式。
默认情况下，模式是单个模式，您可以使用Quantifiers将其转换为循环模式。每个模式可以有一个或多个条件，基于它接受事件。

### 2.2 Quantifiers
在FlinkCEP中，您可以使用以下方法指定循环模式：pattern.oneOrMore（），用于期望一个或多个事件发生的模式（例如之前提到的b +）;和pattern.times（#ofTimes），
用于期望给定类型事件的特定出现次数的模式，例如4个;和patterntimes（#fromTimes，＃toTimes），用于期望特定最小出现次数和给定类型事件的最大出现次数的模式，例如， 2-4为。

您可以使用pattern.greedy（）方法使循环模式变得贪婪，但是您还不能使组模式变得贪婪。您可以使用pattern.optional（）方法使得所有模式，循环与否，可选。

对于名为start的模式，以下是有效的Quantifiers：

```java
 // expecting 4 occurrences
 start.times(4);

 // expecting 0 or 4 occurrences
 start.times(4).optional();

 // expecting 2, 3 or 4 occurrences
 start.times(2, 4);

 // expecting 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).greedy();

 // expecting 0, 2, 3 or 4 occurrences
 start.times(2, 4).optional();

 // expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).optional().greedy();

 // expecting 1 or more occurrences
 start.oneOrMore();

 // expecting 1 or more occurrences and repeating as many as possible
 start.oneOrMore().greedy();

 // expecting 0 or more occurrences
 start.oneOrMore().optional();

 // expecting 0 or more occurrences and repeating as many as possible
 start.oneOrMore().optional().greedy();

 // expecting 2 or more occurrences
 start.timesOrMore(2);

 // expecting 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).greedy();

 // expecting 0, 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).optional().greedy();
 
```

### 2.3 Conditions-条件

在每个模式中，从一个模式转到下一个模式，可以指定其他条件。您可以将使用下面这些条件：

1. 传入事件的属性，例如其值应大于5，或大于先前接受的事件的平均值。

2. 匹配事件的连续性，例如检测模式a，b，c，序列中间不能有任何非匹配事件。

#### 2.3.1 Conditions on Properties-关于属性的条件

可以通过pattern.where（），pattern.or（）或pattern.until（）方法指定事件属性的条件。 这些可以是IterativeConditions或SimpleConditions。


1. 迭代条件：

这是最常见的条件类型。 你可以指定一个条件，该条件基于先前接受的事件的属性或其子集的统计信息来接受后续事件。

下面代码说的是：如果名称以“foo”开头，则迭代条件接受名为“middle”的模式的下一个事件，并且如果该模式的先前接受的事件的价格总和加上当前事件的价格不超过该值 5.0。 
迭代条件可以很强大的，尤其是与循环模式相结合，例如， oneOrMore()。

````java
middle.oneOrMore().where(new IterativeCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value, Context<SubEvent> ctx) throws Exception {
        if (!value.getName().startsWith("foo")) {
            return false;
        }

        double sum = value.getPrice();
        for (Event event : ctx.getEventsForPattern("middle")) {
            sum += event.getPrice();
        }
        return Double.compare(sum, 5.0) < 0;
    }
});
````

注意对context.getEventsForPattern（...）的调用,将为给定潜在匹配项查找所有先前接受的事件。 此操作的代价可能会有所不同，因此在使用条件时，请尽量减少其使用。

2. 简单条件：

这种类型的条件扩展了前面提到的IterativeCondition类，并且仅根据事件本身的属性决定是否接受事件。

```java
start.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return value.getName().startsWith("foo");
    }
});
```

最后，您还可以通过pattern.subtype（subClass）方法将接受事件的类型限制为初始事件类型（此处为Event）的子类型。

```java
start.subtype(SubEvent.class).where(new SimpleCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value) {
        return ... // some condition
    }
});
```

3. 组合条件：

如上所示，您可以将子类型条件与其他条件组合使用。 这适用于所有条件。 您可以通过顺序调用where（）来任意组合条件。
最终结果将是各个条件的结果的逻辑AND。 要使用OR组合条件，可以使用or（）方法，如下所示。

```java
pattern.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // some condition
    }
}).or(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // or condition
    }
});
```

4. 停止条件：

在循环模式（oneOrMore()和oneOrMore().optional()）的情况下，您还可以指定停止条件，例如： 接受值大于5的事件，直到值的总和小于50。


