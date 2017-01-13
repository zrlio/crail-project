---
layout: default
title: I/O Challenges
---

<div style="text-align: justify"> 
<p>
Modern storage and network technologies such as 100Gb/s Ethernet, RDMA, NVMe flash, etc., present new opportunities for data processing systems to further reduce the response times of analytics queries on large data sets. Unfortunately, leveraging modern hardware in systems like Spark, Flink or Hadoop remains challenging, for multiple reasons:
</p>
</div>

* Performance: today's data processing stacks employ many software layers, which is key to making the stacks modular and flexible to use. But the software layers also impose overheads during I/O operations that prevent applications from enjoying the full potential of the high-performance hardware. To eliminate these overheads, I/O operations must interact with the hardware directly from within the application context using principles like RDMA, DPDK or SPDK.

* New opportunities: the improved performance of modern networking and storage hardware opens door to rethink the interplay of I/O and compute in a distributed data processing system. For instance, low latency remote data access enables schedulers to relax on data locality and in turn make better use of compute resources. At one extreme, storage resources can be completely disaggregated which is more cost effective and simplifies maintenance.

* Heterogeneity: with modern hardware I/O has become more complex. Not only are there more options to store data (disk, flash, DRAM, disaggregated storage, etc.) but also it is getting increasingly important to use storage effectively. For  instance, some newer technologies such as PCM permit data access at the byte granularity. Mediating storage access through a block device interface is a bad fit in such a case. Then, with accelerators like GPUs or FPGAs extending the traditional compute layer, access to accelearator-local memory needs to be re-thought as well.

