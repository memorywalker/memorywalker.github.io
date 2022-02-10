---
title: Android Service
date: 2022-02-09 09:25:49
categories:
- android
tags:
- android
- service
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

前端服务用于当用户不需要与应用直接交互，但是又需要知道应用当前的运行状态的场景。前端服务会固定显示通知栏通知，直到服务结束。例如音乐播放器切换到后台后，波形音乐信息可以用前端服务在状态栏显示，一个跑步应用可以实时显示跑步距离。

##### 配置

API level 28 anroid 9 必须声明`FOREGROUND_SERVICE`

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android" ...>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <application ...>
        ...
    </application>
</manifest>
```

##### 前端服务周期

1. 启动一个服务

   ```java
   Context context = getApplicationContext();
   Intent intent = new Intent(...); // Build the intent for the service
   context.startForegroundService(intent);
   ```

2. 在服务的 onStartCommand 接口中调用 [startForeground](https://developer.android.google.cn/reference/android/app/Service#startForeground(int, android.app.Notification)) 让服务在前端运行

   ```java
   Intent notificationIntent = new Intent(this, ExampleActivity.class);
   PendingIntent pendingIntent =
           PendingIntent.getActivity(this, 0, notificationIntent, 0);
   
   Notification notification =
             new Notification.Builder(this, CHANNEL_DEFAULT_IMPORTANCE)
       .setContentTitle(getText(R.string.notification_title))
       .setContentText(getText(R.string.notification_message))
       .setSmallIcon(R.drawable.icon)
       .setContentIntent(pendingIntent)
       .setTicker(getText(R.string.ticker_text))
       .build();
   
   // Notification ID cannot be 0.
   startForeground(ONGOING_NOTIFICATION_ID, notification);
   ```

3. 移除前端服务 使用 [`stopForeground`](https://developer.android.google.cn/reference/android/app/Service#stopForeground(boolean)) 传入boolean变量决定是否同时删除通知栏显示，这个方法执行后，服务还是运行状态。也可以停止服务来结束服务运行，通知栏会自动删除。

##### 声明前端服务类型

声明前端服务的类型，可以让前端服务访问位置，摄像头和麦克风信息

1. 配置文件中需要增加配置

   ```xml
   <manifest>
       ...
       <service ...
           android:foregroundServiceType="location|camera|microphone" />
   </manifest>
   ```

2. 启动服务时指明需要哪些权限

   ```java
   Notification notification = ...;
   Service.startForeground(notification,
           FOREGROUND_SERVICE_TYPE_LOCATION | FOREGROUND_SERVICE_TYPE_CAMERA);
   ```

3. 当应用在后台运行时，前端服务使用的这些权限会有限制，此时不能访问麦克风和摄像头，只有当用户授权了 [`ACCESS_BACKGROUND_LOCATION`](https://developer.android.google.cn/reference/android/Manifest.permission#ACCESS_BACKGROUND_LOCATION) 权限后，才能访问位置信息。当然还有一些特殊情况可以去掉这种[限制](https://developer.android.google.cn/guide/components/foreground-services#bg-access-restriction-exemptions)。

##### 通知栏

以下几种前端服务会立即显示到通知栏：

* The service is associated with a notification that includes [action buttons](https://developer.android.google.cn/training/notify-user/build-notification#Actions).
* The service has a [`foregroundServiceType`](https://developer.android.google.cn/guide/topics/manifest/service-element#foregroundservicetype) of `mediaPlayback`, `mediaProjection`, or `phoneCall`.
* The service provides a use case related to phone calls, navigation, or media playback, as defined in the notification's [category attribute](https://developer.android.google.cn/reference/android/app/Notification#category).
* The service has opted out of the behavior change by passing `FOREGROUND_SERVICE_IMMEDIATE` into [`setForegroundServiceBehavior()`](https://developer.android.google.cn/reference/android/app/Notification.Builder#setForegroundServiceBehavior(int)) when setting up the notification.



#### 绑定服务

绑定服务是一种客户端-服务端模式的服务，当一个组件例如activity绑定了一个服务，activity作为客户端可以向服务发送请求。同时不同进程间可以使用绑定服务实现IPC。

可以同时实现 [onBind()](https://developer.android.google.cn/reference/android/app/Service#onBind(android.content.Intent)) 和 [onStartCommand()](https://developer.android.google.cn/reference/android/app/Service#onStartCommand(android.content.Intent, int, int)) 两个接口，这样一个服务可以正常启动后，再被别的组件绑定。例如用户从一个音乐播放器程序的activity启动了服务进行音乐播放，在用户把音乐程序切换后台后，再切换回来，这个activity可以绑定之前服务，对音乐进行控制。

##### 服务端

当有一个客户端绑定服务后，系统会回调服务的[onBind()](https://developer.android.google.cn/reference/android/app/Service#onBind(android.content.Intent)) 接口，这个接口返回一个`IBinder`对象供客户端访问服务的公共接口。当有多个客户端绑定服务时，只有**第一个绑定**时会回调`onBind`，后面的绑定都复用缓存的同一个`IBinder`接口对象。

如果服务端在`onUnBind()`中返回`true`，那么下次有客户端再绑定服务时，会回调服务的`onRebind`接口。

###### IBinder接口对象

有三种方式提供IBinder接口：

* 提供Binder的子类

  如果服务只是给应用内部使用，且不需要进程间通信，返回一个继承Binder类的对象来提供服务的公共接口最合适。

* 使用Messenger

  如果服务需要在不同进程间通信，由于不同进程间不能获取对方接口信息，所以不能直接调用Binder对象的方法。这时需要使用Messenger，通过消息的方式给服务发送请求。服务中定义一个Handler来处理客户端请求的Message。

  Messenger内部会把所有的客户端请求Message放在一个线程的队列中通知给服务，这样服务中不需要考虑多线程问题。

* 使用AIDL

   Android Interface Definition Language (AIDL)  可以将对象进行序列化后用于进程间的通信。Messenger本质上也是使用了AIDL，只是把所有的请求放在一个队列中执行。当服务需要同时处理多个客户端的请求时，可以使用AIDL的方式，此时需要服务端自己处理多线程。

##### 客户端

客户端通过调用 [bindService()](https://developer.android.google.cn/reference/android/content/Context#bindService(android.content.Intent, android.content.ServiceConnection, int)) 来绑定一个服务，绑定过程是异步的，bindService()会立即返回，客户端需要实现 [ServiceConnection](https://developer.android.google.cn/reference/android/content/ServiceConnection) 用来监控与服务的连接状态。

`bindService(new Intent(Binding.this, MessengerService.class), mConnection, Context.BIND_AUTO_CREATE)`

其中的`mConnection`在绑定成功后收到`onServiceConnected`回调，里面可以获得服务的`onBind`接口返回的`IBinder`对象。

客户端通过调用 `unbindService()` 与服务解绑，当客户端被销毁时，同时也会触发解绑，但是建议不需要服务的时候客户端主动解绑，释放服务资源。

##### 注意事项

* bind和unbind要成对出现。如果客户端只是在用户可见的时候与服务有交互，在`onStart`中绑定，`onStop`中解绑定
* 如果activity切换到后台后还有交互，在`onCreate`中绑定，`onDestory`中解绑定。这种方式activity在整个生命周期中都使用服务，如果服务在另一个进程中运行，这样会增加服务进程的权重，系统更可能杀死这个进程。
* 对象的引用计数会跨进程累计
* 连接发生异常时，会抛出 [DeadObjectException](https://developer.android.google.cn/reference/android/os/DeadObjectException) 

##### 实现Binder类的步骤

1. 服务类中创建一个**Binder**类的实例，这个类提供：
   * 客户端可以调用的公共方法
   * 返回当前的Service类的实例，客户端可以通过这个实例访问服务的公共方法
   * 返回服务中定义的其他类的实例，客户端可以访问这些类的公共方法
2. 服务的onBind()方法返回定义的**Binder**类的实例
3. 客户端在 [onServiceConnected()](https://developer.android.google.cn/reference/android/content/ServiceConnection#onServiceConnected(android.content.ComponentName, android.os.IBinder)) 中获取Binder类对象，并调用其提供的接口。

###### 服务端举例

```java
public class LocalService extends Service {
    // Binder given to clients
    private final IBinder binder = new LocalBinder();
    // Random number generator
    private final Random mGenerator = new Random();

