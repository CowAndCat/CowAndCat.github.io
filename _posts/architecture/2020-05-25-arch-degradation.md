---
layout: post
title: 服务降级 
category: arch
comments: false
--- 

谨以此篇文章记录一次通宵两天+每日临晨2点下班+周末两天加班的项目开发。

# 一、项目背景

在AI时代下的一个toG项目，为了推广一次政治活动的知识要点。

下层依赖ASR语音识别服务，因为语音服务存在性能瓶颈，且容易雪崩——当一台服务器超过承载能力后，请求被转发给其他服务器，造成其他服务器也超过负载，最终整个服务不可用。

# 二、服务降级

为了避免雪崩，需要在雪崩前捕获到不正常行为，并做降级处理。

在本项目中，当ASR语音服务出现超时现象，就可以认为服务出现了不正常，但是也有可能是偶发的错误。

于是加上基于固定窗口的统计机制，如果近期5次请求里，有超过2次的超时响应，那么立马熔断服务，对于熔断期间的请求均返回错误或服务不可用。然后在某个随机时间后恢复。这里用随机时间是为了避免多个业务服务器在同一时间一起恢复，给后端服务造成突发压力。

另外，还可以采用平滑恢复的机制，比如小流量恢复，如果流量正常，再逐渐放开。

## 2.1 实现

窗口统计基于Guava的EvictQueue, FIFO，超出长度的元素会被剔除。

超时监控基于Guava的SimpleTimeLimiter（基于Future），会在线程池中运行单个ASR任务，当超时会抛出TimeoutException异常。

```
import com.google.common.collect.EvictingQueue;
import com.google.common.util.concurrent.SimpleTimeLimiter;
import com.google.common.util.concurrent.TimeLimiter;

public class AudioService {
    private static TimeLimiter timeLimiter;
    private static EvictingQueue<Integer> requestSampleWindow;   
    private static AtomicLong circuitRecoverTime = new AtomicLong(System.currentTimeMillis() - 1000L);

    private int reqSampleSize = 5;
    private double timeoutInSampleRatio = 0.4;
    private int circuitRecoverMaxInSec = 60;
    private int circuitRecoverMinInSec = 30;

    @PostConstruct
    public void init() {
        timeLimiter = SimpleTimeLimiter.create(new ThreadPoolExecutor(5, 10, 0L, TimeUnit.MILLISECONDS,
                new SynchronousQueue<>()));

        requestSampleWindow = EvictingQueue.create(reqSampleSize);
    }

    public void innerHandle(String audio) {
        checkCircuit();
        List<Result> results = null;
        try {
            results = timeLimiter.callWithTimeout(
                    () -> asrClient.syncRecognize(audio),
                    4,
                    TimeUnit.SECONDS);
            samplingTimeout(false);
        } catch (TimeoutException | InterruptedException | ExecutionException e) {
            log.warn("timeout", e);
            // 超时采样
            samplingTimeout(true);
            if (checkSamplingTimeoutRatio()) {
                log.error("Timeout too frequently, then circuit break.", e);
                // 随机生成恢复时间。
                long randCircuitTime = circuitRecoverMaxInSec +
                        new Random().nextInt(circuitRecoverMinInSec - circuitRecoverMaxInSec + 1);
                circuitRecoverTime.set(System.currentTimeMillis() + randCircuitTime * 1000);
                resetSampling();
            }
            throw new AsrResponseTimeout();
        } catch (RejectedExecutionException e) {
            log.error("Too many request", e);
            throw new FailedToParseAudio();
        }
    }

    // 检查是否在熔断期
    private void checkCircuit() {
        if (circuitRecoverTime.get() > System.currentTimeMillis()) {
            log.warn("Asr service is in circuit.");
            throw new AsrCircuitException();
        }
    }

    // 窗口采样
    private synchronized void samplingTimeout(boolean timeout) {
        requestSampleWindow.offer(timeout ? 1 : 0);
    }

    // 触发熔断时重置窗口
    private synchronized void resetSampling() {
        requestSampleWindow.clear();
    }

    // 检查是否达到触发熔断
    private boolean checkSamplingTimeoutRatio() {
        int timeoutCount = 0;
        for (int item : requestSampleWindow) {
            if (item == 1) {
                ++timeoutCount;
            }
        }
        return Double.compare(1.0 * timeoutCount / reqSampleSize, timeoutInSampleRatio) > 0;
    }
```

上述就是一个避免雪崩、加了熔断的服务降级实现。（已将业务相关代码剔除）

