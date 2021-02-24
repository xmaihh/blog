---
title: ANR程序问题分析之traces
date: 2018-07-13 17:50:52
categories: Android Framework
tags: [Android]
toc: true
description: You are my destiny, not a sudden thought hit me.
---
每次发生ANR，这个文件都会被清空，写入新的内容。如果想查看以前发生ANR的信息，可以去查看DB文件，
也就是DropBox的中的日志
跟踪功能，保存历史上发生的所有ANR的日志 
“/ data / system / dropbox”是DB指定的文件存放位置。
日志保存的最长时间，默认是3天
### ANR的异常信息
使用logcat命令查看会得到类似如下的log

```
//WindowManager所在的进程是system_server，进程号是127
I/WindowManager( 127): Input event dispatching timed out sending to com.example.anrdemo/com.example.anrdemo.ANRActivity
//system_server进程中的ActivityManagerService请求kernel向5033进程发送SIGNAL_QUIT请求
//你可以在shell中使用命令达到相同的目的：adb shell kill -3 5033
//和其他的Java虚拟机一样，SIGNAL_QUIT也是Dalvik内部支持的功能之一
I/Process ( 127): Sending signal. PID: 5033 SIG: 3
//5033进程的虚拟机实例接收到SIGNAL_QUIT信号后会将进程中各个线程的函数堆栈信息输出到traces.txt文件中
//发生ANR的进程正常情况下会第一个输出
I/dalvikvm( 5033): threadid=4: reacting to signal 3
I/dalvikvm( 5033): Wrote stack traces to '/data/anr/traces.txt'
... ...//另外还有其他一些进程

//随后会输出CPU使用情况
E/ActivityManager( 127): ANR in com.example.anrdemo (com.example.anrdemo/.ANRActivity)
//Reason表示导致ANR问题的直接原因
E/ActivityManager( 127): Reason: keyDispatchingTimedOut
E/ActivityManager( 127): Load: 3.85 / 3.41 / 3.16

//请注意ago，表示ANR发生之前的一段时间内的CPU使用率，并不是某一时刻的值
E/ActivityManager( 127): CPU usage from 26835ms to 3662ms ago with 99% awake:
E/ActivityManager( 127): 9.4% 98/mediaserver: 9.4% user + 0% kernel
E/ActivityManager( 127): 8.9% 127/system_server: 6.9% user + 2% kernel / faults: 1823 minor
... ...
E/ActivityManager( 127): +0% 5033/com.example.anrdemo: 0% user + 0% kernel
E/ActivityManager( 127): 39% TOTAL: 32% user + 6.1% kernel

//这里是later，表示ANR发生之后
E/ActivityManager( 127): CPU usage from 601ms to 1132ms later with 99% awake:
E/ActivityManager( 127): 10% 127/system_server: 1.7% user + 8.9% kernel / faults: 5 minor
E/ActivityManager( 127): 10% 163/InputDispatcher: 1.7% user + 8.9% kernel
E/ActivityManager( 127): 1.7% 127/system_server: 1.7% user + 0% kernel
E/ActivityManager( 127): 1.7% 135/SurfaceFlinger: 0% user + 1.7% kernel
E/ActivityManager( 127): 1.7% 2814/Binder Thread #: 1.7% user + 0% kernel
... ...
E/ActivityManager( 127): 37% TOTAL: 27% user + 9.2% kernel
```
发生ANR时Android为我们提供了两种“利器”：traces文件和CPU使用率
### ANR信息是如何输出的
ActivityManagerService类中，找到它的appNotResponding函数
```
    final void appNotResponding(ProcessRecord app, ActivityRecord activity,
                                ActivityRecord parent, final String annotation) {
//firstPids和lastPids两个集合存放那些将会在traces中输出信息的进程的进程号
        ArrayList<Integer> firstPids = new ArrayList<Integer>(5);
        SparseArray<Boolean> lastPids = new SparseArray<Boolean>(20);
//mController是IActivityController接口的实例，是为Monkey测试程序预留的，默认为null
        if (mController != null) {
... ...
        }
        long anrTime = SystemClock.uptimeMillis();
        if (MONITOR_CPU_USAGE) {
            updateCpuStatsNow(); //更新CPU使用率
        }
        synchronized (this) { //一些特定条件下会忽略ANR
            if (mShuttingDown) {
                Slog.i(TAG, "During shutdown skipping ANR: " + app + " " + annotation);
                return;
            } else if (app.notResponding) {
                Slog.i(TAG, "Skipping duplicate ANR: " + app + " " + annotation);
                return;
            } else if (app.crashing) {
                Slog.i(TAG, "Crashing app skipping ANR: " + app + " " + annotation);
                return;
            }
            //使用一个标志变量避免同一个应用在没有处理完时重复输出log
            app.notResponding = true;
... ...
         //①当前发生ANR的应用进程被第一个添加进firstPids集合中
            firstPids.add(app.pid);
... ...
        }
... ...
        //②dumpStackTraces是输出traces文件的函数
        File tracesFile = dumpStackTraces(true, firstPids, processStats, lastPids, null);
        String cpuInfo = null;
        if (MONITOR_CPU_USAGE) { //MONITOR_CPU_USAGE默认为true
            updateCpuStatsNow(); //再次更新CPU信息
            synchronized (mProcessStatsThread) {
                //输出ANR发生前一段时间内的CPU使用率
                cpuInfo = mProcessStats.printCurrentState(anrTime);
            }
            info.append(processStats.printCurrentLoad());
            info.append(cpuInfo);
        }
        //输出ANR发生后一段时间内的CPU使用率
        info.append(processStats.printCurrentState(anrTime));
... ...
        //③将ANR信息同时输出到DropBox中
        addErrorToDropBox("anr", app, app.processName, activity, parent, annotation,
                cpuInfo, tracesFile, null);
... ...
        //在Android4.0中可以设置是否不显示ANR提示对话框，如果设置的话就不会显示对话框，并且会杀掉ANR进程
        boolean showBackground = Settings.Secure.getInt(mContext.getContentResolver(),
                Settings.Secure.ANR_SHOW_BACKGROUND, 0) != 0;
        synchronized (this) {
            if (!showBackground && !app.isInterestingToUserLocked() && app.pid != MY_PID) {
... ...
                Process.killProcessQuiet(app.pid);
                return;
            }
... ...
            // 显示ANR提示对话框
            Message msg = Message.obtain();
            HashMap map = new HashMap();
            msg.what = SHOW_NOT_RESPONDING_MSG;
            msg.obj = map;
            map.put("app", app);
            if (activity != null) {
                map.put("activity", activity);
            }
            mHandler.sendMessage(msg);
        }
    }
```
当前发生ANR的应用进程被第一个添加进firstPids集合中，所以会第一个向traces文件中写入信息。反过来说，traces文件中出现的第一个进程正常情况下就是发生ANR的那个进程。不过有时候会很不凑巧，发生ANR的进程还没有来得及输出trace信息，就由于某种原因退出了，所以偶尔会遇到traces文件中找不到发生ANR的进程信息的情况。
dumpStackTraces是输出traces文件的函数，接下来分析这个函数
addErrorToDropBox函数将ANR信息同时输出到DropBox中，它也是个非常有用的日志存放工具，后面也会分析它的作用。
```
 public static File dumpStackTraces(boolean clearTraces, ArrayList<Integer> firstPids,
                                       ProcessStats processStats, SparseArray<Boolean> lastPids, String[] nativeProcs) {
//系统属性“dalvik.vm.stack-trace-file”用来配置trace信息输出文件
        String tracesPath = SystemProperties.get("dalvik.vm.stack-trace-file", null);
        if (tracesPath == null || tracesPath.length() == 0) {
            return null;
        }
        File tracesFile = new File(tracesPath);
        try {
            File tracesDir = tracesFile.getParentFile();
            if (!tracesDir.exists()) tracesFile.mkdirs();
//FileUtils.setPermissions是个很有用的函数，设置文件属性时经常会用到
            FileUtils.setPermissions(tracesDir.getPath(), 0775, -1, -1); // drwxrwxr-x
//clearTraces为true，会删除旧文件，创建新文件
            if (clearTraces && tracesFile.exists()) tracesFile.delete();
            tracesFile.createNewFile();
            FileUtils.setPermissions(tracesFile.getPath(), 0666, -1, -1); // -rw-rw-rw-
        } catch (IOException e) {
            Slog.w(TAG, "Unable to prepare ANR traces file: " + tracesPath, e);
            return null;
        }
//一个重载函数
        dumpStackTraces(tracesPath, firstPids, processStats, lastPids, nativeProcs);
        return tracesFile;
    }
```
之所以trace信息会输出到“/data/anr/traces.txt”文件中，就是系统属性“dalvik.vm.stack-trace-file”设置的。你可以通过在设备的shell中使用setprop和getprop对系统属性进行设置和读取：
```
getpropdalvik.vm.stack-trace-file
setprop dalvik.vm.stack-trace-file /tmp/stack-traces.txt
```
每次发生ANR时都会删除旧的traces文件，重新创建新文件。也就是说Android只保留最后一次发生ANR时的traces信息，那么以前的traces信息就丢失了么？稍后回答。
接着来看重载的dumpStackTraces函数。
```
   private static void dumpStackTraces(String tracesPath, ArrayList<Integer> firstPids,
                                        ProcessStats processStats, SparseArray<Boolean> lastPids, String[] nativeProcs) {
//使用FileObserver监听应用进程是否已经完成写入traces文件的操作
//Android在判断桌面壁纸文件是否设置完成时也是用的FileObserver，很有用的类
        FileObserver observer = new FileObserver(tracesPath, FileObserver.CLOSE_WRITE) {
            public synchronized void onEvent(int event, String path) {
                notify();
            }
        };
... ...
//首先输出firstPids集合中指定的进程，这些也是对ANR问题来说最重要的进程
        if (firstPids != null) {
            try {
                int num = firstPids.size();
                for (int i = 0; i < num; i++) {
                    synchronized (observer) {
//前面提到的SIGNAL_QUIT
                        Process.sendSignal(firstPids.get(i), Process.SIGNAL_QUIT);
                        observer.wait(200);
... ...
                    }
```
### 分析ANR的trace信息
```
//文件中输出的第一个进程的trace信息，正是发生ANR的演示程序
//开头显示进程号、ANR发生的时间点和进程名称
----- pid 9183 at 2012-09-28 22:20:42 -----
Cmd line: com.example.anrdemo
 
DALVIK THREADS: //以下是各个线程的函数堆栈信息
//mutexes表示虚拟机实例中各种线程相关对象锁的value值
(mutexes: tll=0 tsl=0 tscl=0 ghl=0 hwl=0 hwll=0)
//依次是：线程名、线程优先级、线程创建时的序号、①线程当前状态
"main" prio=5 tid=1 TIMED_WAIT
//依次是：线程组名称、suspendCount、debugSuspendCount、线程的Java对象地址、线程的Native对象地址
| group="main" sCount=1 dsCount=0 obj=0x4025b1b8 self=0xce68
//sysTid是线程号，主线程的线程号和进程号相同
| sysTid=9183 nice=0 sched=0/0 cgrp=default handle=-1345002368
| schedstat=( 140838632 210998525 213 )
at java.lang.VMThread.sleep(Native Method)
at java.lang.Thread.sleep(Thread.java:1213)
at java.lang.Thread.sleep(Thread.java:1195)
at com.example.anrdemo.ANRActivity.makeANR(ANRActivity.java:44)
at com.example.anrdemo.ANRActivity.onClick(ANRActivity.java:38)
at android.view.View.performClick(View.java:2486)
at android.view.View$PerformClick.run(View.java:9130)
at android.os.Handler.handleCallback(Handler.java:587)
at android.os.Handler.dispatchMessage(Handler.java:92)
at android.os.Looper.loop(Looper.java:130)
at android.app.ActivityThread.main(ActivityThread.java:3703)
at java.lang.reflect.Method.invokeNative(Native Method)
at java.lang.reflect.Method.invoke(Method.java:507)
at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:841)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:599)
at dalvik.system.NativeStart.main(Native Method)
 
//②Binder线程是进程的线程池中用来处理binder请求的线程
"Binder Thread #2" prio=5 tid=8 NATIVE
| group="main" sCount=1 dsCount=0 obj=0x40750b90 self=0x1440b8
| sysTid=9190 nice=0 sched=0/0 cgrp=default handle=1476256
| schedstat=( 915528 18463135 4 )
at dalvik.system.NativeStart.run(Native Method)
 
"Binder Thread #1" prio=5 tid=7 NATIVE
| group="main" sCount=1 dsCount=0 obj=0x4074f848 self=0x78d40
| sysTid=9189 nice=0 sched=0/0 cgrp=default handle=1308088
| schedstat=( 3509523 25543212 10 )
at dalvik.system.NativeStart.run(Native Method)
 
//线程名称后面标识有daemon，说明这是个守护线程
"Compiler" daemon prio=5 tid=6 VMWAIT
| group="system" sCount=1 dsCount=0 obj=0x4074b928 self=0x141e78
| sysTid=9188 nice=0 sched=0/0 cgrp=default handle=1506000
| schedstat=( 21606438 21636964 101 )
at dalvik.system.NativeStart.run(Native Method)
 
//JDWP线程是支持虚拟机调试的线程，不需要关心
"JDWP" daemon prio=5 tid=5 VMWAIT
| group="system" sCount=1 dsCount=0 obj=0x4074b878 self=0x16c958
| sysTid=9187 nice=0 sched=0/0 cgrp=default handle=1510224
| schedstat=( 366211 2807617 7 )
at dalvik.system.NativeStart.run(Native Method)
//“Signal Catcher”负责接收和处理kernel发送的各种信号，例如SIGNAL_QUIT、SIGNAL_USR1等就是被该线程
//接收到，这个文件的内容就是由该线程负责输出的，可以看到它的状态是RUNNABLE，不过此线程也不需要关心
"Signal Catcher" daemon prio=5 tid=4 RUNNABLE
| group="system" sCount=0 dsCount=0 obj=0x4074b7b8 self=0x150008
| sysTid=9186 nice=0 sched=0/0 cgrp=default handle=1501664
| schedstat=( 1708985 6286621 9 )
at dalvik.system.NativeStart.run(Native Method)
 
"GC" daemon prio=5 tid=3 VMWAIT
| group="system" sCount=1 dsCount=0 obj=0x4074b710 self=0x168010
| sysTid=9185 nice=0 sched=0/0 cgrp=default handle=1503184
| schedstat=( 305176 4821778 2 )
at dalvik.system.NativeStart.run(Native Method)
 
"HeapWorker" daemon prio=5 tid=2 VMWAIT
| group="system" sCount=1 dsCount=0 obj=0x4074b658 self=0x16a080
| sysTid=9184 nice=0 sched=0/0 cgrp=default handle=550856
| schedstat=( 33691407 26336669 15 )
at dalvik.system.NativeStart.run(Native Method)
 
----- end 9183 -----
 
----- pid 127 at 2012-09-28 22:20:42 -----
Cmd line: system_server
... ...
//省略其他进程的信息
```
### 线程状态