    /**
     * Class used for the client Binder.  Because we know this service always
     * runs in the same process as its clients, we don't need to deal with IPC.
     */
    public class LocalBinder extends Binder {
        LocalService getService() {
            // Return this instance of LocalService so clients can call public methods
            return LocalService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    /** method for clients */
    public int getRandomNumber() {
      return mGenerator.nextInt(100);
    }
}
```

###### 客户端举例

```java
public class BindingActivity extends Activity {
    LocalService mService;
    boolean mBound = false;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
    }

    @Override
    protected void onStart() {
        super.onStart();
        // Bind to LocalService
        Intent intent = new Intent(this, LocalService.class);
        bindService(intent, connection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onStop() {
        super.onStop();
        unbindService(connection);
        mBound = false;
    }

    /** Called when a button is clicked (the button in the layout file attaches to
      * this method with the android:onClick attribute) */
    public void onButtonClick(View v) {
        if (mBound) {
            // Call a method from the LocalService.
            // However, if this call were something that might hang, then this request should
            // occur in a separate thread to avoid slowing down the activity performance.
            int num = mService.getRandomNumber();
            Toast.makeText(this, "number: " + num, Toast.LENGTH_SHORT).show();
        }
    }

    /** Defines callbacks for service binding, passed to bindService() */
    private ServiceConnection connection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName className,
                IBinder service) {
            // We've bound to LocalService, cast the IBinder and get LocalService instance
            LocalBinder binder = (LocalBinder) service;
            mService = binder.getService();
            mBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName arg0) {
            mBound = false;
        }
    };
}
```



##### 实现Messenger的步骤

1. 服务实现 [Handler](https://developer.android.google.cn/reference/android/os/Handler) 用来处理客户端发来的请求
2. 服务使用 [Handler](https://developer.android.google.cn/reference/android/os/Handler) 创建一个`Messenger`对象，`Messager`对象中有这个`Handler`的一个引用
3. `Messenger`创建一个`IBinder`用来在`onBind`中返回给客户端
4. 客户端使用`IBinder`对象获得`Messenger`对象，客户端使用`Messenger`对象给服务发送`Message`对象
5. 服务在 [Handler](https://developer.android.google.cn/reference/android/os/Handler) 的`handleMessage()`中处理客户端发来的`Message`
6. 客户端中也可以像服务端一样创建一个`Messenger`对象，在发送消息时，把自己的`Messenger`对象作为`Message`的`replyTo`参数，这样服务收到消息后，可以使用客户端的`Messenger`对象给客户端回消息。

###### 客户端举例

```java
public class MessengerServiceActivities {
// BEGIN_INCLUDE(bind)
    /**
     * Example of binding and unbinding to the remote service.
     * This demonstrates the implementation of a service which the client will
     * bind to, interacting with it through an aidl interface.
     * 
     * Note that this is implemented as an inner class only keep the sample
     * all together; typically this code would appear in some separate class.
     */
    public static class Binding extends Activity {
        /** Messenger for communicating with service. */
        Messenger mService = null;
        /** Flag indicating whether we have called bind on the service. */
        boolean mIsBound;
        /** Some text view we are using to show state information. */
        TextView mCallbackText;
        
