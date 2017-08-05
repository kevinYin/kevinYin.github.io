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
一是 git checkout -b 命令; 二是 日期,就是当天的日期;三是项目名,项目名又是跟当前目录绑定的,比如我们的项目是 trade-service,
一般会打个分支叫 20160927_trade_fixBug,所以项目名就是目录的第一个单词,单词之间是 _ 隔开的. 了解需求之后,就可以逐个点实现,
shell获取当前时间是 `date +%Y%m%d`, 项目名就是获取当前目录,并且按照 - 切割获得第一个元素,所以也就是
cdir="${PWD##*/}";pname=`expr $cdir | cut -d "-" -f1`**
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

## 2. z.sh ， a new autojump
首先，这个东西不是我写的，但是由于这个插件非常的实用，所以把它搬到这里来说下。  
之前刚刚开始用mac的时候，会有提到autojump这个插件，相对来讲会非常实用，可以快速地跳转，但是前提是你必须要写的相对清楚目标文件夹的名字，如果遇到多个文件名一样，但是路径不一样的时候，autojump就会选择其中一个，但是在跳转后你才知道调到哪一个，而不是在enter之前你就知道autojump帮你跳转到的目的目录。   
Z就可以。  
直接说例子：  
终端输入： **z kevin**  
我的所有文件夹目录有多个包含**kevin**字符串的目录，Z 插件会默认调出完全匹配的一个目录 **z /Users/kevin/**，如果这个目录不是你索要找的，那么再按一次tab键，Z 就会把所有符合 **kevin** 的目录打印出来供你选择：  

> _posts git:master ❯ z /Users/kevin/  
> /Users/kevin/.oh-my-zsh  
> /Users/kevin/github  
> /Users/kevin/github/kevinYin.github.io  
> /Users/kevin/github/kevinYin.github.io/_posts  
> /Users/kevin/soft  
> /Users/kevin/soft/Z  
> /Users/kevin/soft/Z/z  

如何安装使用，访问**https://github.com/rupa/z**，写的很详细。对mac用户来讲，日常开发绝对是一款神器。  