| Thread.java中定义的状态 | Thread.cpp中定义的状态 | 说明 |
|--- |---|---|
| TERMINATED | ZOMBIE | 线程死亡，终止运行 |
| RUNNABLE | RUNNING/RUNNABLE | 线程可运行或正在运行 |
| TIMED_WAITING | TIMED_WAIT | 执行了带有超时参数的wait、sleep或join函数 |
| BLOCKED | MONITOR | 线程阻塞，等待获取对象锁 |
| WAITING | WAIT | 执行了无超时参数的wait函数 |
| NEW | INITIALIZING | 新建，正在初始化，为其分配资源 |
| NEW | STARTING | 新建，正在启动 |
| RUNNABLE | NATIVE | 正在执行JNI本地函数 |
| WAITING | VMWAIT | 正在等待VM资源 |
| RUNNABLE | SUSPENDED | 线程暂停，通常是由于GC或debug被暂停 |
|          | UNKNOWN | 未知状态 |
Thread.java中的状态和Thread.cpp中的状态是有对应关系的。可以看到前者更加概括，也比较容易理解，面向Java的使用者；而后者更详细，面向虚拟机内部的环境。traces.txt中显示的线程状态都是Thread.cpp中定义的。另外，所有的线程都是遵循POSIX标准的本地线程。关于线程更多的说明可以查阅源码/dalvik/vm/Thread.cpp中的说明。<!-- 线程的ThreadGroup最好也写进去 -->
traces.txt文件中的这些信息是由每个Dalvik进程的SignalCatcher线程输出的，相关代码可以查看/dalvik/vm/目录下的SignalCatcher.cpp::logThreadStacks函数和Thread.cpp:: dvmDumpAllThreadsEx函数。另外请注意，输出堆栈信息时SignalCatcher会暂停所有线程。
通过该文件很容易就能知道问题进程的主线程发生ANR时正在执行怎样的操作。例如上述示例， ANRActivity在makeANR函数中执行线程sleep时发生ANR，可以推测sleep时间过长，超过了超时上限导致。
### CPU使用率
内容读取自/proc/stat
```
E/ActivityManager( 127): ANR in com.example.anrdemo (com.example.anrdemo/.ANRActivity)
E/ActivityManager( 127): Reason: keyDispatchingTimedOut
E/ActivityManager( 127): Load: 3.85 / 3.41 / 3.16 //➀ CPU平均负载
 
//②ANR发生之前的一段时间内的CPU使用率
E/ActivityManager( 127): CPU usage from 26835ms to 3662ms ago with 99% awake://③
E/ActivityManager( 127): 9.4% 98/mediaserver: 9.4% user + 0% kernel
E/ActivityManager( 127): 8.9% 127/system_server: 6.9% user + 2% kernel / faults: 1823 minor //⑤ minor或者major的页错误次数
... ...
E/ActivityManager( 127)://⑥+0% 5033/com.example.anrdemo: 0% user + 0% kernel
E/ActivityManager( 127): 39% TOTAL: 32% user + 6.1% kernel
 
//⑦ANR发生之后的一段时间内的CPU使用率
E/ActivityManager( 127): CPU usage from 601ms to 1132ms later with 99% awake:
E/ActivityManager( 127): 10% 127/system_server: 1.7% user + 8.9% kernel / faults: 5 minor
E/ActivityManager( 127): 10% 163/InputDispatcher: 1.7% user + 8.9% kernel
E/ActivityManager( 127): 1.7% 127/system_server: 1.7% user + 0% kernel
E/ActivityManager( 127): 1.7% 135/SurfaceFlinger: 0% user + 1.7% kernel
E/ActivityManager( 127): 1.7% 2814/Binder Thread #: 1.7% user + 0% kernel
... ...
E/ActivityManager( 127): 37% TOTAL: 27% user + 9.2% kernel
```
#### CPU负载/平均负载

