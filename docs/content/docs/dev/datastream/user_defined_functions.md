---
title: User-Defined Functions
weight: 5
type: docs
aliases:
  - /dev/user_defined_functions.html
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

# User-Defined Functions

Most operations require a user-defined function. This section lists different
ways of how they can be specified. We also cover `Accumulators`, which can be
used to gain insights into your Flink application.

## Implementing an interface

The most basic way is to implement one of the provided interfaces:

```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
}
data.map(new MyMapFunction());
```

## Anonymous classes

You can pass a function as an anonymous class:
```java
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

## Java 8 Lambdas

Flink also supports Java 8 Lambdas in the Java API.

```java
data.filter(s -> s.startsWith("http://"));
```

```java
data.reduce((i1,i2) -> i1 + i2);
```

## Rich functions

All transformations that require a user-defined function can
instead take as argument a *rich* function. For example, instead of

```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
}
```

you can write

```java
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
}
```

and pass the function as usual to a `map` transformation:

```java
data.map(new MyMapFunction());
```

Rich functions can also be defined as an anonymous class:
```java
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

## Accumulators & Counters

Accumulators are simple constructs with an **add operation** and a **final accumulated result**,
which is available after the job ended.

The most straightforward accumulator is a **counter**: You can increment it using the
```Accumulator.add(V value)``` method. At the end of the job Flink will sum up (merge) all partial
results and send the result to the client. Accumulators are useful during debugging or if you
quickly want to find out more about your data.

Flink currently has the following **built-in accumulators**. Each of them implements the
{{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java" name="Accumulator" >}}
interface.

- {{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/IntCounter.java" name="__IntCounter__" >}},
  {{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/LongCounter.java" name="__LongCounter__" >}}
  and {{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/DoubleCounter.java" name="__DoubleCounter__" >}}:
  See below for an example using a counter.
- {{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/Histogram.java" name="__Histogram__" >}}:
  A histogram implementation for a discrete number of bins. Internally it is just a map from Integer
  to Integer. You can use this to compute distributions of values, e.g. the distribution of
  words-per-line for a word count program.

__How to use accumulators:__

First you have to create an accumulator object (here a counter) in the user-defined transformation
function where you want to use it.

```java
private IntCounter numLines = new IntCounter();
```

Second you have to register the accumulator object, typically in the ```open()``` method of the
*rich* function. Here you also define the name.

```java
getRuntimeContext().addAccumulator("num-lines", this.numLines);
```

You can now use the accumulator anywhere in the operator function, including in the ```open()``` and
```close()``` methods.

```java
this.numLines.add(1);
```

The overall result will be stored in the ```JobExecutionResult``` object which is
returned from the `execute()` method of the execution environment
(currently this only works if the execution waits for the
completion of the job).

```java
myJobExecutionResult.getAccumulatorResult("num-lines");
```

All accumulators share a single namespace per job. Thus you can use the same accumulator in
different operator functions of your job. Flink will internally merge all accumulators with the same
name.

__Custom accumulators:__

To implement your own accumulator you simply have to write your implementation of the Accumulator
interface. Feel free to create a pull request if you think your custom accumulator should be shipped
with Flink.

You have the choice to implement either
{{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/Accumulator.java" name="Accumulator" >}}
or {{< gh_link file="/flink-core/src/main/java/org/apache/flink/api/common/accumulators/SimpleAccumulator.java" name="SimpleAccumulator" >}}.

```Accumulator<V,R>``` is most flexible: It defines a type ```V``` for the value to add, and a
result type ```R``` for the final result. E.g. for a histogram, ```V``` is a number and ```R``` is
 a histogram. ```SimpleAccumulator``` is for the cases where both types are the same, e.g. for counters.

{{< top >}}
