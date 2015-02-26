---
layout: post
title: "dalvik虚拟机与zygote && M0.0.7"
modified:
categories: 
excerpt: "从app_process开始说起"
tags: [android, app_process, zygote]
image:
  feature:
date: 2015-02-10T00:021:56+08:00
---
#####Base On Android 2.3.7

沉寂了一年，可以说这一年经历了很多，想了想还是把旧网站的一部分博客转了过来，同时也开始了学习cpp的道路。

既然学了cpp，就从c++的角度重新看看android吧。

这次说说android系统的启动以及app的启动，同时阐述这段时间各层都发生了什么，先看看调试信息
<figure>
	<a href="/images/2015/02/01.png"><img src="/images/2015/02/01.png"></a>
</figure>

首先是系统启动脚本system/core/rootdir/init.rc文件中

{% highlight cpp %}
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server
    socket zygote stream 666
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
{% endhighlight %}

接着就转到frameworks/base/cmds/app_process/app_main.cpp的main函数

{% highlight cpp %}
int main(int argc, const char* const argv[])
{
    // These are global variables in ProcessState.cpp
    mArgC = argc;
    mArgV = argv;
    
    mArgLen = 0;
    for (int i=0; i<argc; i++) {
        mArgLen += strlen(argv[i]) + 1;
    }
    mArgLen--;

    AppRuntime runtime;
    const char *arg;
    const char *argv0;

    argv0 = argv[0];

    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm
    
    int i = runtime.addVmArguments(argc, argv);

    // Next arg is parent directory
    if (i < argc) {
        runtime.mParentDir = argv[i++];
    }

    // Next arg is startup classname or "--zygote"
    if (i < argc) {
        arg = argv[i++];
        if (0 == strcmp("--zygote", arg)) {
            bool startSystemServer = (i < argc) ? 
                    strcmp(argv[i], "--start-system-server") == 0 : false;
            setArgv0(argv0, "zygote");
            set_process_name("zygote");
            runtime.start("com.android.internal.os.ZygoteInit",
                startSystemServer); //runtime.start方法启动，传入参数ZygoteInit
        } else {
            set_process_name(argv0);

            runtime.mClassName = arg;

            // Remainder of args get passed to startup class main()
            runtime.mArgC = argc-i;
            runtime.mArgV = argv+i;

            LOGV("App process is starting with pid=%d, class=%s.\n",
                 getpid(), runtime.getClassName());
            runtime.start();
        }
    } else {
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        return 10;
    }

}
{% endhighlight %}

接下来跟踪到AndroidRuntime.start，这个函数定义在frameworks/base/core/jni/AndroidRuntime.cpp

{% highlight cpp %}

/*
 * Start the Android runtime.  This involves starting the virtual machine
 * and calling the "static void main(String[] args)" method in the class
 * named by "className".
 */
void AndroidRuntime::start(const char* className, const bool startSystemServer)
{
    LOGD("\n>>>>>> AndroidRuntime START %s <<<<<<\n",
            className != NULL ? className : "(unknown)");

    char* slashClassName = NULL;
    char* cp;
    JNIEnv* env;

    blockSigpipe();

    /* 
     * 'startSystemServer == true' means runtime is obslete and not run from 
     * init.rc anymore, so we print out the boot start event here.
     */
    if (startSystemServer) {
        /* track our progress through the boot sequence */
        const int LOG_BOOT_PROGRESS_START = 3000;
        LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START, 
                       ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
    }

    const char* rootDir = getenv("ANDROID_ROOT");
    if (rootDir == NULL) {
        rootDir = "/system";
        if (!hasDir("/system")) {
            LOG_FATAL("No root directory specified, and /android does not exist.");
            goto bail;
        }
        setenv("ANDROID_ROOT", rootDir, 1);
    }

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //LOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    /* start the virtual machine */
    if (startVm(&mJavaVM, &env) != 0) //虚拟机启动
        goto bail;

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) { //注册jni
        LOGE("Unable to register all android natives\n");
        goto bail;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we only have one argument, the class name.  Create an
     * array to hold it.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;
    jstring startSystemServerStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(2, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);
    startSystemServerStr = env->NewStringUTF(startSystemServer ? 
                                                 "true" : "false");
    env->SetObjectArrayElement(strArray, 1, startSystemServerStr);

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    jclass startClass;
    jmethodID startMeth;

    slashClassName = strdup(className);
    for (cp = slashClassName; *cp != '\0'; cp++)
        if (*cp == '.')
            *cp = '/';

    startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        LOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            LOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray); //启动传入的参数方法，也就是ZygoteInit类

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }

    LOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        LOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        LOGW("Warning: VM did not shut down cleanly\n");

