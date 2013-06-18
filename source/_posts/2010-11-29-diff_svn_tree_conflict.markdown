---
comments: true
date: 2010-11-29 20:37:07
layout: post
title: 整体diff比较SVN的tree conflict文件
wordpress_id: 108
categories:  [Linux]
tags:  [svn, tree_conflict]

---

通过svn merge合并代码时经常会报树冲突(tree conflict)，而对于每个树冲突不能简单的忽略了事，需要和待merge版本相应的文件进行diff才可以。

目前我们组的并行项目很多，svn merge时，树冲突往往非常多，一个一个手动去diff是一个很繁琐的过程，下午大概写了一个shell方法对所有树冲突的文件进行diff，可以将以下代码加入~/.bash_profile文件：

{%codeblock batch_diff lang:shell%}
#设定项目所在目录
export PRJ=/home/work/var/www/htdocs
#根据svn info获取当前目录层次，并构造相应的diff目的文件路径
#如当前在$PRJ/pro_merge/protected/modules/operations下
#则"getDir $PRJ/pro"的结果是$PRJ/pro/protected/modules/operations
#当然前提是svn info中的URL行对应的格式是https://*/*_BRANCH/protected/modules/operations或https://*/*_PD_BL/protected/modules/operations
#可根据实际情况进行修改
function getDir() { DIR=`echo $1 | sed 's/\//\\\\\//g'`; svn info | grep "^URL" | awk '{print $2}' | sed "s/^https:.*\(_BRANCH\|_PD_BL\)/$DIR/g"; }
#通过svn st和awk获取所有树冲突文件相对路径
#依次和指定的目标文件进行diff,其中忽略了对.png .gif等类型文件的处理
function td() { svn st | grep "^[[:space:]]\+C" | awk '$2!~/(.*\.png)|(.*\.gif)/ {print $2}' | xargs -t -i diff -u --exclude="\.svn" {} "`getDir $1`/{}"; }
{%endcodeblock%}

假设合并代码后的目录是$PRJ/pro_merge,而合入的版本代码在$PRJ/pro_src,则可以在$PRJ/pro_merge的任意子目录下执行如下命令：

<!--more-->

{%codeblock batch_diff lang:shell%}
td $PRJ/pro_src
{%endcodeblock%}

即可输出当前目录下所有子目录的树冲突文件和带合并文件的diff情况

+++++++2010年12月6日更新++++++++++

该方法可以进一步扩展，将diff出来的没有不同的文件直接”svn resolved“了，因此写了一个简单的shell脚本：

{%codeblock batch_diff lang:shell%}

#! /bin/sh
function getDir() { DIR=`echo $1 | sed 's/\//\\\\\//g'`; svn info | grep "^URL" | awk '{print $2}' | sed "s/^https:.*\(_BRANC
H\|_PD_BL\)/$DIR/g"; }

files=`svn st | grep "^[[:space:]]\+C" | awk '$2!~/(.*\.png)|(.*\.gif)/ {print $2}'`
dir=`getDir $1`
for file in $files
do
    echo "$file"
    out=`diff -ur --exclude="\.svn" "$file" "$dir/$file" 2>&1`
    if [ $? -eq "0" ]
    then
        echo will svn resolved $file
        svn resolved $file
    else
        echo "$out"
    fi
done

{%endcodeblock%}
