---
layout:     post
title:      "MapReduce"
subtitle:   " \"好好学习，天天向上\""
date:       2019-08-25 18:23:00
author:     "WQ"
header-img: "img/blogImg/2019-08-25/th.jpeg"
catalog: true
tags:
    - Hadoop
---

# MapReduce

![reduce-fetch-merge](/img/blogImg/hadoop/hadoop-shuffle-sort.jpg)

分析过程中忽略和yarn相关的交互。

## MapTask

贯穿整个mapper计算阶段的Mapper.Context，用于封装信息，需要留意下其继承体系中的TaskInputOutputContext。

对于mapper的个数问题，在Mapper类的注释中解释为`The Hadoop Map-Reduce framework spawns one map task for each InputSplit generated by the InputFormat for the job.` 该类的注释很重要。

### Mapper的输入涉及的接口

#### InputFormat

1. Validate the input-specification of the job.
2. ***Split-up the input file(s) into logical InputSplits, each of which is then assigned to an individual Mapper.***
3. Provide the RecordReader implementation to be used to glean input records from the logical InputSplit for processing by the Mapper.

#### InputSplit

InputSplit represents the data to be processed by an individual Mapper.Typically, it presents a byte-oriented view on the input and is the responsibility of RecordReader of the job to process this and present a record-oriented view.

#### RecordReader

The record reader breaks the data into key/value pairs for input to the Mapper.

上面三者的关系在InputFormat#createRecordReader可以看出。

典型实现FileInputFormat

```java
  public List<InputSplit> getSplits(JobContext job) throws IOException {
    StopWatch sw = new StopWatch().start();
    long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
    long maxSize = getMaxSplitSize(job);

    // generate splits
    List<InputSplit> splits = new ArrayList<InputSplit>();
    List<FileStatus> files = listStatus(job);

    //是否忽略子目录
    boolean ignoreDirs = !getInputDirRecursive(job)
      && job.getConfiguration().getBoolean(INPUT_DIR_NONRECURSIVE_IGNORE_SUBDIRS, false);
    for (FileStatus file: files) {
      if (ignoreDirs && file.isDirectory()) {
        continue;
      }
      Path path = file.getPath();
      long length = file.getLen();
      if (length != 0) {
        BlockLocation[] blkLocations;
        if (file instanceof LocatedFileStatus) {
          blkLocations = ((LocatedFileStatus) file).getBlockLocations();
        } else {
          FileSystem fs = path.getFileSystem(job.getConfiguration());
          blkLocations = fs.getFileBlockLocations(file, 0, length);
        }
        //正常情况下时可拆分的，某些压缩文件不行
        if (isSplitable(job, path)) {
          long blockSize = file.getBlockSize();
          long splitSize = computeSplitSize(blockSize, minSize, maxSize);

          long bytesRemaining = length;
          while (((double) bytesRemaining)/splitSize > SPLIT_SLOP) {
            int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
            //将文件拆分成多个InputSplit==>实质是对一些参数的封装，典型可以看DBInputSplit，只有start和end
            splits.add(makeSplit(path, length-bytesRemaining, splitSize,
                        blkLocations[blkIndex].getHosts(),
                        blkLocations[blkIndex].getCachedHosts()));
            bytesRemaining -= splitSize;
          }

          if (bytesRemaining != 0) {
            int blkIndex = getBlockIndex(blkLocations, length-bytesRemaining);
            splits.add(makeSplit(path, length-bytesRemaining, bytesRemaining,
                       blkLocations[blkIndex].getHosts(),
                       blkLocations[blkIndex].getCachedHosts()));
          }
        } else { // not splitable
          if (LOG.isDebugEnabled()) {
            // Log only if the file is big enough to be splitted
            if (length > Math.min(file.getBlockSize(), minSize)) {
              LOG.debug("File is not splittable so no parallelization "
                  + "is possible: " + file.getPath());
            }
          }
          splits.add(makeSplit(path, 0, length, blkLocations[0].getHosts(),
                      blkLocations[0].getCachedHosts()));
        }
      } else { 
        //Create empty hosts array for zero length files
        splits.add(makeSplit(path, 0, length, new String[0]));
      }
    }
    // Save the number of input files for metrics/loadgen
    job.getConfiguration().setLong(NUM_INPUT_FILES, files.size());
    sw.stop();
    if (LOG.isDebugEnabled()) {
      LOG.debug("Total # of splits generated by getSplits: " + splits.size()
          + ", TimeTaken: " + sw.now(TimeUnit.MILLISECONDS));
    }
    return splits;
  }
//对于RecordReader典型的TextInputFormat，其LineRecordReader，将每一行数据作为一个mapper阶段的value
```

总结

InputFormat是对InputSplit和RecordReader的封装，表示如何拆分和读取二进制流数据，RecordReader就是将流数据转换成key-value。

