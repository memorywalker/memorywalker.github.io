---
title: Android Service
date: 2022-02-09 09:25:49
tags: android; service
---

### Service

 [Services overview  | Android Developers (google.cn)](https://developer.android.google.cn/guide/components/services) 

一个应用程序组件，没有界面，即使切换到其他程序还可以长期在后台运行。一个组件可以和一个服务绑定后交互，甚至可以进程间通信。服务可以在后台处理网络通信，播放音乐，文件读写或者与content provider交互。

服务运行在当前进程的主线程中，除非指定，否则服务不会创建自己的线程也不会运行在独立的进程中，因此服务中执行任何阻塞操作需要在单独的线程中执行，避免阻塞主线程导致ANR。

考虑使用WorkManager来代替Service的功能

#### 分类

**前端服务**：显示在通知栏上的服务，用户可以明确知道当前有这个服务在运行，例如音乐播放时，通知栏显示

**后端服务**：后台服务，用户不会感知到在执行，例如下载文件

**绑定服务**：当一个应用组件通过bindService()绑定到这个服务，服务给组件提供C/S模式的交互，也可以进程间通信。绑定服务只在一个组件与他绑定后才会运行，当多个组件和一个服务绑定，只有当所有的组件都解绑后，服务才会销毁。

服务作为一个组件需要在manifest文件中声明，也可声明为私有，这样别的应用程序不能使用。可以在声明中增加`android:description`属性提供一个服务的说明，用户可以看到这个服务的作用。

安全考虑使用一个显式的Intent来启动服务，不要给服务声明intent filter。

#### 生命周期

由于用户可能看不到服务的运行状态，所以服务的生命周期管理十分重要，避免没有被销毁。

**启动服务**：一个组件通过调用startService()运行起来，通过参数Intent将信息传递给服务，服务自己调用stopSelf()或其他组件调用stopService()。启动这个服务的组件即使销毁了，服务还是运行状态。另一个组件可以停止其他组件启动的服务。一个服务可以启动多次，如果服务已经是运行状态，那么startService()执行后会调用onStartCommand()，而不再调用onCreate()

**绑定服务**：其他组件通过调用bindService()运行起来，客户端通过IBinder接口与服务交互。客户端通过调用unbindService()结束连接。服务不需要自己结束。

对于一个启动服务，其他组件还可以bind到这个服务上，此时调用stopService()或stopSelf()并不会结束服务，直到所有绑定的客户端unbind。例如通过启动服务开始播放音乐，其他组件可以通过绑定到这个服务获取当前播放的歌曲信息。

**停止一个服务**，当一个服务有多个并行启动的请求时，多个请求都会执行onStartCommand()，如果有一个触发停止，可能会导致新启动服务被停止掉，因此可以在stopSelf(int)中传入对应请求onStartCommand()的startId，在stopSelf()中判断如果id不是当前最新的id，就不能停止。

系统在内存很少时会结束后台运行的服务，如果服务与用户当前交互的界面绑定，不太会被销毁；如果一个服务声明为前端服务，几乎不会被自动销毁；系统销毁一个服务后，当资源满足后，还会把服务运行起来，此时会执行onStartCommand()接口。根据onStartCommand()的返回值`START_NOT_STICKY`/`START_STICKY`/`START_REDELIVER_INTENT`，系统会决定重启服务时传入的Intent的方式。

![android](/uploads/android/service_lifecycle.png)

#### 基本接口

 onStartCommand()  组件调用startService()启动服务时会回调这个接口，只要有调用这个接口，就需要手动调用stopService()来释放

onBind() 组件通过调用bindService()与服务绑定会回调这个接口，这个接口需要返回一个IBinder接口，用来实现客户端与服务的交互。如果不希望被绑定，返回null。

onCreate() 只会在服务初始化调用一次，如果服务已经运行，不会被回调。例如绑定一个已经启动服务，不会回调这个接口。可以在这里创建线程

onDestroy() 系统销毁服务回调，可以用来释放创建的资源例如线程。

#### 举例

```java
public class HelloService extends Service {
  private Looper serviceLooper;
  private ServiceHandler serviceHandler;

  // Handler that receives messages from the thread
  private final class ServiceHandler extends Handler {
      public ServiceHandler(Looper looper) {
          super(looper);
      }
      @Override
      public void handleMessage(Message msg) {
          // Normally we would do some work here, like download a file.
          // For our sample, we just sleep for 5 seconds.
          try {
              Thread.sleep(5000);
          } catch (InterruptedException e) {
              // Restore interrupt status.
              Thread.currentThread().interrupt();
          }
          // Stop the service using the startId, so that we don't stop
          // the service in the middle of handling another job
          stopSelf(msg.arg1);
      }
  }

  @Override
  public void onCreate() {
    // Start up the thread running the service. Note that we create a
    // separate thread because the service normally runs in the process's
    // main thread, which we don't want to block. We also make it
    // background priority so CPU-intensive work doesn't disrupt our UI.
    HandlerThread thread = new HandlerThread("ServiceStartArguments",
            Process.THREAD_PRIORITY_BACKGROUND);
    thread.start();

    // Get the HandlerThread's Looper and use it for our Handler
    serviceLooper = thread.getLooper();
    serviceHandler = new ServiceHandler(serviceLooper);
  }

  @Override
  public int onStartCommand(Intent intent, int flags, int startId) {
      Toast.makeText(this, "service starting", Toast.LENGTH_SHORT).show();

      // For each start request, send a message to start a job and deliver the
      // start ID so we know which request we're stopping when we finish the job
      Message msg = serviceHandler.obtainMessage();
      msg.arg1 = startId;
      serviceHandler.sendMessage(msg);

      // If we get killed, after returning from here, restart
      return START_STICKY;
  }

  @Override
  public IBinder onBind(Intent intent) {
      // We don't provide binding, so return null
      return null;
  }

  @Override
  public void onDestroy() {
    Toast.makeText(this, "service done", Toast.LENGTH_SHORT).show();
  }
}

// Start Sevice
Intent intent = new Intent(this, HelloService.class);
startService(intent);
```

#### 前端服务

#### 后台服务

#### 绑定服务

#### AIDL