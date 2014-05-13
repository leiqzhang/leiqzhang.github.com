---
comments: true
date: 2014-05-04 22:59:31
layout: post
title: 'DevStack环境的Python版本升级和UT环境搭建'
categories:  [OpenStack]
tags:  [devstack, virtualenv, pip]

---

# DevStack环境的Python版本升级和UT环境搭建

## 背景

在CentOS 6.5上面安装了devstack，由于6.5默认的python是2.6.6版本，在进行UT时，只能采用nosetests的方式，而此种方式和gerrit检测粒度不一致，往往造成在本地执行测试用例全部通过而上传到社区后测试用例不通过的情况。

通过执行run_tests.sh的方式，可以保持和gerrit检测粒度一致，但其需要python 2.7版本。

本文记录了直接在现有环境上升级python版本，并保证devstack相应升级及UT环境正常搭建的过程。

## 升级Python版本

CentOS 6.5还没有现成的Python 2.7版本的RPM包，故需要从源码编译安装。由于系统中还存在Python 2.6.6，为了不与其产生环境冲突，会通过virtualenv的方式对不同版本的Python进行环境隔离。

### 安装源码编译所需的包

```
yum groupinstall "Development tools"
yum install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel
```

### 下载Python 2.7源码并编译安装

```
wget http://python.org/ftp/python/2.7.6/Python-2.7.6.tar.xz
tar xvf Python-2.7.6.tar.xz
cd Python-2.7.6
./configure --prefix=/usr/local --enable-unicode=ucs4 --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"
make && make altinstall
```

需要注意的是，这里通过make altinstall的方式可保证安装出来的python文件带有版本信息，如python2.7。同时通过configure制定prefix，将python 2.7安装到/usr/local目录下。

如上步骤完成后，会在/usr/local/bin目录下安装有python2.7等文件

### 安装distribute

```
wget https://pypi.python.org/packages/source/d/distribute/distribute-0.7.3.zip#md5=c6c59594a7b180af57af8a0cc0cf5b4a

unzip distribute-0.7.3.zip

cd distribute-0.7.3

python2.7 setup.py install
```
通过如上命令，python2.7的程序包路径为/usr/local/lib/python2.7/site-packages/

### 安装virtualenv

```
easy_install-2.7 virtualenv
```

5. 使用virtualenv

```
cd /opt

virtualenv-2.7 --distribute stackenv27
```
如上命令会在/opt目录下创建名字为stackenv27的目录，该目录下会包含bin、include和lib等几个子目录，其实际是通过拷贝文件或者符号链接的方式创建python2.7的虚拟环境。如/opt/stackenv27/bin目录下的文件如下：

```
-rw-r--r--. 1 root root 2196 Apr 30 01:59 activate
-rw-r--r--. 1 root root 1252 Apr 30 01:59 activate.csh
-rw-r--r--. 1 root root 2465 Apr 30 01:59 activate.fish
-rw-r--r--. 1 root root 1129 Apr 30 01:59 activate_this.py
-rwxr-xr-x. 1 root root  246 Apr 30 01:59 easy_install
-rwxr-xr-x. 1 root root  246 Apr 30 01:59 easy_install-2.7
-rwxr-xr-x. 1 root root  218 Apr 30 01:59 pip
-rwxr-xr-x. 1 root root  218 Apr 30 01:59 pip2
-rwxr-xr-x. 1 root root  218 Apr 30 01:59 pip2.7
lrwxrwxrwx. 1 root root    9 Apr 30 01:59 python -> python2.7
lrwxrwxrwx. 1 root root    9 Apr 30 01:59 python2 -> python2.7
-rwxr-xr-x. 1 root root 9776 Apr 30 01:59 python2.7
```

如上所示，此目录下会有python2.7文件，同时会有python和python2俩符号链接指向此文件。

要启用此python2.7环境，可以通过上述目录的activate文件来启动：

```
source /opt/stackenv27/bin/activate

#output：
[root@stack bin]# source /opt/stackenv27/bin/activate
(stackenv27)[root@stack bin]# which python
/opt/stackenv27/bin/python
```

## 升级devstack

```
cd /opt/stack

find . -name "requirements.txt" -exec pip install --upgrade -r {} \;
```
## 重启devstack中服务

退出之前的screen session

```
[stack@stack ~]$ screen -ls
There is a screen on:
	9181.stack	(Detached)
1 Socket in /var/run/screen/S-stack.

[stack@stack ~]$ screen -X -S 9181 quit
```

然后activate Python2.7虚拟环境，并重新启动screen

```
source /opt/stackenv27/bin/activate

cd /home/stack/devstack/

./rejoin-stack.sh
```

其中一些服务的启动入口是/usr/bin/xxx, 比如/usr/bin/cinder-api, 其默认是用/usr/bin/python来执行的，需要将其启动方式进行相应更改, 以强制其使用虚拟环境下的python进行运行。

比如如果启动方式为:

```
/usr/bin/cinder-api --config-file /etc/cinder/cinder.conf
```
则可改为：

```
`which python` /usr/bin/cinder-api --config-file /etc/cinder/cinder.conf
```
## 典型问题记录

当启动cinder-api服务时，报如下错误：

```
ImportError: No module named MySQLdb
```

则是因为在升级devstack时没有安装成功相关python库导致，此时手动安装：

```
pip install --upgrade mysql-python
```

但此过程中会在gcc编译时报类似如下错误：

```
    _mysql.c:2195: error: ‘_mysql_ConnectionObject’ has no member named ‘open’
    _mysql.c:2196: error: ‘_mysql_ConnectionObject’ has no member named ‘converter’
    _mysql.c:2205: error: ‘_mysql_ResultObject’ has no member named ‘result’
    _mysql.c: In function ‘_mysql_ConnectionObject_dealloc’:
```

此类错误是因为mysql-devel rpm未安装导致，需要首先通过yum安装mysql-devel包，然后再安装mysql-python即可