In the [Blog](http://crail.io/blog) section we discuss each of those challenges in more detail.

## Crail Architecture

<div style="text-align: justify"> 
<p>
Crail aims at providing a comprehensive solution to the above challenges in a form that is non-intrusive and compatible with the Apache data processing ecosystem. In particular, Crail is designed to be consumeable by different compute engines such as Spark, Flink, Solr, etc, with very little integration effort. 
</p>
</div>

<h3 id="overview">Overview</h3>

<div style="text-align: justify">
<p>
The backbone of the Crail I/O architecture is the Crail Distributed File System (CrailFS), a high-performance multi-tiered data store for temporary data in analytics workloads. Data processing frameworks and workloads may directly interact with CrailFS for fast storage of in-flight data, but more commonly the interaction takes place through one of the Crail modules. As an example, the CrailHDFS adapter provides a standard HDFS interface allowing applications to use CrailFS the same way they use regular HDFS. Applications may want to use CrailHDFS for short-lived performance critical data, and regular HDFS for long-term data storage. SparkCrail is a Spark specific module implementing various I/O operations such as shuffle, broadcast, etc. Both CrailHDFS and SparkCrail can be used transparently with no need to recompile either the application or the data processing framework. 
</p>
</div>
<br>
<img src="architecture.png" width="500" align="middle">
<br><br>
<div style="text-align: justify">
<p>
Crail modules are thin layers on top of CrailFS and implementing new modules for a particular data processing framework or a specific I/O operation requires only a moderate amount of work. At the same time, modules inherit the full benefits of CrailFS in terms of user-level I/O, performance and storage tiering. In the Blog section we show that Spark2Crail permits all-to-all data shuffling very close to the speed of the 100Gb/s network fabric. 
</p>
</div>

<h3 id="fs">Crail Distributed File System</h3>

<div style="text-align: justify">
<p>
CrailFS implements a file system namespace across a cluster of RDMA interconnected storage resources such as DRAM or flash. 
Storage resources may be co-located with the compute nodes of the cluster, or disagreggated inside the data center, or a mix of both. Files in the Crail namespace consist of arrays of blocks distributed across storage resources in the cluster. Crail groups storage resources into different tiers (e.g, disk, flash, DRAM) and permits files to be created in specific tiers but also across tiers. For instance, by default Crail uses horizontal tiering where higher performing storage resources are filled up across the cluster prior to using lower performing tiers -- resulting in a more effective usage of storage hardware.
</p>
</div>

<br>
<img src="filesystem.png" width="700" align="middle">
<br><br>

<div style="text-align: justify">
<p>
Access to storage resources over the network -- as happening during file read/write operations -- are implemented using RDMA. For instance, accesses to blocks residing in the DRAM tier are implemented using one-sided read/write RDMA operations. With one-sided operations the storage nodes remain completely passive, thus, not are not wasting any CPU cycles for I/O. At the same time, the client benefits from zero-copy data placements, freeing CPU cycles that would otherwise be used for memory copying, context switching etc. One-sided operations are also very effective in reading or writing subranges of a storage block as they only ship over the network the actual data that is read or written, instead of shipping the entire block. 
</p>
<p>
In Crail, storage tiers are actual plugins. A storage tier defines the type of storage and network hardware it supports. For instance, the disaggregated flash tier supports shared flash storage accessed through a iSER (iSCSI over RDMA), versus, the upcoming NVMef storage will be supporting NVMe flash access over RDMA fabrics. 
</p>
</div>
<br>
<img src="tiering.png" width="650" align="middle">
<br><br>
<div style="text-align: justify">
<p>
Files in Crail are append-and-overwrite with a single-writer per file at a given time. File write ownership is granted in the form of leases that expire if not used. Generally, all the read/write operations are asynchronous, facilitating interleaving of computation and networking during data processing. Crail also exports functions to allocate dedicated I/O buffers from a resuseable pool. Aside from the standard file system operations, Crail provides extra semantics providing detailed control as to which storage tier and location preference should be used when allocating storage resources for files. A simple example of a Crail write operation is shown below:
</p>
</div>
    CrailConfiguration conf = new CrailConfiguration();
    CrailFS fs = CrailFS.newInstance(conf);
    CrailFile file = fs.createFile(filename, 0, 0).get().syncDir();
    CrailOutputStream outstream = file.getDirectOutputStream();
    ByteBuffer dataBuf = fs.allocateBuffer();
    Future<DataResult> future = outputStream.write(dataBuf);
    ...
    future.get();
<div style="text-align: justify">
<p>
Crail not only exports a Java API but it is written entirely in Java, which makes it easy to use and allows for a better integration with data processing frameworks like Spark, Flink, Hadoop, etc. Crail is based on <a href="https://github.com/zrlio/disni">DiSNI</a>, a user-level network and storage stack for the Java virtual machine. 
</p>
</div>

<h3>Crail HDFS Adapter</h3>

The Crail HDFS adapter has two advantages. First, administrators can interact with Crail using the standard HDFS shell:

    ./bin/crail fs -mkdir /test
    ./bin/crail fs -ls /
    ./bin/crail fs -copyFromLocal <path-to-local-file> 
    ./bin/crail fs -cat /test/<file-name>

Second, regular HDFS-based application will transparently work on Crail when using fully qualified path names:

    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(conf);
    fs.create("crail://test/hello.txt");
    
 <h3 id="spark">SparkCrail Module</h3>   

<div style="text-align: justify">
<p>
The SparkCrail module includes a Crail based shuffle engine as well as a broadcast implementation. The shuffle engine maps rey ranges to directories in CrailFS. Each map task, while partitioning the data, appends key/value pairs to individual files in the corresponding directories. Tasks running on the same core within the cluster append to the same files, which reduces storage fragmentation. 
</p>
</div>

<br>
<img src="shuffle.png" width="550" align="middle">
<br><br>

<div style="text-align: justify">
<p>
As with the Crail HDFS adaptor, the shuffle engine benefits from the performance and tiering benefits of the Crail file system. For instance, individual shuffle files are served using horizontal tiering. In most cases that means the files are growing into the memory tier as long as there is some DRAM available in the cluster, after which they extend to the flash tier. The shuffle engine further uses the Crail location affinity API to make sure local DRAM and flash is preferred over remote DRAM and flash respectively. Note that the shuffle engine is also completely zero-copy, as it transfers data directly from the I/O memory of the mappers and to the I/O memory of the reducers. 
</p>
</div>

<div style="text-align: justify">
<p>
The Crail-based Broadcast broadcast plugin for Spark stores broadcast variables in Crail files. In contrast to shuffle engine, broadcast is implemented without location affinity, which makes sure the underlying blocks of the Crail files are distributed across the cluster, leading to a better load balancing when reading broadcast variables. Crail Shuffle and broadcast components can be enabled in Spark by setting the following system properties in spark-defaults.conf:
</p>
</div>

    spark.shuffle.manager		org.apache.spark.shuffle.crail.CrailShuffleManager
    spark.broadcast.factory		org.apache.spark.broadcast.CrailBroadcastFactory

<div style="text-align: justify">
<p>
Both, broadcast and shuffle require Spark data objects to be serialized into byte streams (that is case also for the default Spark broadcast and shuffle components). Even though both Crail components work fine with any of the Spark built-in serializers (e.g. Kryo), to achieve the best possible performance applications running on Crail are encouraged to provide serialization and deserialization methods for their data types explicitly. One reason for this is that the built-in Spark serializers assume byte streams of type java.io.(InputStream/OutputStream). These stream types are less powerful than Crail streams. For instance, streams of type InputStream/OutputStream export a synchronous API and are restricted to on-heap memory. Crail streams, on the other hand, export an asynchronous API and integrate well with off-heap memory to reduce data copies. By defining custom serialization/deserialization methods, applications can take full advantage of Crail during broadcast and shuffle operations. Moreover, serializers dedicated to one particular application type may further exploit information about the specific application data types to achieve a better performance. For instance, a custom serializer for a sorting application running on key/value objects of a fixed length byte array will not need to store serialization meta data, which reduces the final data size and simplifies the serialization process. 
</p>
</div>

<br>
<img src="serializer.png" width="490" align="middle">
<br><br>

<div style="text-align: justify">
<p>
Serialization is one important aspect for broadcast and shuffle operations, sorting another, even though specific to shuffling. Sorting directly follows the network fetch phase in a shuffle operation if a key ordering is given. Again, the Crail shuffle engine works fine with the Spark built-in sorter, but often the shuffle performance can be improved by an application specific sorter. For instance, an application may use the Crail GPU tier to store data. In that case, sorting can be pushed to the GPU, rather fetching the data into main memory and sorting it on the CPU. In other cases, the application may know the data types in advance and uses it to simplify sorting (e.g. use Radix sort instead TimSort).
</p>
<p>
Application specific serializers and sorter can be defined by setting the following system properties in spark-defaults.conf:
</p>
</div>

    spark.crail.shuffle.serializer  
    spark.crail.shuffle.sorter	
    






