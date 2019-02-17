---
title: MR之社交粉丝数据查找共同好友
categories:
  - big-data
tags:
  - MapReduce
abbrlink: a2cd52e0
date: 2018-02-12 17:08:17
---

### 需求

以下是qq的好友列表数据，冒号前是一个用，冒号后是该用户的所有好友（数据中的好友关系是单向的）

A:B,C,D,F,E,O

B:A,C,E,K

C:F,A,D,I

D:A,E,F,L

E:B,C,D,M,L

F:A,B,C,D,E,O,M

G:A,C,D,E,F

H:A,C,D,E,O

求出哪些人两两之间有共同好友，及他俩的共同好友都有谁？

<br/>

### 解体思路

**第一步  **

map

读一行   A:B,C,D,F,E,O

输出   \<B,A>\<C,A>\<D,A>\<F,A>\<E,A>\<O,A>

在读一行   B:A,C,E,K

输出  \<A,B>\<C,B>\<E,B>\<K,B>

<br/>

REDUCE

拿到的数据比如\<C,A>\<C,B>\<C,E>\<C,F>\<C,G>......

输出：  

\<A-B,C>

\<A-E,C>

\<A-F,C>

\<A-G,C>

\<B-E,C>

\<B-F,C>.....

<br/>

**第二步**

map

读入一行\<A-B,C>

直接输出\<A-B,C>

 <br/>

reduce

读入数据 \<A-B,C>\<A-B,F>\<A-B,G>.......

输出：

 A-B  C,F,G,.....

<br/>

## 代码实现

#### 第一步，查找一个人是哪些人的共同好友

```java
public class SharedFriendsStepOne {

  static class SharedFriendsStepOneMapper extends Mapper<LongWritable, Text, Text, Text> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
      // A:B,C,D,F,E,O
      String line = value.toString();
      String[] person_friends = line.split(":");
      String person = person_friends[0];
      String friends = person_friends[1];

      for (String friend : friends.split(",")) {

        // 输出<好友，人>
        context.write(new Text(friend), new Text(person));
      }

    }

  }

  static class SharedFriendsStepOneReducer extends Reducer<Text, Text, Text, Text> {

    @Override
    protected void reduce(Text friend, Iterable<Text> persons, Context context) throws IOException, InterruptedException {

      StringBuffer sb = new StringBuffer();

      for (Text person : persons) {
        sb.append(person).append(",");

      }
      context.write(friend, new Text(sb.toString()));
    }

  }

  public static void main(String[] args) throws Exception {

    Configuration conf = new Configuration();

    Job job = Job.getInstance(conf);
    job.setJarByClass(SharedFriendsStepOne.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(Text.class);

    job.setMapperClass(SharedFriendsStepOneMapper.class);
    job.setReducerClass(SharedFriendsStepOneReducer.class);

    FileInputFormat.setInputPaths(job, new Path("D:/srcdata/friends"));
    FileOutputFormat.setOutputPath(job, new Path("D:/temp/out"));

    job.waitForCompletion(true);

  }

}
```

<br/>

#### 第二步 

拿到的数据是上一个步骤的输出结果 A I,K,C,B,G,F,H,O,D,

友 人，人，人

发出 <人-人，好友> ，这样，相同的“人-人”对的所有好友就会到同1个reduce中去

~~~~java
public class SharedFriendsStepTwo {

  static class SharedFriendsStepTwoMapper extends Mapper<LongWritable, Text, Text, Text> {

    // 拿到的数据是上一个步骤的输出结果
    // A I,K,C,B,G,F,H,O,D,
    // 友 人，人，人
    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {

      String line = value.toString();
      String[] friend_persons = line.split("\t");

      String friend = friend_persons[0];
      String[] persons = friend_persons[1].split(",");

      Arrays.sort(persons);

      for (int i = 0; i < persons.length - 1; i++) {
        for (int j = i + 1; j < persons.length; j++) {
          // 发出 <人-人，好友> ，这样，相同的“人-人”对的所有好友就会到同1个reduce中去
          context.write(new Text(persons[i] + "-" + persons[j]), new Text(friend));
        }

      }

    }

  }

  static class SharedFriendsStepTwoReducer extends Reducer<Text, Text, Text, Text> {

    @Override
    protected void reduce(Text person_person, Iterable<Text> friends, Context context) throws IOException, InterruptedException {

      StringBuffer sb = new StringBuffer();

      for (Text friend : friends) {
        sb.append(friend).append(" ");

      }
      context.write(person_person, new Text(sb.toString()));
    }

  }

  public static void main(String[] args) throws Exception {

    Configuration conf = new Configuration();

    Job job = Job.getInstance(conf);
    job.setJarByClass(SharedFriendsStepTwo.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(Text.class);

    job.setMapperClass(SharedFriendsStepTwoMapper.class);
    job.setReducerClass(SharedFriendsStepTwoReducer.class);

    FileInputFormat.setInputPaths(job, new Path("D:/temp/out/part-r-00000"));
    FileOutputFormat.setOutputPath(job, new Path("D:/temp/out2"));

    job.waitForCompletion(true);

  }

}
~~~~

