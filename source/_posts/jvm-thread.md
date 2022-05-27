---
layout: post
title: "如何定位消耗CPU最多的线程"
date: 2017-06-29
excerpt: "觉得【你假笨】的文章不错，也有梯度，我也慢慢消化，消化掉的文章在这里记录下，以后自己可以回过头来看看"
categories: 
- Java
tags: 
- Thread
comments: true
---

话不多说了，先来看代码吧

```java
public class Test{
    public static void main(String args[]){
            for(int i=0;i<10;i++){
                    new Thread(){
                            public void run(){
                                    try{
                                            Thread.sleep(100000);
                                    }catch(Exception e){}
                            }
                    }.start();
            }
            Thread t=new Thread(){
                    public void run(){
                            int i=0;
                            while(true){
                                    i=(i++)/100;
                            }
                    }
            };
            t.setName("Busiest Thread");
            t.start();
    }
}
```
这个例子里新创建了11个线程，其中10个线程没干什么事，主要是sleep，另外有一个线程在循环里一直跑着，可以想象这个线程是这个进程里最耗cpu的线程了，那怎么把这个线程给抓出来呢？

首先我们可以通过top -Hp <pid>来看这个进程里所有线程的cpu消耗情况，得到类似下面的数据

```linux
$ top -Hp 18207
top - 19:11:43 up 573 days,  2:43,  2 users,  load average: 3.03, 3.03, 3.02
Tasks:  44 total,   1 running,  43 sleeping,   0 stopped,   0 zombie
Cpu(s): 18.8%us,  0.0%sy,  0.0%ni, 81.1%id,  0.0%wa,  0.0%hi,  0.0%si,  0.0%st
Mem:  99191752k total, 98683576k used,   508176k free,   128248k buffers
Swap:  1999864k total,   191064k used,  1808800k free, 17413760k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
18250 admin     20   0 26.1g  28m  10m R 99.9  0.0   0:19.50 java Test
18207 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18208 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.09 java Test
18209 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18210 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18211 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18212 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18213 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18214 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18215 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18216 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18217 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18218 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18219 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18220 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18221 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18222 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18223 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18224 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18225 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18226 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
18227 admin     20   0 26.1g  28m  10m S  0.0  0.0   0:00.00 java Test
```

拿到这个结果之后，我们可以看到cpu最高的线程是pid为18250的线程，占了99.8%：

```linux
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
18250 admin     20   0 26.1g  28m  10m R 99.9  0.0   0:19.50 java Test
```

接着我们可以通过jstack <pid>的输出来看各个线程栈:

```linux
$ jstack 18207
2016-03-30 19:12:23
Full thread dump OpenJDK 64-Bit Server VM (25.66-b60 mixed mode):

"Attach Listener" #30 daemon prio=9 os_prio=0 tid=0x00007fb90be13000 nid=0x47d7 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"DestroyJavaVM" #29 prio=5 os_prio=0 tid=0x00007fb96245b800 nid=0x4720 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"Busiest Thread" #28 prio=5 os_prio=0 tid=0x00007fb91498d000 nid=0x474a runnable [0x00007fb9065fe000]
   java.lang.Thread.State: RUNNABLE
    at Test$2.run(Test.java:18)

"Thread-9" #27 prio=5 os_prio=0 tid=0x00007fb91498c800 nid=0x4749 waiting on condition [0x00007fb906bfe000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at Test$1.run(Test.java:9)

"Thread-8" #26 prio=5 os_prio=0 tid=0x00007fb91498b800 nid=0x4748 waiting on condition [0x00007fb906ffe000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at Test$1.run(Test.java:9)

"Thread-7" #25 prio=5 os_prio=0 tid=0x00007fb91498b000 nid=0x4747 waiting on condition [0x00007fb9073fe000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at Test$1.run(Test.java:9)

"Thread-6" #24 prio=5 os_prio=0 tid=0x00007fb91498a000 nid=0x4746 waiting on condition [0x00007fb9077fe000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
    at java.lang.Thread.sleep(Native Method)
    at Test$1.run(Test.java:9)
...
```

上面的线程栈我们注意到nid的值其实就是线程ID，它是十六进制的，我们将消耗cpu最高的线程18250，转成十六进制0X47A，然后从上面的线程栈里找到nid=0X47A的线程，其栈为：

```linux
"Busiest Thread" #28 prio=5 os_prio=0 tid=0x00007fb91498d000 nid=0x474a runnable [0x00007fb9065fe000]
   java.lang.Thread.State: RUNNABLE
    at Test$2.run(Test.java:18)
```    

即将最耗cpu的线程找出来了，是Businest Thread