# 什么是Actor

上一节[Actor系统](actor-systems.md) 解释了actor如何组成一个树形结构，actor是创建一个应用的最小单位。本节单独来看一个actor，解释在实现它时遇到的概念。

一个Actor是一个容器，它包含了状态，行为，邮箱，子Actor和一个监管策略。所有这些包含在一个Actor Reference里。

## Actor引用

一个actor对象需要与外界隔离开才能从actor模型中获益。因此，actor是以actor引用的形式展现给外界的，actor引用可以被自由的无限制地传来传去。内部对象和外部对象的这种划分使得所有想要的操作能够透明：重启actor不需要更新别处的引用、将实际actor对象放置到远程主机上、向另外一个应用程序发送消息。但最重要的方面是外界不可能从actor对象的内部获取它的状态，除非这个actor非常不明智地将信息公布出去。

## 状态

Actor对象通常包含一些变量来反映actor所处的可能状态。这可能是一个明确的状态机（如使用 FSM 模块)，或是一个计数器，一组监听器，待处理的请求等。这些数据使得actor有价值，并且必须将这些数据保护起来不被其它的actor所破坏。好消息是在概念上每个Akka actor都有它自己的轻量线程，这个线程是完全与系统其它部分隔离的。这意味着你不需要使用锁来进行资源同步，可以放心地并发性地来编写你的actor代码。

在幕后，Akka会在一组线程上运行一组Actor，通常是很多actor共享一个线程，对某一个actor的调用可能会在不同的线程上进行处理。Akka保证这个实现细节不影响处理actor状态的单线程性。

由于内部状态对于actor的操作是至关重要的，所以状态不一致是致命的。当actor失败并由其监管者重新启动，状态会进行重新创建，就象第一次创建这个actor一样。这是为了实现系统的“自愈合”。

可以视情况而定，一个actor的状态可以自动的恢复到重启之前的状态。这可以通过持久化接收的消息并在重启后替换它们而做到。

## 行为

每次当一个消息被处理时，消息会与actor的当前的行为进行匹配。行为是一个函数，它定义了处理当前消息所要采取的动作，假如客户已经授权过了，那么就对请求进行处理，否则拒绝请求。这个行为可能随着时间而改变，例如由于不同的客户在不同的时间获得授权，或是由于actor进入了“非服务”模式，之后又变回来。这些变化要么通过将它们放进从行为逻辑中读取的状态变量中实现，要么函数本身在运行时被替换出来，见`become`和`unbecome`操作。但是actor对象在创建时所定义的初始行为是专有的，所以当actor重启时会恢复这个初始行为

## 邮箱

Actor的用途是处理消息，这些消息是从其它的actor（或者从actor系统外部）发送过来的。连接发送者与接收者的纽带是actor的邮箱：每个actor有且仅有一个邮箱，所有的发来的消息都在邮箱里排队。排队按照发送操作的时间顺序来进行，这意味着从不同的actor发来的消息在运行时没有一个固定的顺序，这是由于actor分布在不同的线程中。从另一个角度讲，从同一个actor发送多个消息到相同的actor，则消息会按发送的顺序排队。

可以有不同的邮箱实现供选择，缺省的是FIFO：actor处理消息的顺序与消息入队列的顺序一致。这通常是一个好的选择，但是应用可能需要对某些消息进行优先处理。在这种情况下，可以使用优先邮箱来根据消息优先级将消息放在某个指定的位置，甚至可能是队列头，而不是队列末尾。如果使用这样的队列，消息的处理顺序是由队列的算法决定的，而不是FIFO。

Akka与其它actor模型实现的一个重要差别在于当前的行为必须处理下一个从队列中取出的消息，Akka不会去扫描邮箱来找到下一个匹配的消息。无法处理某个消息通常是作为失败情况进行处理，除非actor覆盖了这个行为。

## 子Actor

每个actor都是一个潜在的监管者：如果它创建了子actor来委托处理子任务，它会自动地监管它们。子actor列表维护在actor的上下文中，actor可以访问它。对列表的更改是通过创建`context.actorOf(...)`或者停止`context.stop(child)`子actor来实现，并且这些更改会立刻生效。实际的创建和停止操作在幕后以异步的方式完成，这样它们就不会“阻塞”其监管者。

## 监管策略

Actor的最后一部分是它用来处理其子actor错误状况的机制。错误处理是由Akka透明地进行处理的，将[监管与监控]中所描述的策略中的一个应用于每个出现的失败。由于策略是actor系统组织结构的基础，所以一旦actor被创建了它就不能被修改。

考虑到每个actor只有唯一的策略，这意味着如果一个actor的子actors应用了不同的策略，这些子actor应该按照相应的策略来进行分组，生成中间的监管者，这再一次地应用到了根据任务到子任务的划分方法来组织actor系统的结构。

## 当Actor终止时

一旦一个actor终止了，也就是，失败了并且不能用重启来解决，停止它自己或者被它的监管者停止，它会释放它的资源，将它邮箱中所有未处理的消息放进系统的“死信邮箱”。而actor引用中的邮箱将会被一个系统邮箱所替代，系统邮箱会将所有新的消息重定向到“死信邮箱”。 但是这些操作只是尽力而为，所以不能依赖它来实现“保证投递”。

不是简单地把消息扔掉的灵感来源于我们测试：我们在事件总线上注册了TestEventListener来接收死信，然后将每个收到的死信在日志中生成一条警告-这对于更快地解析测试失败非常有帮助。我们觉得这个功能也可能用于其它的目的。