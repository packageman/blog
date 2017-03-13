---
title: Linux 中关于 bash 不传递信号
date: 2016-07-01 09:39:14
tags:
---

## 引言

最近在做一个 PHP(所使用的 PHP 版本为 5.6.13) 的项目，由于 PHP 是同步执行的，而项目中又需要一些异步处理了任务，于是引入了 [PHP-Resque](https://github.com/chrisboulton/php-resque) 这个开源框架来实现了一个任务队列。使用 [Supervisor](http://supervisord.org/) 来管理任务队列中的 resque worker(后文将统一使用 worker 表示) 进程。

## 问题产生

我们在业务上有一个异步发短信的业务，由于需求变更更改了短信模板，在修改完短信模板后通过 `supervisorctl restart {worker_name}` 重起 worker 后, 发现发送的短信中仍有部分短信使用旧的短信模板。

## 找出原因

作为第一次遇到这种奇怪事情的第一反应——“见鬼了！修改的代码没有生效？不能啊，明明有的短信是正确的啊”。但是这种诡异事情的确发生了，于是我作为一名程序员的本能反应就是 ssh 到服务器上 `ps -ef | grep resque` 一下。以下是部分截图

![Resque-process](/images/resque-supervisor/resque_processes.png)

<!-- more -->

发现有一些 worker 的进程的父进程 ID 是 1，即 init 进程。而在正常情况下 worker 进程是由 supervisor 管理的，他们的父(或祖先)进程应该是 supervisor 的进程 ID。

![Supervisor-process](/images/resque-supervisor/supervisor_process.png)

正好最近在看《UNIX环境高级编程》这本书，里面有介绍，当一个进程的父进程终止后，这个进程就会变为孤儿进程，从而会被 init 进程领养。所以我猜想父进程是 1 的 worker 就是由于 supervisor 在重启 worker 时并没有把旧的 worker 的进程全部清理掉导致的。

首先把这些"有问题"的 worker 手动清理掉，再测试一下发短信功能，一些正常。看来就是这个“有问题”的 worker 捣的鬼了，可是这些“有问题”的 worker 是如何产生的？

读了下 PHP-Resque 的源码，发现：
- worker 在启动前会注册一个信号处理函数，专门用来处理进程在执行过程中收到的信号。
- worker 在启动后会定时去取任务队列中的任务，如果有任务则 worker 进程将 fork 出一个子进程去执行任务，父进程等待子进程完成并退出后，继续执行取任务的操作。以下是部分代码片段：

```php
<?php
// https://github.com/chrisboulton/php-resque/blob/master/lib/Resque/Worker.php#L145
public function work($interval = Resque::DEFAULT_INTERVAL, $blocking = false)
{
	$this->updateProcLine('Starting');
	// startup 中会调用 registerSigHandlers
	$this->startup();
	while(true) {
		if($this->shutdown) {
			break;
		}
		...
		$this->child = Resque::fork();
		// Forked and we're the child. Run the job.
		if ($this->child === 0 || $this->child === false) {
			$status = 'Processing ' . $job->queue . ' since ' . strftime('%F %T');
			$this->updateProcLine($status);
			$this->logger->log(Psr\Log\LogLevel::INFO, $status);
			$this->perform($job);
			if ($this->child === 0) {
				exit(0);
			}
		}
		if($this->child > 0) {
			// Parent process, sit and wait
			$status = 'Forked ' . $this->child . ' at ' . strftime('%F %T');
			$this->updateProcLine($status);
			$this->logger->log(Psr\Log\LogLevel::INFO, $status);
			// Wait until the child process finishes before continuing
			pcntl_wait($status);
			$exitStatus = pcntl_wexitstatus($status);
			if($exitStatus !== 0) {
				$job->fail(new Resque_Job_DirtyExitException(
					'Job exited with exit code ' . $exitStatus
				));
			}
		}
		$this->child = null;
		$this->doneWorking();
	}
	$this->unregisterWorker();
}

// https://github.com/chrisboulton/php-resque/blob/master/lib/Resque/Worker.php#L343
private function registerSigHandlers()
{
	if(!function_exists('pcntl_signal')) {
		return;
	}
	pcntl_signal(SIGTERM, array($this, 'shutDownNow'));
	pcntl_signal(SIGINT, array($this, 'shutDownNow'));
	pcntl_signal(SIGQUIT, array($this, 'shutdown'));
	pcntl_signal(SIGUSR1, array($this, 'killChild'));
	pcntl_signal(SIGUSR2, array($this, 'pauseProcessing'));
	pcntl_signal(SIGCONT, array($this, 'unPauseProcessing'));
	$this->logger->log(Psr\Log\LogLevel::DEBUG, 'Registered signals');
}
```

看到这儿，我猜想会不会是因为，在 worker fork 一个子进程后，这个子进程正在执行任务，这时使用 supervisor 重启 worker 导致了父进程收到了 *SIGTERM* 或 *SIGINT* 信号立即退出(shutDowNow)。而子进程还在执行任务(未退出)导致了子进程被 init 进程领养？

然而这个猜想在我看完 [`shutDownNow`](https://github.com/chrisboulton/php-resque/blob/master/lib/Resque/Worker.php#L391), [`shutdown`](https://github.com/chrisboulton/php-resque/blob/master/lib/Resque/Worker.php#L381), [`killChild`](https://github.com/chrisboulton/php-resque/blob/master/lib/Resque/Worker.php#L401) 这些信号回调函数的实现后被无情的否定了。请看以下代码：

```php
<?php
public function shutdownNow()
{
	$this->shutdown();
	$this->killChild();
}

public function shutdown()
{
	$this->shutdown = true;
	...
}

public function killChild()
{
	...
	if(exec('ps -o pid,state -p ' . $this->child, $output, $returnCode) && $returnCode != 1) {
		...
		posix_kill($this->child, SIGKILL);
		$this->child = null;
	}
	...
}
```

可以看到 worker 进程的退出是通过 `$this->shutdown` 变量控制的，而这个变量的检查是在子进程退出后或根本没有任务可执行的时候判断的。所以不论是 `shutdown` 还是 `shutDownNow`，父进程的退出都不会有子进程存在。

题外话：[Supervisor 在关闭进程时默认发送的是 *SIGTERM* 并设置 10 秒种的超时时间，如果超过这个时间进程仍然没有关闭，Supervisor 将发送 *SIGKILL* 强制关闭该进程](http://supervisord.org/configuration.html#program-x-section-values)。为了避免在重新启动 worker 的时候导致正在执行的任务由于强制终止而失败，在生产环境中我们应该使用 *SIGQUIT* 停止 worker 并根据业务需要设置合理的超时时间以实现平滑关闭。

目前为止，我了解到的是 Supervisor 重启发送 *SIGTERM* (这里我使用的是默认的配置)首先关闭 worker, worker 强制结束子进程(如果有的话)再退出，Supervisor 启动新的 worker。这一切都看起来很合理，可事实却并非如此。

到底是哪里出问题了？这个问题困扰了我很久，终于读到了这篇文章 [Docker and the PID 1 zombie reaping problem](https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/)，里面提到在使用 `/bin/bash -c` 启动进程的时候一定要小心，因为这样启动的进程相当于先执行 bash，再由 bash 创建一个子进程执行真正要执行的进程。而 bash 是不会处理信号的也不会传播信号给子进程，这样在发送 *SIGTERM* 给 bash 进程就会导致 bash 进程退出，bash 的子进程变为孤儿进程... 读到这，感觉这一幕似曾相识啊，赶紧看看 worker 在 Supervisor 中配置文件，果然是用 bash 启动的。以下是 Supervisor 中的 worker 的配置文件：

![Supervisor-config](/images/resque-supervisor/supervisor_config.png)

现在我终于搞清楚了为什么使用 supervisor 重起 worker 会导致一些父进程为 1 的 worker 进程产生了。我把这个过程用图的方式展现出来，希望能更容易理解一点:)

![Resque-process-model](/images/resque-supervisor/supervisor_process_model.png)

## 解决问题

现在已经找到问题了，该解决问题了。其实思路很简单，可以从两方面考虑：

- 不再使用 bash 的方式启动，改为使用 `exec` (适用于在进程结束没有后续工作要处理的情况)
- 通过代码捕获 bash 收到的信号，并将信号发送给子进程

这里我使用第二种方式，以下是示例代码：

```sh
#!/bin/bash

shutdown() {
  echo 'gracefully shutting down resque worker...'
  kill -s QUIT ${pid}
  wait ${pid}
}

trap shutdown SIGINT SIGQUIT SIGTERM

phpbrew use 5.6.13
php /srv/.../bin/resque &

pid=$!
wait ${pid}
```

上代码表示当 bash 进程收到 *SIGINT* 或 *SIGQUIT* 或 *SIGTERM* 信号后会执行函数 `shutdown`，进而会发送 *SIGQUIT* 给 worker 进程并等待 worker 进程结束。

## 总结

- 都是书读的少，还总结啥，看书去了:)

## 参考资料

- https://github.com/chrisboulton/php-resque
- http://supervisord.org/
- https://blog.phusion.nl/2015/01/20/docker-and-the-pid-1-zombie-reaping-problem/
- https://github.com/inetfuture/blog/issues/3
- http://veithen.github.io/2014/11/16/sigterm-propagation.html