bail:
    free(slashClassName);
}

{% endhighlight %}
        
这个函数的作用是启动Android系统运行时库，它主要做了三件事情，一是调用函数startVM启动虚拟机，二是调用函数startReg注册JNI方法，三是调用了com.android.internal.os.ZygoteInit类的main函数。[^1]

然后ZygoteInit启动了

{% highlight java %}
public static void main(String argv[]) {
    try {
        VMRuntime.getRuntime().setMinimumHeapSize(5 * 1024 * 1024);

        // Start profiling the zygote initialization.
        SamplingProfilerIntegration.start();

        registerZygoteSocket(); //注册套接字，接收新app启动信息
        EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
            SystemClock.uptimeMillis());
        preloadClasses(); //预加载类信息
        //cacheRegisterMaps();
        preloadResources(); //预加载资源、
        EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
            SystemClock.uptimeMillis());

        // Finish profiling the zygote initialization.
        SamplingProfilerIntegration.writeZygoteSnapshot();

        // Do an initial gc to clean up after startup
        gc();

        // If requested, start system server directly from Zygote
        if (argv.length != 2) {
            throw new RuntimeException(argv[0] + USAGE_STRING);
        }

        if (argv[1].equals("true")) {
            startSystemServer(); //启动SystemServer
        } else if (!argv[1].equals("false")) {
            throw new RuntimeException(argv[0] + USAGE_STRING);
        }

        Log.i(TAG, "Accepting command socket connections");

        if (ZYGOTE_FORK_MODE) { //zygote的关键，当有app启动时，会fork出一份拷贝，否则一直loop
            runForkMode();
        } else {
            runSelectLoopMode();
        }

        closeServerSocket();
    } catch (MethodAndArgsCaller caller) {
        caller.run();
    } catch (RuntimeException ex) {
        Log.e(TAG, "Zygote died with exception", ex);
        closeServerSocket();
        throw ex;
    }
}
{% endhighlight %}
上面的过程可以归纳为下图
<figure>
	<a href="/images/2015/02/02.png"><img src="/images/2015/02/02.png"></a>
</figure>

接下来的代码就不贴了，接下来顺序是->runSelectLoopMode()->runOnce() [ZygoteConnection.java]->Zygote.forkAndSpecialize(...)
如下图所示
<figure>
	<a href="/images/2015/02/03.png"><img src="/images/2015/02/03.png"></a>
</figure>
ActivityManagerService管理android应用创建的新进程。ActivityManagerService启动新的进程是从其成员函数startProcessLocked开始.
ActivityManagerService.startProcessLocked定义在/frameworks/base/services/java/com/android/server/am/ActivityManagerService.java
{% highlight java %}
...
 int pid = Process.start("android.app.ActivityThread",
                    mSimpleProcessManagement ? app.processName : null, uid, uid,
                    gids, debugFlags, null);
...
{% endhighlight %}

process.start()根据是否支持多进程会通过zygote调用到RuntimeInit.invokeStaticMain()或者是直接调用process.invokeStaticMain(),走到调用ActivityThread.main()函数，这样一个app就启动起来了。[^2]
{% highlight java %}
...
cl.getMethod("main", new Class[] { String[].class })
                    .invoke(null, args);            
...
{% endhighlight %}

补充一点，Dalvik虚拟机的Zygote并不是提供一个运行时容器，它提供的只是一个用于共享的进程，所有的应用程序运行，都是独立的，OS级别的进程，直接受到OS层面的资源控制以及调度的影响，只是他们共享Zygote说预加载的类而已。这也就是我为什么说，Dalvik就像是给每个应用程序在底层加了个套子，应该属于进程虚拟。[^3]

当一个app启动起来的流程应该是下面这样的
<figure>
	<a href="/images/2015/02/04.png"><img src="/images/2015/02/04.png"></a>
</figure>
<figure>
	<a href="/images/2015/02/05.png"><img src="/images/2015/02/05.png"></a>
</figure>

#####To be Continue…

原创文章，转载请注明： 转载自 <a href="http://archcodev.com">:-X archcodev</a>

###参考
[^1]: <http://blog.csdn.net/luoshengyang/article/details/8914953>
[^2]: <http://www.cnblogs.com/coding-hundredOfYears/archive/2012/10/28/2742026.html>
[^3]: <http://www.zhihu.com/question/20207106>