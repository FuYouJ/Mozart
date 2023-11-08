  最近想把disruptor这个队列使用到自己项目中去，简单写了一波demo来波性能测试结果发现是负优化，目前阶段觉得是原来的代码写的太好了，具体好在哪里，后面再仔细分析。于是想着是不是可以从其他成功应用的项目上学到一些最佳实践呢。
当我翻阅代码发现，一行代码格外的醒目。
```java
Thread.sleep(0);
```
这是为了解决什么问题？为什么要这么写？用Thread.yield()行不行。
* 用Thread.yield()行不行

  答案是可以的，Thread.yield()和Thread.sleep()底层实现都是一样的。

  Thread.yield()底层实现

  ```cpp
  JVM_ENTRY(void, JVM_Yield(JNIEnv *env, jclass threadClass))
    JVMWrapper("JVM_Yield");
    //检查是否设置了DontYieldALot参数,默认为fasle
    //如果设置为true,直接返回
    if (os::dont_yield()) return;
   //如果ConvertYieldToSleep=true（默认为false）,调用os::sleep,否则调用os::yield
    if (ConvertYieldToSleep) {
      os::sleep(thread, MinSleepInterval, false);//sleep 1ms
    } else {
      os::yield();
    }
  
  void os::yield(){
    sched_yield();
  }
  ```

  Thread.sleep()底层实现

  ```cpp
  JVM_ENTRY(void, JVM_Sleep(JNIEnv* env, jclass threadClass, jlong millis))
    JVMWrapper("JVM_Sleep");
  
    if (millis < 0) {//参数校验
      THROW_MSG(vmSymbols::java_lang_IllegalArgumentException(), "timeout value is negative");
    }
  
    //如果线程已经中断，抛出中断异常,关于中断的实现，在另一篇文章中会讲解
    if (Thread::is_interrupted (THREAD, true) && !HAS_PENDING_EXCEPTION) {
      THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
    }
   //设置线程状态为SLEEPING
    JavaThreadSleepState jtss(thread);
  
    EventThreadSleep event;
  
    if (millis == 0) {
      //如果设置了ConvertSleepToYield(默认为true),和yield效果相同
      if (ConvertSleepToYield) {
        os::yield();
      } else {//否则调用os::sleep方法
        ThreadState old_state = thread->osthread()->get_state();
        thread->osthread()->set_state(SLEEPING);
        os::sleep(thread, MinSleepInterval, false);//sleep 1ms
        thread->osthread()->set_state(old_state);
      }
    } else {//参数大于0
     //保存初始状态，返回时恢复原状态
      ThreadState old_state = thread->osthread()->get_state();
      //osthread->thread status mapping:
      // NEW->NEW
      //RUNNABLE->RUNNABLE
      //BLOCKED_ON_MONITOR_ENTER->BLOCKED
      //IN_OBJECT_WAIT,PARKED->WAITING
      //SLEEPING,IN_OBJECT_WAIT_TIMED,PARKED_TIMED->TIMED_WAITING
      //TERMINATED->TERMINATED
      thread->osthread()->set_state(SLEEPING);
      //调用os::sleep方法，如果发生中断，抛出异常
      if (os::sleep(thread, millis, true) == OS_INTRPT) {
        if (!HAS_PENDING_EXCEPTION) {
          if (event.should_commit()) {
            event.set_time(millis);
            event.commit();
          }
          THROW_MSG(vmSymbols::java_lang_InterruptedException(), "sleep interrupted");
        }
      }
      thread->osthread()->set_state(old_state);//恢复osThread状态
    }
    if (event.should_commit()) {
      event.set_time(millis);
      event.commit();
    }
  ```

  可以看到当Thread.sleep(long n)传入参数是0的时候，和yield()是一个方法。

* 解决了什么问题

在Seatunnel的任务同步中，有checkpoint机制，和数据库的类似，框架可以通过checkpoint知道自己的任务同步到那个点了，有一个用户想通过触发checkpoint来停止任务，但是他发现不能够在预期的时间内停止，获取checkpoint的锁总是失败。

分析了这个问题后，他发现checkpointLock是通过synchronized锁定的，而synchronized是不公平的锁。在单核环境中，由于 CPU 负载过高，线程饥饿的可能性更大。检查点流无法获取 checkpointLock。

于是他在代码里加入了一行代码```Thread.sleep(0L)```，就解决了这个问题。