CPU负载是指某一时刻系统中运行队列长度之和加上当前正在CPU上运行的进程数，而CPU平均负载可以理解为一段时间内正在使用和等待使用CPU的活动进程的平均数量。在Linux中“活动进程”是指当前状态为运行或不可中断阻塞的进程。通常所说的负载其实就是指平均负载。
用一个从网上看到的很生动的例子来说明（不考虑CPU时间片的限制），把设备中的一个单核CPU比作一个电话亭，把进程比作正在使用和等待使用电话的人，假如有一个人正在打电话，有三个人在排队等待，此刻电话亭的负载就是4。使用中会不断的有人打完电话离开，也会不断的有其他人排队等待，为了得到一个有参考价值的负载值，可以规定每隔5秒记录一下电话亭的负载，并将某一时刻之前的一分钟、五分钟、十五分钟的的负载情况分别求平均值，最终就得到了三个时段的平均负载。
实际上我们通常关心的就是在某一时刻的前一分钟、五分钟、十五分钟的CPU平均负载，例如以上日志中这三个值分别是3.85、3.41、3.16，说明前一分钟内正在使用和等待使用CPU的活动进程平均有3.85个，依此类推。在大型服务器端应用中主要关注的是第五分钟和第十五分钟的两个值，但是Android主要应用在便携手持设备中，有特殊的软硬件环境和应用场景，短时间内的系统的较高负载就有可能造成ANR，所以笔者认为一分钟内的平均负载相对来说更具有参考价值。
CPU的负载和使用率没有必然关系，有可能只有一个进程在使用CPU，但执行的是复杂的操作；也有可能等待和正在使用CPU的进程很多，但每个进程执行的都是简单操作。
实际处理问题时偶尔会遇到由于平均负载高引起的ANR，典型的特征就是系统中应用进程数量多，CPU总使用率较高，但是每个进程的CPU使用率不高，当前应用进程主线程没有异常阻塞，一分钟内的CPU平均负载较高。

