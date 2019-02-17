---
title: MR之自定义inputFormat
categories:
  - big-data
tags:
  - MapReduce
abbrlink: 987ce803
date: 2018-02-12 18:15:51
---

### 需求

无论hdfs还是mapreduce，对于小文件都有损效率，实践中，又难免面临处理大量小文件的场景，此时，就需要有相应解决方案

<br/>

### 分析

小文件的优化无非以下几种方式：

1、 在数据采集的时候，就将小文件或小批数据合成大文件再上传HDFS

2、 在业务处理之前，在HDFS上使用mapreduce程序对小文件进行合并

3、 在mapreduce处理时，可采用combineInputFormat提高效率

<br/>

## 实现

本节实现的是上述第二种方式

程序的核心机制：

自定义一个InputFormat

改写RecordReader，实现一次读取一个完整文件封装为KV

在输出时使用SequenceFileOutPutFormat输出合并文件

<br/>

### 代码

#### 自定义InputFromat

~~~java
public class WholeFileInputFormat extends
  FileInputFormat<NullWritable, BytesWritable> {
  //设置每个小文件不可分片,保证一个小文件生成一个key-value键值对
  @Override
  protected boolean isSplitable(JobContext context, Path file) {
    return false;
  }

  @Override
  public RecordReader<NullWritable, BytesWritable> createRecordReader(
    InputSplit split, TaskAttemptContext context) throws IOException,
  InterruptedException {
    WholeFileRecordReader reader = new WholeFileRecordReader();
    reader.initialize(split, context);
    return reader;
  }
}

~~~

<br/>

#### 自定义RecordReader

~~~~java
class WholeFileRecordReader extends RecordReader<NullWritable, BytesWritable> {
  private FileSplit fileSplit;
  private Configuration conf;
  private BytesWritable value = new BytesWritable();
  private boolean processed = false;

  @Override
  public void initialize(InputSplit split, TaskAttemptContext context)
    throws IOException, InterruptedException {
    this.fileSplit = (FileSplit) split;
    this.conf = context.getConfiguration();
  }

  @Override
  public boolean nextKeyValue() throws IOException, InterruptedException {
    if (!processed) {
      byte[] contents = new byte[(int) fileSplit.getLength()];
      Path file = fileSplit.getPath();
      FileSystem fs = file.getFileSystem(conf);
      FSDataInputStream in = null;
      try {
        in = fs.open(file);
        IOUtils.readFully(in, contents, 0, contents.length);
        value.set(contents, 0, contents.length);
      } finally {
        IOUtils.closeStream(in);
      }
      processed = true;
      return true;
    }
    return false;
  }

  @Override
  public NullWritable getCurrentKey() throws IOException,
  InterruptedException {
    return NullWritable.get();
  }

  @Override
  public BytesWritable getCurrentValue() throws IOException,
  InterruptedException {
    return value;
  }

  @Override
  public float getProgress() throws IOException {
    return processed ? 1.0f : 0.0f;
  }

  @Override
  public void close() throws IOException {
    // do nothing
  }
}

~~~~

<br/>

#### 定义mapreduce处理流程

~~~~java
public class SmallFilesToSequenceFileConverter extends Configured implements
  Tool {
  static class SequenceFileMapper extends
    Mapper<NullWritable, BytesWritable, Text, BytesWritable> {
    private Text filenameKey;

    @Override
    protected void setup(Context context) throws IOException,
    InterruptedException {
      InputSplit split = context.getInputSplit();
      Path path = ((FileSplit) split).getPath();
      filenameKey = new Text(path.toString());
    }

    @Override
    protected void map(NullWritable key, BytesWritable value,
                       Context context) throws IOException, InterruptedException {
      context.write(filenameKey, value);
    }
  }

  @Override
  public int run(String[] args) throws Exception {
    Configuration conf = new Configuration();
    System.setProperty("HADOOP_USER_NAME", "hdfs");
    String[] otherArgs = new GenericOptionsParser(conf, args)
      .getRemainingArgs();
    if (otherArgs.length != 2) {
      System.err.println("Usage: combinefiles <in> <out>");
      System.exit(2);
    }

    Job job = Job.getInstance(conf,"combine small files to sequencefile");
    //		job.setInputFormatClass(WholeFileInputFormat.class);
    job.setOutputFormatClass(SequenceFileOutputFormat.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(BytesWritable.class);
    job.setMapperClass(SequenceFileMapper.class);
    return job.waitForCompletion(true) ? 0 : 1;
  }

  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new SmallFilesToSequenceFileConverter(),
                                  args);
    System.exit(exitCode);

  }
}
~~~~