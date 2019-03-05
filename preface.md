# Preface 

This document will go through the source code implementation of important workflows of Apache Spark(mainly SparkCore and SparkSQL). It assumes you have known how to use Spark quite well and understood the key concepts such as DataFrame, RDD, Transformation, Action, Job, Stage, Task, Driver, Executor, etc. If you are not, please read the [official documentation](https://spark.apache.org/docs/latest/index.html) and other books, and do exercises. 

This document doesn’t explain everything in detail. The best way is read along with the Spark source code. 

The Code analysis is based on commit [f85ed9a3e55083b0de0e20a37775efa92d248a4f](https://github.com/apache/spark/tree/f85ed9a3e55083b0de0e20a37775efa92d248a4f) (between Spark 2.4 and 2.5) 

## References: 
* https://spark.apache.org/docs/latest/index.html 
* https://jaceklaskowski.gitbooks.io/mastering-apache-spark/ 