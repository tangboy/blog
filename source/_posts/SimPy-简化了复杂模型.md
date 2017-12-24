---
title: SimPy 简化了复杂模型
date: 2017-11-06 14:36:10
tags:
    - python
    - Simpy
---

在我遇到 SimPy 包的其中一位创始人 Klaus Miller 时，从他那里知道了这个包。Miller 博士阅读过几篇提出使用 Python 2.2+ 生成器实现半协同例程和“轻便”线程的技术的 可爱的 Python专栏文章。特别是（使我很高兴的是），他发现在用 Python 实现 Simula-67 样式模拟时，这些技术很有用。

结果表明 Tony Vignaux 和 Chang Chui 以前曾创建了另一个 Python 库，它在概念上更接近于 Simscript，而且该库使用了标准线程技术，而不是我的半协同例程技术。该小组在一起研究时，认为基于生成器的样式更有效得多，并且最近在 SourceForge 上发起了使用 GPL 的项目，称为 SimPy（请参阅 [参考资料](https://simpy.readthedocs.io/en/latest/contents.html)，获得 SimPy 主页的链接），目前处于 beta 测试版状态。Vignaux 教授希望他在惠灵顿维多利亚大学（University of Victoria）的将来大学教学中使用统一的 SimPy 包；我相信该库也非常适合应用到各类实用问题中。

在本专栏文章中，我将一直使用食品杂货店内具有多条通道的付款区域这个相当简单的示例。通过使用所演示的模拟，我们可以根据对扫描器技术、购物者习惯、人员配备需求等进行的各种更改所产生的经济上和等待时间上的含义提出问题。这个建模的优点是在您对所做的更改产生的含义有清晰的想法时，它让您能提前制定策略。很明显，大多数读者不会专门经营一家食品杂货店，但这些技术可以广泛地应用于各类系统中。

> 随机的定义
> 与“连接”相类似，它是那些 最适合形容其作业的词汇之一 - 再也找不到更适合的了：
> **随机（stochastic）**，源自希腊语 stokhastikos（形容词） 
> 1. 推测的、与推测相关的或者具有推测特点的；好推测的。 
> 2. 在统计学上：涉及或包含一个随机变量或多个随机变量，或涉及偶然性或概率。


## 模拟的概念
SimPy 库只提供了三个抽象／父类，并且它们对应于模拟的三个基本概念。有许多其它常规函数和常量用于控制模拟的运行，但重要的概念都与这些类结合在一起。

模拟中的核心概念是 _进程_。一个进程只是一个对象，它完成某些任务，随后在它准备完成下一个任务之前有时会等待一会儿。在 SimPy 中，您还可以“钝化”进程，这意味着在一个进程完成一个任务后，只有当其它进程要求该进程完成其它任务时，它才会去做。把进程当作尝试完成一个目标，常常是很有用的。在编写进程时，通常把它编写成可以在其中执行多个操作的循环。在每个操作之间，可以插入 Python“yield”语句，它让模拟调度程序在返回控制之前执行每个等待进程的操作。

进程执行的许多操作取决于 _资源_ 的使用。资源只是在可用性方面受到限制。在生物学模型中，资源可能是食物供应；在网络模型中，资源可以是路由器或有限带宽通道；在我们的市场模拟中，资源是付款通道。资源执行的唯一任务是在任何给定的时间内将它的使用限于一个特定的进程上。在 SimPy 编程模型下，进程单独决定它要保留资源的时间有多长，资源本身是被动的。在实际系统中，SimPy 模型可能适合概念性方案，也可能不适合；很容易想象到资源在本质上会限制其利用率（例如，如果服务器计算机在必需的时间帧内没有获得满意的响应，则它会中断连接）。但作为编程问题，进程或资源是否是“主动”方就不是特别重要（只要确保您理解了您的意图）。

最后一个 SimPy 类是 _监控程序_。实际上监控程序不是很重要，只不过它很方便。监控程序所做的全部任务就是记录向它报告的事件，并保存有关这些事件的统计信息（平均值、计数、方差等）。该库提供的 Monitor 类对记录模拟措施是个有用的工具，但您也可以通过您想使用的其它任何技术来记录事件。事实上，我的示例使 Monitor 子类化，以提供某些（稍微）增强的能力。

## 设置商店：对模拟编程
在我所撰写的大部分文章中，我都会马上给出样本应用程序，但在本例中，我认为带您经历食品杂货店应用程序的每个步骤会更有用。如果您愿意的话，可以把每个部分剪贴在一起；SimPy 创造者们将在将来的发行版中包含我的示例。
SimPy 模拟中的第一步是几个常规的导入（import）语句：

清单 1. 导入 SimPy 库
```python
#!/usr/bin/env python
from __future__ import generators
from SimPy import Simulation
from SimPy.Simulation import hold, request, release, now
from SimPy.Monitor import Monitor
import random
from math import sqrt
```

有些 SimPy 附带的示例使用 import * 样式，但我更喜欢使我填充的名称空间更清晰。对于 Python 2.2（SimPy 所需的最低版本），将需要如指出的那样，导入生成器特性。对于 Python 2.3 以后的版本，不需要这样做。
对于我的应用程序，我定义了几个运行时常量，它们描述了在特定的模拟运行期间我感兴趣的几个方案。在我更改方案时，我必须在主脚本内编辑这些常量。要是这个应用程序的内容更充实，那么我就可能用命令行选项、环境变量或配置文件来配置这些参数。但就目前而言，这个样式已经足够了：

清单 2. 配置模拟参数
```python
AISLES = 5         # Number of open aisles
ITEMTIME = 0.1     # Time to ring up one item
AVGITEMS = 20      # Average number of items purchased
CLOSING = 60*12    # Minutes from store open to store close
AVGCUST = 1500     # Average number of daily customers
RUNS = 10          # Number of times to run the simulation
```

我们的模拟需要完成的主要任务是定义一个或多个进程。对于模拟食品杂货店，我们感兴趣的进程是在通道处付款的顾客。

清单 3. 定义顾客的操作
```python
class Customer(Simulation.Process):
    def __init__(self):
        Simulation.Process.__init__(self)
        # Randomly pick how many items this customer is buying
        self.items = 1 + int(random.expovariate(1.0/AVGITEMS))
    def checkout(self):
        start = now()           # Customer decides to check out
        yield request, self, checkout_aisle
        at_checkout = now()     # Customer gets to front of line
        waittime.tally(at_checkout-start)
        yield hold, self, self.items*ITEMTIME
        leaving = now()         # Customer completes purchase
        checkouttime.tally(leaving-at_checkout)
        yield release, self, checkout_aisle

```

每位顾客已经决定采购一定数量的商品。（我们的模拟不涉及从食品杂货店通道上选择商品；顾客只是推着他们的手推车到达付款处。）我不能确定这里的指数变量分布确实是一个精确的模型。在其低端处我感觉是对的，但我感到对实际购物者究竟采购了多少商品的最高极限有点失实。在任何情况下，您可以看到如果可以使用更好的模型信息，则调整我们的模拟是多么简单。

顾客采取的操作是我们所关注的。顾客的“执行方法”就是 .checkout() 。这个进程方法通常被命名为 .run() 或 .execute() ，但在我的示例中， .checkout() 似乎是最可描述的。您可以对它起任何您希望的名称。 Customer 对象所采取的实际 操作仅仅是检查几个点上的模拟时间，并将持续时间记录到 waittime 和 checkouttime 监控程序中。但在这些操作之间是至关重要的 yield 语句。在第一种情况中，顾客请求资源（付款通道）。只有当顾客获得了所需的资源之后，他们才能做其它操作。一旦来到付款通道，顾客实际上就在付款了 — 所花时间与所购商品的数量成比例。最后，经过付款处之后，顾客就释放资源，以便其他顾客可以使用它。

上述代码定义了 Customer 类的操作，但我们需要在运行模拟之前，创建一些实际的顾客对象。我们 可以为一天中将要购物的每位顾客生成顾客对象，并为每位顾客分配相应的付款时间。但更简洁的方法是“在每位顾客到商店时”，让工厂对象生成所需的顾客对象。实际上模拟并不会同时对一天内将要购物的所有顾客感兴趣，而是只对那些要同时争用付款通道的顾客感兴趣。注意： Customer_Factory 类本身是模拟的一部分 — 它是一个进程。尽管对于这个客户工厂，您可能联想到人造的机器工人（la Fritz Lang 的 Metropolis），但还是应该只把它看作编程的便利工具；它并不直接对应已建模域中的任何事物。

清单 4. 生成顾客流
```python
class Customer_Factory(Simulation.Process):
    def run(self):
        while 1:
            c = Customer()
            Simulation.activate(c, c.checkout())
            arrival = random.expovariate(float(AVGCUST)/CLOSING)
            yield hold, self, arrival
```

正如我前面提到的，我想收集一些当前 SimPy Monitor 类没有解决的统计信息。也就是，我并不仅仅对平均付款时间感兴趣，而且还对给定方案中最糟糕情况感兴趣。所以我创建了一个增强的监控程序，它收集最小和最大的计数值。

用监控程序监视模拟

```python
class Monitor2(Monitor):
    def __init__(self):
        Monitor.__init__(self)
        self.min, self.max = (int(2**31-1),0)
    def tally(self, x):
        Monitor.tally(self, x)
        self.min = min(self.min, x)
        self.max = max(self.max, x)
```

我们模拟的最后一步当然是 运行它。在大多数标准示例中，只运行一次模拟。但对于我的食品杂货店，我决定通过几次模拟进行循环，每次对应于某一天的业务。这看来是个好主意，因为有些统计信息会随每天的情况而有相当大的不同（因为到达的顾客人次以及所购商品数采用随机产生的不同值）。

清单 6. 每天运行模拟
```python
for run in range(RUNS):
    waittime = Monitor2()
    checkouttime = Monitor2()
    checkout_aisle = Simulation.Resource(AISLES)
    Simulation.initialize()
    cf = Customer_Factory()
    Simulation.activate(cf, cf.run(), 0.0)
    Simulation.simulate(until=CLOSING)
    #print "Customers:", checkouttime.count()
    print "Waiting time average: %.1f" % waittime.mean(), \
          "(std dev %.1f, maximum %.1f)" % (sqrt(waittime.var()),waittime.max)
    #print "Checkout time average: %1f" % checkouttime.mean(), \
    #      "(standard deviation %.1f)" % sqrt(checkouttime.var())
print 'AISLES:', AISLES, '  ITEM TIME:', ITEMTIME

```

## 三人不欢：一些结果（以及它们意味着什么）
当我最初考虑食品杂货店模型时，我认为模拟可以解答几个直接问题。例如，我想象店主可能会选择购买改进的扫描仪（减少 ITEMTIME ），或者选择雇佣更多职员（增加 AISLES ）。我想只要在每个方案下运行这个模拟（假设雇员和技术成本给定的情况下），并确定上面两种选择哪种更能减少成本。

只有运行了模拟后，我才意识到可能会出现比预料的更有趣的事情。查看收集的所有数据，我意识到我不知道要尝试优化的是什么。 什么。例如，减少 平均付款时间和减少 最差情况的时间，哪个更重要？哪些方面会提高总体顾客满意度？另外，如何比较顾客在付款之前所用的等待时间以及扫描所购商品所花的时间？以我个人的经验，我会在等待的队列中感到不耐烦，但在扫描我的商品时，我不会感到很麻烦（即使这会花一些时间）。

当然，我没有经营食品杂货店，所以我不知道所有这些问题的答案。但这个模拟确实让我准确地决定什么是折衷方案；而且它很简单，足以稍作调整就可适用于许多行为（包括那些还未显式地参数化的行为 — 例如，“一整天中顾客 真的会一直不断地来吗？”）。

我只要演示最后一个示例，就可以说明该模型的价值。我在上面曾写道复杂系统的行为难以概念化。我认为这里的示例可以证明这一事实。在可用的通道从 6 条减少到 5 条（其它参数不变）时，您认为会出现什么情况？最初我想会 稍微增加最糟糕情况下的付款时间。而事实并非如此：


清单 7. 通道数变化前后运行的两个样本
```python
% python Market.py
Waiting time average: 0.5 (std dev 0.9, maximum 4.5)
Waiting time average: 0.3 (std dev 0.6, maximum 3.7)
Waiting time average: 0.4 (std dev 0.8, maximum 5.6)
Waiting time average: 0.4 (std dev 0.8, maximum 5.2)
Waiting time average: 0.4 (std dev 0.8, maximum 5.8)
Waiting time average: 0.3 (std dev 0.6, maximum 5.2)
Waiting time average: 0.5 (std dev 1.1, maximum 5.2)
Waiting time average: 0.5 (std dev 1.0, maximum 5.4)
AISLES: 6   ITEM TIME: 0.1
% python Market.py
Waiting time average: 2.1 (std dev 2.3, maximum 9.5)
Waiting time average: 1.8 (std dev 2.3, maximum 10.9)
Waiting time average: 1.3 (std dev 1.7, maximum 7.3)
Waiting time average: 1.7 (std dev 2.1, maximum 9.5)
Waiting time average: 4.2 (std dev 5.6, maximum 21.3)
Waiting time average: 1.6 (std dev 2.6, maximum 12.0)
Waiting time average: 1.3 (std dev 1.6, maximum 7.5)
Waiting time average: 1.5 (std dev 2.1, maximum 11.2)
AISLES: 5   ITEM TIME: 0.1
```

减少一条付款通道不是使平均等待时间增加 1/5 或类似的情况，而是使它增加了大约 4 倍。而且，最不幸的顾客（在这些特定的运行期间）的等待时间从 6 分钟增加到了 21 分钟。如果我是经理，我认为了解这个极限情况对顾客满意度而言是极其重要的。谁会早已知道这一点呢？