---
comments: true
date: 2011-02-25 20:44:03
layout: post
slug: php-ignore_user_abort
title: PHP的ignore_user_abort方法
wordpress_id: 133
categories:
- PHP
tags: [php]

---

按照PHP文档说明：


> 在 PHP 内部，系统维护着连接状态，其状态有三种可能的情况：

> 
> 
	
>   * 0 - NORMAL（正常）
> 
	
>   * 1 - ABORTED（异常退出）
> 
	
>   * 2 - TIMEOUT（超时）
> 

当 PHP 脚本正常地运行 NORMAL 状态时，连接为有效。当远程客户端中断连接时，ABORTED  状态的标记将会被打开。远程客户端连接的中断通常是由用户点击 STOP 按钮导致的。当连接时间超过 PHP 的时限（请参阅 [**set_time_limit()**](http://www.php.net/manual/en/function.set-time-limit.php) 函数）时，TIMEOUT 状态的标记将被打开。

可以决定脚本是否需要在客户端中断连接时退出。有时候让脚本完整地运行会带来很多方便，即使没有远程浏览器接受脚本的输出。默认的情况是当远程客户端连接中断时脚本将会退出。该处理过程可由  php.ini 的 ignore_user_abort 或由 Apache .conf  设置中对应的“php_value ignore_user_abort”以及 [**ignore_user_abort()**](http://www.php.net/manual/en/function.ignore-user-abort.php) 函数来控制。如果没有告诉 PHP 忽略用户的中断，脚本将会被中断，除非通过  [**register_shutdown_function()**](http://www.php.net/manual/en/function.register-shutdown-function.php) 设置了关闭触发函数。通过该关闭触发函数，当远程用户点击  STOP 按钮后，脚本再次尝试输出数据时，PHP 将会检测到连接已被中断，并调用关闭触发函数。


对于ignore_user_abort的理解误区主要是，认为应当用户在一个脚本加载期间点击了浏览器的“stop”按钮时：



	
  1. 服务器端如果没有调用ignore_user_abort(true)，那么服务器端脚本的执行将会中断

	
  2. 如果调用了ignore_user_abort(true), 那么服务器端脚本将会继续执行

	
  3. 点击stop后，通过connection_status方法，就可以获取连接状态为ABORTED


而实际上，只有php执行了输出操作，要将字符串输出到浏览器时才会去检测连接状态，也只有这时候才可以知道用户是否点击了"stop"按钮。当然，这是指后台脚本真正将字符串输出到浏览器时，如果中间存在缓存，则必须在缓存内容真正flush到浏览器时才可以知道连接状态（这里也需要注意一些杀毒软件会首先对网页输出进行缓存）。

实际上，ignore_user_abort的作用是：

	
  1. 当脚本执行前调用了ignore_user_abort(true)忽略用户中断后，当有输出操作需要将信息输出给浏览器时，脚本会检测到连接其实已经终止了，会将connection的status设置为ABORTED，但会继续执行后面的脚本，而不会异常退出

	
  2. 而当脚本没有设置忽略用户中断时，在第一次将信息输出到浏览器时，检测到连接中断，此时后台脚本就会直接异常退出，并不会继续执行下去


也就是说，如果一个PHP脚本没有输出，那么不论是否用户中断，也无论是否设置了忽略中断(ignore_user_abort)，脚本均会执行下去直至结束（除非超过脚本最长执行时间或者脚本本身异常)；当PHP脚本有输出时，仅会在输出到达浏览器时可以检测到用户是否已经中断，并根据ignore_user_abort的设置来决定是直接结束还是继续进行。
