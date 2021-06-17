---
title: XXL-JOB任务一直卡在进行中状态排查
tags:
  - xxl-job
  - 经验
categories:
  - 问题排查
date: 2021-06-17 19:11:11
---

&ensp;&ensp;最近发现 xxl-job（xxl-job是一个轻量级分布式任务调度平台） 的任务一直卡在进行中状态，期间 dump 过执行器的进程，发现线程已经没了，而且没有任何异常日志，网上查看原因，这是作者给的答案：

![avatar](/images/xxl-job-running-check/1.png)

&ensp;&ensp;于是查询任务调度日志，发现确实没有回调。但是仍旧不明白线程为啥突然没了，而且还不报错。..后来就看 xxl-job 源码，看看 xxl-job 的调度原理，但因为有工作原因也还没查到，先放一边了。

&ensp;&ensp;直到今天补偿优惠券发放，发现没发完，一看任务还在执行中，dump 下进程却发现线程已经没了，才知道又碰到了这问题，还是没有任何异常日志。

![avatar](/images/xxl-job-running-check/2.png)

&ensp;&ensp;看上图处于运行中的任务（因为截图慢了，当时第一个 taskId = 88 的任务调度时间是 2020-11-15 04:05:20），调度机器和第三个任务是同一个（因为该任务阻塞处理策略设置的是丢弃后续调度，所以碰到这种情况肯定不正常（第三个任务不应该处于进行中状态，第一个任务确实是处于运行中，只是任务运行时间很长））。

&ensp;&ensp;仔细看这张图，发现第二个任务 taskId = 86 这个也不应该一直处于进行中状态，这个任务执行时间很短的，而且它调度的时间和第一个调度任务调度时间非常接近，难道这期间发生了什么？

&ensp;&ensp;开始排查这个运行的比较快的任务。

![avatar](/images/xxl-job-running-check/3.png)

&ensp;&ensp;如上图，查询这个任务的关键字日志，发现了一条 main 线程打的日志很特别：xxl-job register jobhandler success ,好像见过，搜一下看看

![avatar](/images/xxl-job-running-check/4.png)

&ensp;&ensp;有很多条，想起这是注册任务的日志。。查看源码

![avatar](/images/xxl-job-running-check/5.png)

&ensp;&ensp;发现是在启动时注册的，难道程序重启了？

&ensp;&ensp;再次搜索重启关键字

![avatar](/images/xxl-job-running-check/6.png)

&ensp;&ensp;还真的重启了..,难怪任务线程突然没了，早应该想到的。

![avatar](/images/xxl-job-running-check/12.webp)

&ensp;&ensp;这是重启前与重启后的两条日志（后面要用到），查看包发布记录没人操作过进程，难道包发布自动重启了？

&ensp;&ensp;于是联系运维开发热线，确认了包发布不会自动重启进程，除非进程挂了，包发布会自动拉起。

![avatar](/images/xxl-job-running-check/7.png)

&ensp;&ensp;查看监控，也能看到进程确认重启了，但重启前内存好好的，full gc 却发生在重启后。。这时突然想到测试机有碰到过由于系统内存不足导致进程被杀的情况，于是看了一下机器内存

![avatar](/images/xxl-job-running-check/8.png)

&ensp;&ensp;确实剩余不多了，完全有可能，于是查询相关日志。

![avatar](/images/xxl-job-running-check/9.png)

&ensp;&ensp;可以看到光标那行，04:04:28 有个 Java 进程被杀了，时间刚刚对应上面图中进程重启前的最后一条日志。

&ensp;&ensp;至此，终于破案了，**系统内存不足，导致进程被杀，来不及回调 xxl-job,并且没有任何异常，然后再被包发布拉起，xxl-job 中的任务也就一直处于进行中了。**

![avatar](/images/xxl-job-running-check/10.png)

&ensp;&ensp;xxl-job 确实做了这种情况的处理。但是没有生效，阅读源码发现

```base
            // monitor
while (!toStop) {
	try {
		// 任务结果丢失处理：调度记录停留在 "运行中" 状态超过10min，且对应执行器心跳注册失败不在线，则将本地调度主动标记失败；
		Date losedTime = DateUtil.addMinutes(new Date(), -10);
		List<Long> losedJobIds  = XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().findLostJobIds(losedTime);
		if (losedJobIds!=null && losedJobIds.size()>0) {
			for (Long logId: losedJobIds) {
				XxlJobLog jobLog = new XxlJobLog();
				jobLog.setId(logId);
				jobLog.setHandleTime(new Date());
				jobLog.setHandleCode(ReturnT.FAIL_CODE);
				jobLog.setHandleMsg( I18nUtil.getString("joblog_lost_fail") );
				XxlJobAdminConfig.getAdminConfig().getXxlJobLogDao().updateHandleInfo(jobLog);
			}
		}
	} catch (Exception e) {
		if (!toStop) {
			logger.error(">>>>>>>>>>> xxl-job, job fail monitor thread error:{}", e);
		}
	}
    try {
        TimeUnit.SECONDS.sleep(60);
    } catch (Exception e) {
        if (!toStop) {
            logger.error(e.getMessage(), e);
        }
    }
}
```