### Mapper的计算过程

计算过程很简单，就是循环判断是否还有nextKeyValue，有则不断的调用mapper逻辑。输出是用context中write方法，实质是效用OutputFormat中的write。

在计算完成之后，需要注意MapTask调用的done方法，committer.commitTask将临时目录中的文件移动到正式目录中去。

### Mapper的输出

#### OutputFormat

1. Validate the output-specification of the job. For e.g. check that the output directory doesn't already exist.
2. Provide the RecordWriter implementation to be used to write out the output files of the job. Output files are stored in a FileSystem.

#### OutputCommitter

1. Setup the job during initialization. For example, create the temporary output directory for the job during the initialization of the job.
2. Cleanup the job after the job completion. For example, remove the temporary output directory after the job completion.
3. **Setup the task temporary output.**
4. Check whether a task needs a commit. This is to avoid the commit procedure if a task does not need commit.
5. **Commit of the task output.**
6. Discard the task commit.

由于每个task都是用一个独立的临时输出文件（但目录结构不一样），所以在移动到正式目录时如果出现相同的文件名将会覆盖，所以在写OutputFormat时需要注意文件名的命名不要重复，可以学习下FileOutputFormat中命名方式。

典型FileOutputCommitter

job临时目录：outPath+_temporary+appAttemptId

每个task的临时目录是：在job的临时目录+_temporary+taskAttemptID

注意getDefaultWorkFile()就是使用的这种临时目录

#### RecordWriter

RecordWriter writes the output <key, value> pairs to an output file.

### 其他在MapTask中重要的类和方法

如果没有NumReduceTasks为0，直接使用NewDirectOutputCollector作为map阶段的输出，忽略reducer的逻辑，否则则使用NewOutputCollector。

#### NewDirectOutputCollector

没什么可说的，直接使用OutputFormat中创建的RecordWriter写数据。

#### NewOutputCollector

在有reduce阶段时使用的，在用户调用context.write时先写到org.apache.hadoop.mapred.MapOutputCollector当中，是其使用MapOutputBuffer作为数据缓存，整个类占据了MapTask的一半的篇幅。

##### MapOutputBuffer

MapOutputBuffer中使用外部排序【快排(默认)和堆排】，在flush阶段会进行临时文件的合并，即本地合并，可以看下Merger的逻辑。

###### SpillThread

单独的线程，当缓存数据超过最大限制需要将其写入到临时文件中（通过锁进行线程同步），默认的文件名为out/spil+spilnumber+.out。在输出到文件之前需要进行排序，对key-value进行排序，设置OutputKeyComparator也可以。如果没有定义combiner的话就直接输出了，否则还得调用combine方法，然后通过combine方法调用的输出才是真的输出值。如果需要执行combiner的话，那么combiner个数和MapTask个数一样，但是其输出是由分区觉得的【也决定了输出文件个数】

###### CombinerRunner

逻辑和执行reduce基本一致。

## ReduceTask

### Reduce的输入涉及的接口

#### ShuffleConsumerPlugin

其实现类为org.apache.hadoop.mapreduce.task.reduce.Shuffle，主要作用是将分散的map输出数据汇聚到相应的reduce节点中。

##### EventFetcher

作为一个独立的线程，通过yarn的api循环获取已经完成的map，并获取其事件交给ShuffleScheduler处理。

##### ShuffleScheduler

作为shuffle相关的调度中心

##### Fetcher

从ShuffleScheduler中获取可以copy的map输出ip和路径，这里的路径的最小单位是mapTask的输出路径（org.apache.hadoop.mapreduce.task.reduce.ShuffleSchedulerImpl#getMapsForHost），获取到的数据交给MergeManager进行合并。

##### MergeManager

对fetcher获取到的数据进行合并。

有两种处理类InMemoryMapOutput和OnDiskMapOutput，这两个类只是用于存储fetch到的数据。在MergeManager类的构造方法中会创建merger（独立线程），用于文件合并。

在使用memMerger时，如果有定义combiner，也会在这里处理合并过程中的数据，在merge的结束阶段会进行finalMerge，即将所有文件合并到一个当中，合并完成之后获得一个keyValueIterator，整个迭代器就是reduce中需要使用到的。

### ReduceContextImpl

这个类贯穿reduce计算阶段的，这里有一个重要的配置参数，即在job中设置的setOutputValueGroupingComparator，如果没有这个值就默认是用key的比较器，如果还没有那么key必须继承WritableComparable接口。

这个类的作用在org.apache.hadoop.mapreduce.task.ReduceContextImpl#nextKeyValue中，当我们在reduce中调用迭代器获取下一个value的时候，在该类的ValueIterator中可以看到，在不断的通过比较器判断是否时在同一组当中【前提是在shuffle阶段已经使用外部排序了】，注意比较的是key。