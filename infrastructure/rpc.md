# Spark RPC 

Major classes: **RpcEnv**, **RpcEndpoint**, **RpcEndpointRef**)

The RPC (remote procedure call) is used in Spark for remote communications. It’s goal is to replace Akka used in Spark. Thus, the design heavily inspired by Akka. (See [SPARK-5124](https://jira.apache.org/jira/browse/SPARK-5124), [Spark-5293](https://issues.apache.org/jira/browse/SPARK-5293)). You can think of it as a lite version of Akka actor system.

[neoReMinD](https://github.com/neoremind) extracted the Spark RPC(based on Spark 2.1) to a [standalone project](https://github.com/neoremind/kraps-rpc). [This section](https://github.com/neoremind/kraps-rpc#1-how-to-run) demostrates what Spark RPC does and how to use it.

The author also write an article [In Depth Analysis of Spark RPC](https://zhuanlan.zhihu.com/p/28893155) in Chinese. It explained Spark RPC quite well and was endorsed by the main designer Shixiong Zhu.

