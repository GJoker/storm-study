Storm引入DRPC主要是利用storm的实时计算能力来并行化CPU密集型（CPU intensive）的计算任务。DRPC的storm topology以函数的参数流作为输入，而把这些函数调用的返回值作为topology的输出流。
####概览
Distribute RPC是由一个"DRPC服务器"协调（storm自带了一个实现）。DRPC服务器协调：1）、接收一个RPC请求。2）、发送请求到storm topology。3）、从storm topology接收结果。4）、把结果发回给等待的客户端。从客户端的角度来看一个DRPC调用跟一个普通的RPC调用没有任何区别。

**DRPC大致的工作流程：**
![这里写图片描述](http://img.blog.csdn.net/20151021124839979)

客户端给Drpc服务器发送要执行的函数（function）的名字，以及这个函数的参数。实现了这个函数的topology使用DRPCSpout从DRPC服务器接收函数调用流，每个函数调用被DRPC服务器标记了一个唯一的id。这个topology然后计算结果，在topology的最后，一个叫做ReturnResults的bolt会连接到DRPC服务器，并且把这个调用的的结果发送给DRPC服务器（通过那个唯一的id标识）。DRPC服务器用那个唯一id来跟等待的客户端匹配上，唤醒这个客户端并且把结果发送给它。

**LinearDRPCTopologyBuilder的工作原理**

 1. DRPCSpout发射tuple：[args，return-info]。return-info包含DRPC服务器的主机地址、端口号以及当前请求的request-id（DRPC服务器生成）
 
 2. Drpc Topology包含以下元素：
	- DRPCSpout
    - PrepareRequest（生成request-id,return info以及args）
    - CoordinatedBolt 
    - JoinResult—通过return info组合结果。
    - ReturnResult—连接到DRPC服务器并且返回结果。
 3. LinearDRPCTopologyBuilder是利用storm的原语来构建高级抽象的很好例子。
 
    
  