&ensp;&ensp;JobLosedMonitorHelper 类有一个线程在干这事，任务结果丢失处理：调度记录停留在 "运行中" 状态超过 10min，且对应执行器心跳注册失败不在线，则将本地调度主动标记失败。该任务确实有运行超过 10min,但是再看执行器里面的 XxlJobSpringExecutor 类：

```bash
public class XxlJobSpringExecutor extends XxlJobExecutor implements ApplicationContextAware, SmartInitializingSingleton, DisposableBean {
    private static final Logger logger = LoggerFactory.getLogger(XxlJobSpringExecutor.class);


    // start
    @Override
    public void afterSingletonsInstantiated() {

        // init JobHandler Repository
        /*initJobHandlerRepository(applicationContext);*/

        // init JobHandler Repository (for method)
        initJobHandlerMethodRepository(applicationContext);

        // refresh GlueFactory
        GlueFactory.refreshInstance(1);

        // super start
        try {
            super.start();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    // destroy
    @Override
    public void destroy() {
        super.destroy();
    }
}
```
&ensp;&ensp;destory 方法会去执行 ExecutorRegistryThread 中的 stop 方法：

```bash
public void toStop() {
    toStop = true;
    // interrupt and wait
    registryThread.interrupt();
    try {
        registryThread.join();
    } catch (InterruptedException e) {
        logger.error(e.getMessage(), e);
    }
}
```

&ensp;&ensp;ExecutorRegistryThread 中的 register 线程代码如下：

```bash
registryThread = new Thread(new Runnable() {
	@Override
	public void run() {
		// registry
		while (!toStop) {
			try {
				RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
				for (AdminBiz adminBiz : XxlJobExecutor.getAdminBizList()) {
					try {
						ReturnT<String> registryResult = adminBiz.registry(registryParam);
						if (registryResult != null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
							registryResult = ReturnT.SUCCESS;
							logger.debug(">>>>>>>>>>> xxl-job registry success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
							break;
						} else {
							logger.info(">>>>>>>>>>> xxl-job registry fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
						}
					} catch (Exception e) {
						logger.info(">>>>>>>>>>> xxl-job registry error, registryParam:{}", registryParam, e);
					}
				}
			} catch (Exception e) {
				if (!toStop) {
					logger.error(e.getMessage(), e);
				}
			}
			try {
				if (!toStop) {
					TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
				}
			} catch (InterruptedException e) {
				if (!toStop) {
					logger.warn(">>>>>>>>>>> xxl-job, executor registry thread interrupted, error msg:{}", e.getMessage());
				}
			}
		}
		// registry remove
		try {
			RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
			for (AdminBiz adminBiz : XxlJobExecutor.getAdminBizList()) {
				try {
					ReturnT<String> registryResult = adminBiz.registryRemove(registryParam);
					if (registryResult != null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
						registryResult = ReturnT.SUCCESS;
						logger.info(">>>>>>>>>>> xxl-job registry-remove success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
						break;
					} else {
						logger.info(">>>>>>>>>>> xxl-job registry-remove fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
					}
				} catch (Exception e) {
					if (!toStop) {
						logger.info(">>>>>>>>>>> xxl-job registry-remove error, registryParam:{}", registryParam, e);
					}
				}
			}
		} catch (Exception e) {
			if (!toStop) {
				logger.error(e.getMessage(), e);
			}
		}
		logger.info(">>>>>>>>>>> xxl-job, executor registry thread destory.");
	}
});
```

&ensp;&ensp;可以看到 register remove 部分有去 registerRemove 当前执行器地址并且打印相关日志，但是系统强制 kill 了进程，导致 destroy()方法没有执行，执行器没有进行 registerRemove 回调，xxl-job admin 也就无法感知执行器已经失败不在线，导致没有终止这个运行大于 10min 的任务，看重启前的日志也可以说明确实没有进行 registerRemove 回调，所以不管什么系统做好优雅关闭确实很重要。







