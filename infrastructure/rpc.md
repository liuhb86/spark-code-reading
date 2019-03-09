#Spark RPC 

Major classes: **RpcEnv**, **RpcEndpoint**, **RpcEndpointRef**)

The RPC (remote procedure call) is used in Spark for remote communications. It’s goal is to replace Akka used in Spark. Thus, the design heavily inspired by Akka. (See [SPARK-5124](https://jira.apache.org/jira/browse/SPARK-5124), [Spark-5293](https://issues.apache.org/jira/browse/SPARK-5293)). You can think of it as a lite version of Akka actor system.
This article [In Depth Analysis of Spark RPC( in Chinese)](https://zhuanlan.zhihu.com/p/28893155) explained Spark RPC quite well and was endorsed by the main designer Shixiong Zhu. The article was based on Spark 2.1, so the details may be different. But it doesn’t affect the main idea. 