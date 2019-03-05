---
title: Chaining Custom PySpark DataFrame Transformations
date: 2019-01-19 14:49:09
tags:
    - Spark
    - Python
    - PySpark
---

和基于Scala Spark API一样，我们也期望在Pyspark中也能够将不同的transformation方法连接起来统一执行。

这篇博客主要描述如何给Pyspark DataFrame添加`transform`方法，从而支持能够将自定义的DataFrame transformation连接起来。

同时，我们也会介绍如何利用[cytoolz](https://github.com/pytoolz/cytoolz)来批量顺序执行自定义变换函数

## Chaining DataFrame Transformations with Lambda

首先，在原生的Pyspark DataFrame增加`transform`方法，从而我们能够串联执行DataFrame变换

```python
from pyspark.sql.dataframe import DataFrame

def transform(self, f):
    return f(self)
    
DataFrame.transform = transform
```

接下来，我们定义一些简单的DataFrame变换方法来测试`transform`

```python
def with_greeting(df):
    return df.withColumn("greeting", lit("hi"))
    
def with_something(df, something):
    return df.withColumn("something", lit(something))
```

创建一个DataFrame然后串联执行`with_greeting`和`with_something`

```python
data = [("jsoe", 1), ("li", 2), ("liz", 3)]
source_df = spark.createDataFrame(data, ["name", "age"])

actual_df = (source_df
                .transform(lambda df: with_greeting(df))
                .transform(lambda df: with_something(df, "crazy")))
                
print(actual_df.show())

+----+---+--------+---------+
|name|age|greeting|something|
+----+---+--------+---------+
|jose|  1|      hi|    crazy|
|  li|  2|      hi|    crazy|
| liz|  3|      hi|    crazy|
+----+---+--------+---------+
```

对于只有一个DataFrame参数的自定义变换中，`lambda`是可以省略的，从而我们可以简化调用方式如下

```python
actual_df = (source_df
             .transform(with_greeting)
             .transform(lambda df: with_something(df, "crazy")))
```

如果我们没有定义`DataFrame#transform`方法，我们就不需要像下面的代码一样，来调用不同的transformation

```python
df1 = with_greeting(source_df)
actual_df = with_something(df1, "moo")
```

比较上述代码，采用`transform`来调用不同的DataFrame的变化，能够避免定义中间DataFrame，从而使得代码更加清晰

接下来，我们进一步探讨transformations的其他定义形式，让整个`transform`更加清晰明了

<!--more-->

## Chaining DataFrame Transformations with functools.partial

定义一个`with_jacket` DataFrame变换，该变换将会增加`jacket`列到原始DataFrame中

```python
def with_jacket(word, df):
    return df.withColumn("jacket", lit(word))
```

我们使用上文中同样的`source_df`和 `with_greeting`方法，采用functools.partial来连接不同的变换

```python
from functools import partial

actual_df = (source_df
             .transform(with_greeting)
             .transform(partial(with_jacket, "warm")))
             
print(actual_df.show())

+----+---+--------+------+
|name|age|greeting|jacket|
+----+---+--------+------+
|jose|  1|      hi|  warm|
|  li|  2|      hi|  warm|
| liz|  3|      hi|  warm|
+----+---+--------+------+

```

从上文可以看出，`functools.partial`能够帮助我们节省掉写`lambda`， 不过我们可以做的更好

## Defining DataFrame transformations as nested functions

利用嵌套方式来定义DataFrame Transformation，是一种更为优雅的方式解决连接调用，我们定义一个`with_funny`防暑，并且增加一列`funny`到DataFrame

```python
def with_funny(word):
    def inner(df):
        return df.withColumn("funny", lit(word))
    return inner
```

同样采用上文中的`source_df`和`with_greeting`

```python
actual_df = (source_df
            .transform(with_greeting)
            .transform(with_funny("haha)))
            
print(actual_df.show())

+----+---+--------+-----+
|name|age|greeting|funny|
+----+---+--------+-----+
|jose|  1|      hi| haha|
|  li|  2|      hi| haha|
| liz|  3|      hi| haha|
+----+---+--------+-----+
```

以上，我们可以发现，已经可以完全摆脱`lambda`关键字，调用的方式和Scala API基本一致

## Function composition with cytoolz

我们可以在定义DataFrame transformation的时候，增加`@curry`装饰器，并且利用`cytoolz`提供的composition函数来运行他们

```python
from cytoolz import curry
from cytoolz.functoolz import compose

@curry
def with_stuff1(arg1, arg2, df):
    return df.withColumn("stuff1", lit(f"{arg1} {arg2}"))
    
@curry
def with_stuff2(arg, df):
    return df.withColumn("stuff2", lit(arg))
    
data = [("jose", 1), ("li", 2), ("liz", 3)]
source_df = spark.createDataFrame(data, ["name", "age"])


pipeline = compose(
    with_stuff1("nice", "person"),
    with_stuff2("yoyo")
)
actual_df = pipeline(source_df)

print(actual_df.show())
+----+---+------+-----------+
|name|age|stuff2|     stuff1|
+----+---+------+-----------+
|jose|  1|  yoyo|nice person|
|  li|  2|  yoyo|nice person|
| liz|  3|  yoyo|nice person|
+----+---+------+-----------+

```

但是需要注意的是,`compose`函数是从右往左(下往上)执行，因此为了能够满足从上往下执行的习惯，需要做如下修改

```python
pipeline = compose(*reversed([
    with_stuff1("nice", "person"),
    with_stuff2("yoyo")
]))
actual_df = pipeline(source_df)

print(actual_df.show())
+----+---+-----------+------+
|name|age|     stuff1|stuff2|
+----+---+-----------+------+
|jose|  1|nice person|  yoyo|
|  li|  2|nice person|  yoyo|
| liz|  3|nice person|  yoyo|
+----+---+-----------+------+
```