        /**
         * Handler of incoming messages from service.
         */
        class IncomingHandler extends Handler {
            @Override
            public void handleMessage(Message msg) {
                switch (msg.what) {
                    case MessengerService.MSG_SET_VALUE:
                        mCallbackText.setText("Received from service: " + msg.arg1);
                        break;
                    default:
                        super.handleMessage(msg);
                }
            }
        }
        
        /**
         * Target we publish for clients to send messages to IncomingHandler.
         * 通过消息把这个对象发送到服务，服务再利用这个对象给客户端回消息
         */
        final Messenger mMessenger = new Messenger(new IncomingHandler());
        
        /**
         * Class for interacting with the main interface of the service.
         */
        private ServiceConnection mConnection = new ServiceConnection() {
            public void onServiceConnected(ComponentName className,
                    IBinder service) {
                // This is called when the connection with the service has been
                // established, giving us the service object we can use to
                // interact with the service.  We are communicating with our
                // service through an IDL interface, so get a client-side
                // representation of that from the raw service object.
                mService = new Messenger(service); // 得到服务端的Messenger，用来给服务发消息
                mCallbackText.setText("Attached.");

                // We want to monitor the service for as long as we are
                // connected to it.
                try {
                    Message msg = Message.obtain(null,
                            MessengerService.MSG_REGISTER_CLIENT);
                    // 把自己的Messenger发给服务，好让服务可以给客户端回消息
                    msg.replyTo = mMessenger;
                    mService.send(msg);
                    
                    // Give it some value as an example.
                    msg = Message.obtain(null,
                            MessengerService.MSG_SET_VALUE, this.hashCode(), 0);
                    mService.send(msg);
                } catch (RemoteException e) {
                    // In this case the service has crashed before we could even
                    // do anything with it; we can count on soon being
                    // disconnected (and then reconnected if it can be restarted)
                    // so there is no need to do anything here.
                }
                
                // As part of the sample, tell the user what happened.
                Toast.makeText(Binding.this, R.string.remote_service_connected,
                        Toast.LENGTH_SHORT).show();
            }

            public void onServiceDisconnected(ComponentName className) {
                // This is called when the connection with the service has been
                // unexpectedly disconnected -- that is, its process crashed.
                mService = null;
                mCallbackText.setText("Disconnected.");

                // As part of the sample, tell the user what happened.
                Toast.makeText(Binding.this, R.string.remote_service_disconnected,
                        Toast.LENGTH_SHORT).show();
            }
        };
        
