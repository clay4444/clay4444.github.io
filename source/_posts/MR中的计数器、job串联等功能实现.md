---
title: MR中的计数器、job串联等功能实现
categories:
  - big-data
tags:
  - MapReduce
abbrlink: 9799c850
date: 2018-05-16 12:57:33
---



### 计数器应用

在实际生产代码中，常常需要将数据处理过程中遇到的不合规数据行进行全局计数，类似这种需求可以借助mapreduce框架中提供的全局计数器来实现

示例代码如下：

~~~~java
public class MultiOutputs {
    //通过枚举形式定义自定义计数器
    enum MyCounter{MALFORORMED,NORMAL}

    static class CommaMapper extends Mapper<LongWritable, Text, Text, LongWritable> {

        @Override
        protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

            String[] words = value.toString().split(",");

            for (String word : words) {
                context.write(new Text(word), new LongWritable(1));
            }
            //对枚举定义的自定义计数器加1
            context.getCounter(MyCounter.MALFORORMED).increment(1);
            //通过动态设置自定义计数器加1
            context.getCounter("counterGroupa", "countera").increment(1);
        }
    }
~~~~

<br/>

### 多job串联

一个稍复杂点的处理逻辑往往需要多个mapreduce程序串联处理，多job的串联可以借助mapreduce框架的JobControl实现 

~~~~java
ControlledJob cJob1 = new ControlledJob(job1.getConfiguration());
ControlledJob cJob2 = new ControlledJob(job2.getConfiguration());
ControlledJob cJob3 = new ControlledJob(job3.getConfiguration());

// 设置作业依赖关系
cJob2.addDependingJob(cJob1);
cJob3.addDependingJob(cJob2);

JobControl jobControl = new JobControl("RecommendationJob");
jobControl.addJob(cJob1);
jobControl.addJob(cJob2);
jobControl.addJob(cJob3);

cJob1.setJob(job1);
cJob2.setJob(job2);
cJob3.setJob(job3);

// 新建一个线程来运行已加入JobControl中的作业，开始进程并等待结束
Thread jobControlThread = new Thread(jobControl);
jobControlThread.start();
while (!jobControl.allFinished()) {
    Thread.sleep(500);
}
jobControl.stop();

return 0;
~~~~