>提示：Linux内核不断进行着CPU负载的记录，我们可以在任意时刻通过在shell中执行“cat /proc/loadavg”查看。

#### ANR发生之前和之后一段时间的CPU使用率
CPU使用率可以理解为一段时间（记作T）内除CPU空闲时间（记作I）之外的时间与这段时间T的比值，用公式表示可以写为：
```
CPU使用率= (T – I) / T
```
而时间段T是两个采样时间点的时间差值。
之所以可以这样计算，是因为Linux内核会把从系统启动开始到当前时刻CPU活动的所有时间信息都记录下来，我们可以通过查看“/proc/stat”文件获取这些信息。主要包括以下几种时间，这些时间都是从系统启动开始计算的，单位都是0.01秒：
- user： CPU在用户态的运行时间，不包括nice值为负数的进程运行的时间
- nice： CPU在用户态并且nice值为负数的进程运行的时间
- system：CPU在内核态运行的时间
- idle： CPU空闲时间，不包括iowait时间
- iowait： CPU等待I/O操作的时间
- irq： CPU硬中断的时间
- softirq：CPU软中断的时间

>注意：随着Linux内核版本的不同，包含的时间类型有可能不同,Android源码只需要关心以上七种类型即可

CPU使用率的计算是在ProcessStats类中实现的。如果在某两个时刻T1和T2（T1 < T2）进行采样记录，CPU使用率的整个算法可以归纳为以下几个公式：
```
userTime = (user2 + nice2) – (user1 + nice1)
systemTime = system2 - system1
idleTime = idle2 - idle1
iowaitTime = iowait2 - iowait1
irqTime = irq2 - irq1
softirqTime = softirq2 - softirq1
TotalTime = userTime + systemTime + idleTime + iowaitTime + irqTime + softirqTime
```
有了以上数据就可以计算具体的使用率了，例如用户态CPU使用率为：
```userCpuUsage = userTime / TotalTime```
依此类推可以计算其他类型的使用率。而整个时间段内CPU使用率为：
```CpuUsage = (TotalTime – idleTime) / TotalTime```
以上计算的是整个系统的CPU使用率，对于指定进程的使用率是通过读取该进程的“/proc/进程号/stat”文件计算的，而对于指定进程的指定线程的使用率是通过读取该线程的“/proc/进程号/task/线程号/stat”文件计算的。进程和线程的CPU使用率只包含该进程或线程的总使用率、用户态使用率和内核态使用率。
AMS在执行appNotResponding函数过程中，共输出了两个时间段的CPU使用率，通常情况下在ANR发生时间点之前和之后各有一段。两段使用率都是两次调用ProcessStats对象的update函数，每次调用update函数时会保留上一时间点的采样数据，并记录当前时间点的采样数据。然后再调用ProcessStats对象的printCurrentState函数
代码位置ActivityManagerService.java → appNotResponding
```

    //第一次使用成员变量mProcessStats采样
if(MONITOR_CPU_USAGE)

    {
        updateCpuStatsNow();
    }
......
    //声明了一个局部变量，参数true表示包括线程信息
    final ProcessStats processStats = new ProcessStats(true);
    //将processStats作为实参传入，在dumpStackTraces中相隔500毫秒两次调用其update函数进行采样
    File tracesFile = dumpStackTraces(true, firstPids, processStats, lastPids);
    String cpuInfo = null;
if(MONITOR_CPU_USAGE)

    {
//因为在第一次调用后，可能由于输出trace信息等操作，中间执行了较长的时间，所以有第二次使用成员变量
//mProcessStats采样，尽量使得采样时间点靠后。
//此函数中要求连续两次采样时间间隔不少于5秒，所以一般不会执行第二次采样。一旦执行，就会出现两个采样
//时间点一个在ANR发生之前，另一个在其之后，或者两个时间点都在ANR发生之后的情况。
        updateCpuStatsNow();
        synchronized (mProcessStatsThread) {
//mProcessStats是成员变量，创建对象时传入的参数是false，所以不包括线程信息
//此处先输出ANR发生之前一段时间内的CPU使用率
            cpuInfo = mProcessStats.printCurrentState(anrTime);
        }
        info.append(processStats.printCurrentLoad());
        info.append(cpuInfo);
    }
//processStats对象是在ANR发生后创建并采样的，所以输出的是ANR发生之后一段时间内的CPU使用率
info.append(processStats.printCurrentState(anrTime));
```
#### 非睡眠时间百分比

