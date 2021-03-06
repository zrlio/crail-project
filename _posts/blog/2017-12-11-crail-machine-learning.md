---
layout: post
title: "Machine Learning Accelerating with Crail [DRAFT]"
author: Michael Kaufmann
category: blog
comments: true
---

In this blog post we're looking at a machine learning application on top of Apache Spark, its I/O behavior and how the Crail whole-stack optimization approach can help to accelerate this and other similar workloads.

When we think of how to accelerate machine learning applications, we mostly think of GPU-based hardware acceleration. What we often forget though, is that the time spent in the I/O path (data copying, (de-)serialization, network transfers) can also account for a large fraction of the total runtime for distributed machine learning applications. To demonstrate this, we analyzed the I/O behavior of ???, a Spark-integrated, native machine learning library with optional GPU support.

### Machine Learning Primer
The term *machine learning* encompasses a large number of different approaches and algorithms to extract knowledge from (large amounts of) known data and to use that knowledge to infer (predict) certain properties from unknown data.

TBD
#### Support Vector Machines
TBD

#### CoCoA
TBD

### Hardware Configuration
* Cluster
  * 8 compute + 1 management node
* Node configuration
  * Model: IBM Power Systems S822LC (PowerNV 8335-GTB)
  * CPU: POWER8NVL @ 4 GHz, 160 hardware threads
  * DRAM: 512 GB DDR4 @ 1600 MHz
  * GPU: 4x NVIDIA Tesla P100 (not used in this benchmark)
  * Network: 1x 100 GBit/s Mellanox ConnectX-5 Infiniband
* Software
  * Red Hat Enterprise Linux Server release 7.4 (Maipo)
  * Crail 1.0
  * ???
  * Spark 2.1.0 (modified)

### Test Setup
For our tests we mainly used the sparse KDDA [https://pslcdatashop.web.cmu.edu/KDDCup/downloads.jsp] dataset with 20M features and 8.4M samples. The data itself was read from a shared NFS volume due to limitations of the underlying ??? library at the time of testing. This, however, has no impact on the test results since ??? is used in both cases.

All tests were run on 8 physical nodes with 16 single-core executors in total. Both configurations (vanilla/crail) was run 10 times with 50 iterations of the CoCoA algorithm each.

### I/O Behavior Analysis
Crail can accelerate various I/O tasks of Spark Applications, namely shuffle, broadcast and the  transfer of task results. For the CoCoA [https://arxiv.org/pdf/1409.1458.pdf] algorithm that ??? is implementing, shuffle is not relevant since the training data is transferred only once (and as mentioned earlier, this is done via NFS), in the beginning of the test and remains on the worker nodes for the entire duration of the training. However, the model generated by the local solver has to be reduced at the end of every iteration (using a `reduce` or `treeReduce` in Spark) and transmitted back to each worker before the next iteration can start. This is done using broadcasts.

The next figure shows a high level view of CoCoA's communication pattern with 2 workers and 1 driver process. Blue arrows represent broadcast communication whereas green arrows represent a reduce operation.

<div style="text-align:center"><img src ="/img/blog/crail-machine-learning/cocoa.svg" width="250"/></div>

The first step in accelerating this workload was to analyze where time is being spent. An initial breakdown resulted in this

```
Chart of time-spent (per iteration) breakdown
* broadcast 386  = 18.91%
* reduce    1185 = 58.06%
* compute   470  = 23.03%
```

As we can see from the analysis, almost 77% of the time is spent in the I/O path. This is not least due to the highly optimized compute part of the ??? library. However, to accelerate this workload further, we have to optimize the I/O path.

### Task Result I/O
Task result serialization is a critical operation between two stages and makes up the majority of the overall runtime. During each iteration, local solvers update the model. These updates have to be combined using a simple reduce (or tree reduce) operation in order to improve the global model. Given a size of approximately 100MB for our model (other datasets may result in smaller or larger models), this is a time consuming process during which no learning can occur.

Task result (de-)serialization is especially important when we use tree reduce. While tree reduce lowers the CPU load on the driver for the actual reduce function (vector addition in our case) by distributing the load across multiple executors, it adds some overhead because data has to be (de-)serialized once per node in the reduce tree.

```
Figure showing difference between reduce, tree reduce.
```

Our analysis showed that task results are serialized twice (which also incurrs buffer allocation and data copying) and copied once more into the block store.

#### First Serialization
Here, the original task result is serialized for the first time.

Executor.scala:386
```scala
        val valueBytes = resultSer.serialize(value)
```

#### Second Serialization
Then, the serialized results data is encapsulated in a DirectTaskResult object and serialized again.

Executor.scala:405
```scala
        val directResult = new DirectTaskResult(valueBytes, accumUpdates)
        val serializedDirectResult = ser.serialize(directResult)
        val resultSize = serializedDirectResult.limit
```

#### Final Copying
Finally, Spark has to determine whether the serialized data is small enough to be sent via an RPC or whether it has to store it in the block manager (if resultSize > maxDirectResultSize). Since we deal with a fairly large amount of data, Spark always puts it in the block manager and only sends a reference via RPC.

Executor.scala:410
```scala
        val serializedResult: ByteBuffer = {
          if (maxResultSize > 0 && resultSize > maxResultSize) {
            logWarning(s"Finished $taskName (TID $taskId). Result is larger than maxResultSize " +
              s"(${Utils.bytesToString(resultSize)} > ${Utils.bytesToString(maxResultSize)}), " +
              s"dropping it.")
            ser.serialize(new IndirectTaskResult[Any](TaskResultBlockId(taskId), resultSize))
          } else if (resultSize > maxDirectResultSize) {
            val blockId = TaskResultBlockId(taskId)
            env.blockManager.putBytes(
              blockId,
              new ChunkedByteBuffer(serializedDirectResult.duplicate()),
              StorageLevel.MEMORY_AND_DISK_SER)
            logInfo(
              s"Finished $taskName (TID $taskId). $resultSize bytes result sent via BlockManager)")
            ser.serialize(new IndirectTaskResult[Any](blockId, resultSize))
          } else {
            logInfo(s"Finished $taskName (TID $taskId). $resultSize bytes result sent to driver")
            serializedDirectResult
          }
        }
```

### Cutting through the stack with Crail
With Crail, we can avoid the double-serialization and almost all data copying by providing zero-copy buffers and the possibility to provide a custom serializer for our application. While this poses additional work for the application developer, it can significantly reduce the time spent in task result reduction.

```scala
Show same code using Crail.
```

#### I/O Path Optimization
Accelerating the broadcast operation is fairly simple since Crail provides a plugin for Spark which
reduced the time spent in broadcast I/O from 386 ms to just 83 ms (4.65x faster).

The task result reduce path proved to be a greater challenge, though. Spark provides an easy way to
replace its native block transfer method, however this had little to no effect. Further analysis
showed that the main bottleneck was not the I/O transfer itself but the way the data was prepared 
for transmission. Before the task result data even got to the block transfer manager, it was copied
multiple times and serialized twice in the Executor (TODO: Doublecheck those numbers) and similarly
deserialized and copied multiple times for every reduce step. Using Crail, it was relatively easy to
modify this code for zero-copy and single serialization. This reduced the time spent in the reduce
path from 1185 ms to 680 ms (1.74x faster).

### Putting it all together
When using all optimizations we were able to reduce the runtime of the ??? machine learning
application from ~2.5s to 1.4s per iteration (1.77x faster).

<div style="text-align:center"><img src ="/img/blog/crail-machine-learning/final.svg" width="550"/></div>
