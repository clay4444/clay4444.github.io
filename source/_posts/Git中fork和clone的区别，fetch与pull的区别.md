---
title: Git中fork和clone的区别，fetch与pull的区别
tags:
  - Git
abbrlink: 44e5bf34
date: 2018-04-23 09:40:22
---

### fork和clone的区别

fork：在github页面，点击fork按钮。将别人的仓库复制一份到自己的仓库。

clone：将github中的仓库克隆到自己本地电脑中。

<br/>

### pull request的作用

比如在仓库的主人（A）没有把我们添加为项目合作	者的前提下，我们将A的某个仓库名为“a”的仓库clone到自己的电脑中，在自己的电脑进行修改，但是我们会发现我们没办法通过push将代码贡献到B中。

<br/>

所以要想将你的代码贡献到B中，我们应该：

1. 在A的仓库中fork项目a （此时我们自己的github就有一个一模一样的仓库a，但是URL不同）
2. 将我们修改的代码push到自己github中的仓库B中
3. pull request ，主人就会收到请求，并决定要不要接受你的代码

也可以可以申请为项目a的contributor，这样可以直接push。

<br/>

### fork了别人的项目到自己的repository之后，别人的项目更新了，我们fork的项目怎么更新？

首先fetch网上的更新到自己的项目上，然后再判断、merge。这里就涉及了下一个问题，pull和fetch有啥区别。

<br/>

### pull和fetch的区别

fetch+merge与pull效果一样。但是要多用fetch+merge，这样可以检查fetch下来的更新是否合适。pull直接包含了这两步操作，如果你觉得网上的更新没有问题，那直接pull也是可以的。

<br/>

### git fetch

相当于是从远程获取最新版本到本地，不会自动merge

```
git fetch origin master
git log -p master..origin/master
git merge origin/master
```

以上命令的含义：

首先从远程的origin的master主分支下载最新的版本到origin/master分支上。
然后比较本地的master分支和origin/master分支的差别。
最后进行合并。

<br/>

上述过程其实可以用以下更清晰的方式来进行：

```
git fetch origin master:tmp
git diff tmp
git merge tmp
```

从远程获取最新的版本到本地的test分支上

之后再进行比较合并

<br/>

### git pull

相当于是从远程获取最新版本并merge到本地

```
git pull origin master
```

上述命令其实相当于git fetch 和 git merge

在实际使用中，git fetch更安全一些

因为在merge前，我们可以查看更新情况，然后再决定是否合并。