在记录CPU使用率的每个采样时间点时使用了两种记录方法：SystemClock.uptimeMillis()和SystemClock.elapsedRealtime()，两者的区别就是uptimeMillis不包含睡眠时间，所以两个采样时间点之间的uptimeMillis和elapsedRealtime之比就是非睡眠时间百分比。

#### 页错误次数

进程的CPU使用率最后输出的“faults: xxx minor/major”部分表示的是页错误次数，当次数为0时不显示。major是指Major Page Fault（主要页错误，简称MPF），内核在读取数据时会先后查找CPU的高速缓存和物理内存，如果找不到会发出一个MPF信息，请求将数据加载到内存。Minor是指Minor Page Fault（次要页错误，简称MnPF），磁盘数据被加载到内存后，内核再次读取时，会发出一个MnPF信息。一个文件第一次被读写时会有很多的MPF，被缓存到内存后再次访问MPF就会很少，MnPF反而变多，这是内核为减少效率低下的磁盘I/O操作采用的缓存技术的结果。

如果ANR发生时发现CPU使用率中iowait占比很高，可以通过查看进程的major次数来推断是哪个进程在进行磁盘I/O操作。<!-- 求证一下 -->

#### 新增和移除的进程或线程

如果一个进程或线程的CPU使用率前有“+”，说明该进程或线程是在最后两次CPU使用率采样时间段内新建的；反之如果是“-”，说明该进程或线程在采样时间段内终止了；如果是空，说明该进程或线程是在倒数第二次采样时间点之前已经存在。