        void doBindService() {
            // Establish a connection with the service.  We use an explicit
            // class name because there is no reason to be able to let other
            // applications replace our component.
            bindService(new Intent(Binding.this, 
                    MessengerService.class), mConnection, Context.BIND_AUTO_CREATE);
            mIsBound = true;
            mCallbackText.setText("Binding.");
        }
        
        void doUnbindService() {
            if (mIsBound) {
                // If we have received the service, and hence registered with
                // it, then now is the time to unregister.
                if (mService != null) {
                    try {
                        // 解绑的时候，通知服务也取消注册当前客户端的Messenger实例
                        Message msg = Message.obtain(null,
                                MessengerService.MSG_UNREGISTER_CLIENT);
                        msg.replyTo = mMessenger;
                        mService.send(msg);
                    } catch (RemoteException e) {
                        // There is nothing special we need to do if the service
                        // has crashed.
                    }
                }
                
                // Detach our existing connection.
                unbindService(mConnection);
                mIsBound = false;
                mCallbackText.setText("Unbinding.");
            }
        }
 // END_INCLUDE(bind)
        
        /**
         * Standard initialization of this activity.  Set up the UI, then wait
         * for the user to poke it before doing anything.
         */
        @Override
        protected void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);

            setContentView(R.layout.messenger_service_binding);

            // Watch for button clicks.
            Button button = (Button)findViewById(R.id.bind);
            button.setOnClickListener(mBindListener);
            button = (Button)findViewById(R.id.unbind);
            button.setOnClickListener(mUnbindListener);
            
            mCallbackText = (TextView)findViewById(R.id.callback);
            mCallbackText.setText("Not attached.");
        }

        private OnClickListener mBindListener = new OnClickListener() {
            public void onClick(View v) {
                doBindService();
            }
        };

