---
layout: post  
title:  "用shell写的小工具"  
date:   2016-09-27 23:16  
categories: 工具  
permalink: /Priest/shell-tool

---

一个同事说过,喜欢用mac做开发的原因是可以用shell来为自己写很多小程序来提高开发效率,以前不觉得怎么样,到后面反反复复做过
一些重复性的工作后,就愈发觉得似乎是有很多重复的工作其实是可以通过写点小程序去替代人工处理.  

## 1.用git打分支  

** 需求:** 厂里采用了git进行打分支,常规的操作是:  

> git checkout 20160927_projectName_描述  

这个看起来已经很简洁了,就一行命令,但是写多了发现还是有很多东西是重复的  
一是 git checkout 命令; 二是 日期,就是当天的日期;三是项目名,项目名又是跟当前目录绑定的,比如我们的项目是 trade-service,
一般会打个分支叫 20160927_trade_fixBug,所以项目名就是目录的第一个单词,单词之间是"_"隔开的. 了解需求之后,就可以逐个点实现,
shell获取当前时间是 **`date +%Y%m%d`**, 项目名就是获取当前目录,并且按照 - 切割获得第一个元素,所以也就是 **cdir="${PWD##*/}";pname=`expr $cdir | cut -d "-" -f1`**
所以最终的实现就是:

```
alias gb='function gb(){
        tname="`date +%Y%m%d`";
        cdir="${PWD##*/}";
        pname=`expr $cdir | cut -d "-" -f1`;
        git checkout -b $tname"_"$pname"_"$1; 
  };
gb'
```
写在 ~/.bash_profile,然后source下就可以用了,效果如下 :

```
    trade-service git:master # gb fixBug                                                                     
    Switched to a new branch '20160926_trade_fixBug'
    trade-service git:20160926_trade_fixBug #
```
虽然很多人打字快,看起来是没有任何的区别,但是我觉得一个程序员开发过程就是一个不断思考的过程,能敲2次键盘实现的坚决不要敲4次.  

## 2.git提交代码