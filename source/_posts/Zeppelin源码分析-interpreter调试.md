---
title: Zeppelin源码分析-interpreter调试
date: 2019-06-12 10:57:37
tags:
    - Zeppelin
---

前面提到了interpreter是以单独的process启动的，想要debug interpreter，需要设置启动interpreter进程的jvm以debug方式启动，然后让IDE进行remote debug，具体步骤如下：

1. 在bin/interpreter.sh脚本中JAVA_INTP_OPTS变量中加入如下参数：

```bash
JAVA_INTP_OPTS+=" -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=**`expr ${PORT} + 1`**  -Dzeppelin.log.file=${ZEPPELIN_LOGFILE}"
```

加粗部分保证启动interpreter的jvm以debug方式启动，监听的端口号比RemoteInterpreterServer process监听的端口号+1（采用`expr${PORT} + 1`）这里不能写成固定的端口，因为每种interpreter都会启动一个独立的process，该process监听的socket端口是zeppelin在运行时随机获取一个可用的端口（没有被占用的端口）。如果写成固定的端口，那么每种interpreter process在进行remote debug的时候，端口就会冲突。

2. 启动ZeppelinServer的调试，可以直接run，不用以debug方式启动。
3. 打开浏览器，访问http://localhost:8080，（如果在shiro.ini中配置了auth，则需要登录），然后创建一个Note，interpreter binding了spark。insert一个paragraph，首行写入%spark (表明采用SparkInterpreter来解释此paragrapph中的代码)，换行，写入scala代码println(“hello,world”)，由于这里主要演示debug interpreter，以最简单的hello world说明问题。然后点击运行该paragraph。此时zeppelin server会调用bin/interpreter.sh脚本，传入“端口、interpreter加载的目录(这里是${ZEPPELIN_HOME}/interpreter/spark)、interpreter各自的log file位置”这些参数以启动该interpreter jvm。由于之前在JAVA_INTP_OPTS设置了jvm支持远程调试的参数，该jvm可以通过IDE remote debug。
4. 通过ps –ef|grep interpreter来查找RemoteInterpreterServer process启动的参数，可以看到-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=58679类似的输出，表明该jvm在58679端口上支持remote debug。

在SparkInterpreter.java中的重点位置设置断点，如在方法
```java
open()和interpret(String line, InterpreterContext context) 
```
首行设置断点。
5. 在IDE，以Idea为例：创建Remote Run/Debug Configuration，填入端口号58679

{%asset_img 1.png %}


即可启动remote debug。

需要注意的是，由于hello world代码简短，可能在你在IDE中启动Remote debug时该Interpreter已经执行完了。再次点击执行该paragraph即可命中SparkInterpreter中的断点。
