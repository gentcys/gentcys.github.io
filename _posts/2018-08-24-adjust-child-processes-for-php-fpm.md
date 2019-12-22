---
title: "调整PHP-FPM子进程"
date: 2018-08-24 06:12:15 +0800
category: posts
comments: true
---
## 什么是PHP-FPM，以及PHP-FPM子进程

### PHP-FPM

PHP-FPM (FastCGI Process Manager) 是用于替换 PHP FastCGI 的大部分附加功能，对于高负载网站是非常有用的。

它的功能包括:

- 支持平滑停止/启动的高级进程管理功能;
- 可以工作于不同的 uid/gid/chroot 环境下，并监听不同的端口和使用不同的 php.ini 配置文件（可取代 safe_mode 的设置）
- stdout 和 stderr 日志记录
- 在发生意外情况的时候能够重新启动并缓存被破坏的 opcode
- 文件上传优化支持
- "慢日志" - 记录脚本（不仅记录文件名，还记录 PHP backtrace 信息，可以使用 ptrace或者类似工具读取和分析远程进程的运行数据）运行所导致的异常缓慢
- fastcgi_finish_request() - 特殊功能：用于在请求完成和刷新数据后，继续在后台执行耗时的工作（录入视频转换、统计处理等）
- 动态／静态子进程产生
- 基本 SAPI 运行状态信息（类似Apache的 mod_status）
- 基于 php.ini 的配置文件

### PHP-FPM 子进程

PHP-FPM 根据配置文件设置的参数进行 `fork`，从而衍生一定个数的子进程，衍生的方式有如下:

- `static`，子进程的数量是固定的，`pm = static`
- `onedemand` 子进程根据请求衍生， `pm = onedemand`
- `dynamic` 子进程的数量被动态设置，相关指令: `pm.max_children, pm.start_servers, pm.min_spare_servers, pm.max_spare_servers`

相关指令:
- `pm.max_children` 最大子进程数量
- `pm.start_servers` 起始子进程数量
- `pm.min_spare_servers` `static` 模式时当子进程数量少于 `pm.min_spare_servers` 时会补足数量
- `pm.max_spare_servers` `static` 模式时当子进程数量超过 `pm.max_spare_servers` 会杀死多余的进程
- `pm.max_requests` 当一个进程处理的请求累积到 `pm.max_requests` 个之后，自动重启该进程

## 为什么要调整PHP-FPM子进程

根据不同场景，设置计算后不同的设置指定 `PHP-FPM` 子进程的衍生方式，可以最大化的利用服务器的内存以及CPU，并不浪费也可以提升性能。

## 如何调整PHP-FPM子进程

PHP程序在执行完成后，都会存在内存泄漏的问题。所欲对于我们的服务器我们该如何选择呢？
对于内存小于 1G/2G 的服务器来说，选择 `dynamic` 会更有优势或者更小的 `onedemand`；对于大内存的服务器比如8G以上内存，`static` 是最好的选择。
这样不需要进行额外的进程数目控制，会提高效率。
建议 `pm.max_spare_servers`设置为 `内存大小`/30，`pm.min_spare_servers` 建议根据服务器的负载情况来设置。
`pm.max_request` 建议根据监控内存的使用变化来设置，在没有泄漏内存的条件之上加大数值如1024。
