---
title: Chaining Custom DataFrame Transformations in Spark
date: 2019-01-19 13:50:57
tags:
    - Spark
    - Chain
    - Transform
    - Scala
---

在Spark中，可以采用`implicit classes`或者`Dataset#transform`来连接DataFrame的变换，这篇博客着重描如何连接DataFrame变换操作，并且详细解释说明为啥`Dataset#transform`方式要比`implicit classes`更有优势。

## Dataset Transform方法

Dataset transform方法提供了[concise syntax for chaining custom transformations](http://spark.apache.org/docs/latest/api/scala/#org.apache.spark.sql.Dataset)

假如我们有两个方法

1. `withGreeting()`方法，该方法在原有的DataFrame基础上增加`greeting`列
2. `withFarewell()`方法，该方法在原有的DataFrame基础上增加`farewell`列

```scala
def withGreeting(df: DataFrame): DataFrame = {
    df.withColumn("greeting", lit("hello world"))
}

def withFarewell(df: DataFrame): DataFrame = {
    df.withColumn("farewell", lit("goodbye"))
}
```

<!--more-->


我们可以利用`transform`方法来运行`withGreeting()`和`withFarewell()`

```scala
val df = Seq(
    "funny",
    "persion"
).toDF("something")

val weirdDf = df
    .transform(withGreeting)
    .transform(withFarewell)
    
weirdDf.show()

+---------+-----------+--------+
|something|   greeting|farewell|
+---------+-----------+--------+
|    funny|hello world| goodbye|
|   person|hello world| goodbye|
+---------+-----------+--------+

```

当然，`transform`方法也可以很容易和Spark DataFrame内置的方法结合在一起使用，例如`select`

```scala
df
  .select("something")
  .transform(withGreeting)
  .transform(withFarewell)
```

如果我们不使用`transform`，那么未来实现相同的功能，代码需要层层嵌套，大大降低了代码的可读性

```scala
withFarewell(withGreeting(df))

// even worse

withFarewell(withGreeting(df)).select("something")
```

## Transform Method with Arguments

带参数的自定义DataFrame变化方法也能够通过`transform`连接起来，但是必须要采用scala中的currying方式来实现

我们依然以上面的DataFrame为例，增加一个带有一个string参数的变化方法

```scala
def withGreeting(df:DataFrame): DataFrame = {
    df.withColumn("greeting", lit("Hello World"))
}

def withCat(name: String)(df: DataFrame): DataFrame = {
    df.withColumn("cats", lit(s"$name meow"))
}
```

这时，我们可以使用`transform`方法将`withGreeting`和`withCat`连接起来

```scala
val df = Seq(
    "funny",
    "person"
).toDF("something")


val niceDf = df
      .transform(withGreeting)
      .transform(withCat("puffy"))
      
niceDf.show()

+---------+-----------+----------+
|something|   greeting|      cats|
+---------+-----------+----------+
|    funny|hello world|puffy meow|
|   person|hello world|puffy meow|
+---------+-----------+----------+
```


## Monkey Patching with Implicit Classes

Implicit classes能够将方法加入到已经存在的勒种，下面的例子就是将`withGreeting`和`withFarewell`加入到DataFrame这个类本身中

```scala
object BadImplicit{
    implicit class DataFrameTransforms(df: DataFrame) {
        def withGreeting(): DataFrame = {
            df.withColumn("greeting", lit("Hello world"))
        }
        def withFarewell(): DataFrame = {
            df.withColumn("farewell", lit("goodbye"))
        }
    }
}
```

此时`withGreeting()`和`withFarewell`可以采用下面的方式串联起来执行

```scala
import BadImplicit._

val df = Seq(
    "funny",
    "persion"
).toDF("something")

val hiDF = df.withGreeting().withFarewell()
```

## Avoid Implicit Classes

> Changing base classes is known as monkey patching and is a delightful feature of Ruby but can be perilous in untutored hands. — Sandi Metz

虽然Sandi的评论针对的是Ruby语音，但是这个准则同样适用于scala
因此，在实际项目中，不建议采用Implici Classes， 况且，Spark已经提供了非常好用的`transform`方法解决多个变化连接执行的问题。

