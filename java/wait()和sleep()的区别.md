　　这两个方法来自不同的类分别是，sleep来自Thread类，和wait来自Object类。
sleep是Thread的静态类方法，谁调用的谁去睡觉，即使在a线程里调用b的sleep方法，实际上还是a去睡觉，要让b线程睡觉要在b的代码中调用sleep。

　　锁: 最主要是sleep方法没有释放锁，而wait方法释放了锁，使得其他线程可以使用同步控制块或者方法。　　
　　
　　sleep不出让系统资源；wait是进入线程等待池等待，出让系统资源，其他线程可以占用CPU。一般wait不会加时间限制，因为如果wait线程的运行资源不够，再出来也没用，要等待其他线程调用notify/notifyAll唤醒等待池中的所有线程，才会进入就绪队列等待OS分配系统资源。sleep(milliseconds)可以用时间指定使它自动唤醒过来，如果时间不到只能调用interrupt()强行打断。
Thread.sleep(0)的作用是“触发操作系统立刻重新进行一次CPU竞争”。

　使用范围：wait，notify和notifyAll只能在同步控制方法或者同步控制块里面使用，而sleep可以在任何地方使用。

```
   synchronized(x){ 
      x.notify() 
     //或者wait() 
   }
```