        private OnClickListener mUnbindListener = new OnClickListener() {
            public void onClick(View v) {
                doUnbindService();
            }
        };
    }
}
```



###### 服务端举例

```java
//BEGIN_INCLUDE(service)
public class MessengerService extends Service {
    /** For showing and hiding our notification. */
    NotificationManager mNM;
    /** Keeps track of all current registered clients. */
    ArrayList<Messenger> mClients = new ArrayList<Messenger>();
    /** Holds last value set by a client. */
    int mValue = 0;
    
    /**
     * Command to the service to register a client, receiving callbacks
     * from the service.  The Message's replyTo field must be a Messenger of
     * the client where callbacks should be sent.
     */
    static final int MSG_REGISTER_CLIENT = 1;
    
    /**
     * Command to the service to unregister a client, ot stop receiving callbacks
     * from the service.  The Message's replyTo field must be a Messenger of
     * the client as previously given with MSG_REGISTER_CLIENT.
     */
    static final int MSG_UNREGISTER_CLIENT = 2;
    
    /**
     * Command to service to set a new value.  This can be sent to the
     * service to supply a new value, and will be sent by the service to
     * any registered clients with the new value.
     */
    static final int MSG_SET_VALUE = 3;
    
    /**
     * Handler of incoming messages from clients.
     */
    class IncomingHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_REGISTER_CLIENT:
                    // 注册一个客户端Messenger，用来给对应的客户端应答Message
                    mClients.add(msg.replyTo);
                    break;
                case MSG_UNREGISTER_CLIENT:
                    mClients.remove(msg.replyTo);
                    break;
                case MSG_SET_VALUE:
                    mValue = msg.arg1;
                    for (int i=mClients.size()-1; i>=0; i--) {
                        try {
                            mClients.get(i).send(Message.obtain(null,
                                    MSG_SET_VALUE, mValue, 0));
                        } catch (RemoteException e) {
                            // The client is dead.  Remove it from the list;
                            // we are going through the list from back to front
                            // so this is safe to do inside the loop.
                            mClients.remove(i);
                        }
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }
    
    /**
     * Target we publish for clients to send messages to IncomingHandler.
     * 提供给客户端使用的Messenger对象，客户使用它来发消息给服务
     */
    final Messenger mMessenger = new Messenger(new IncomingHandler());
    
    @Override
    public void onCreate() {
        mNM = (NotificationManager)getSystemService(NOTIFICATION_SERVICE);

        // Display a notification about us starting.
        showNotification();
    }

    @Override
    public void onDestroy() {
        // Cancel the persistent notification.
        mNM.cancel(R.string.remote_service_started);

        // Tell the user we stopped.
        Toast.makeText(this, R.string.remote_service_stopped, Toast.LENGTH_SHORT).show();
    }
    
    /**
     * When binding to the service, we return an interface to our messenger
     * for sending messages to the service.
     */
    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }

    /**
     * Show a notification while this service is running.
     */
    private void showNotification() {
        // In this sample, we'll use the same text for the ticker and the expanded notification
        CharSequence text = getText(R.string.remote_service_started);

        // The PendingIntent to launch our activity if the user selects this notification
        PendingIntent contentIntent = PendingIntent.getActivity(this, 0,
                new Intent(this, Controller.class), 0);

        // Set the info for the views that show in the notification panel.
        Notification notification = new Notification.Builder(this)
                .setSmallIcon(R.drawable.stat_sample)  // the status icon
                .setTicker(text)  // the status text
                .setWhen(System.currentTimeMillis())  // the time stamp
                .setContentTitle(getText(R.string.local_service_label))  // the label of the entry
                .setContentText(text)  // the contents of the entry
                .setContentIntent(contentIntent)  // The intent to send when the entry is clicked
                .build();

        // Send the notification.
        // We use a string id because it is a unique number.  We use it later to cancel.
        mNM.notify(R.string.remote_service_started, notification);
    }
}
//END_INCLUDE(service)
```



#### AIDL

一般不会用到