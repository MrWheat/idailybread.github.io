title: 入门篇　第二章 服务与广播接收者
date: 2014-11-12 22:40:14
categories: Android开发 - 倔强青铜
---
# 第一节 服务 #
　　服务（`Service`）运行在程序的后台，通常用来完成一些现不需要用户界面但是需要一直运行的功能，如`处理网络事务(消息推送)`、`播放音乐`等。

<br>　　此时就有疑问，既然Service用来执行耗时任务，那和开启一个`Thread`有什么区别？

	-  首先，Service最明显的一个优势是，提高进程的优先级。
	   -  当系统内存不足时，系统会从众多后台进程中选择优先级最低的进程，予以杀掉。
	   -  同时Android也规定了，内部没有Service的进程比有Service的进程要更优先被杀掉。
	-  然后，Service和Thread并不冲突。
	   -  Service的各个生命周期方法都是在主线程中调用的，如果它内部需要执行耗时操作，那么同样要开启Thread。

<br>　　简单的说：

	-  对于一些简单的、短暂的http请求等异步操作可以使用Thread类，因为此时不需要考虑进程被杀掉的风险。
	-  对于长时间运行的、重要的耗时操作，如音乐播放、消息推送、上传下载等操作，应该为该操作启动一个服务，而不是简单的创建一个工作线程，这样可以提升进程的优先级，降低当在系统内存不足时，它被杀掉的几率。

<br>　　不过，除了执行耗时操作外，`Service`另一个重要的功能则是`进程间通信`（IPC，后面章节中会详细介绍）。

## 基础入门 ##
<br>
### 普通Service ###
<br>　　你可以继承下面两个类之一来创建自己的Service类：

	-  Service：这是所有服务的基类，它的各个生命周期方法都是在应用程序的主线程中被调用。
	-  IntentService：Service类的子类，它使用工作线程（非UI线程）来依次处理启动请求。

<br>　　范例1：继承Service类。
``` java
package com.example.androidtest;
import android.app.Service;
import android.content.Intent;
import android.os.IBinder;
public class MyService extends Service {

    // 当Service创建时，系统会调用这个方法。在Service被销毁之前，此方法只会被调用一次。
    public void onCreate() {
        super.onCreate();
    }

    // 此方法有两个调用时机：
    // 第一：onCreate被调用后，会接着调用此方法。
    // 第二：当Service已经启动后，有人再次调用startService方法试图启动服务时，此方法会被调用，但不会调用onCreate。
    public int onStartCommand(Intent intent, int flags, int startId) {
        return super.onStartCommand(intent, flags, startId);
    }

    // 当Service销毁时，系统会调用这个方法。
    // 你应该使用这个方法来实现一些清理资源的工作，如清理线程等。
    public void onDestroy() {
        super.onDestroy();
    }

    // 当程序想通过调用bindService()方法跟这个Service绑定时，系统会调用这个方法，具体后述。
    // 你必须实现这个方法，因为它是抽象的，如果你不想处理绑定请求可以返回null。
    public IBinder onBind(Intent arg0) {
        return null;
    }
}
```
	语句解释：
	-  从onStartCommand方法中返回的值必须是以下常量，它们用来告诉系统当服务需要被杀死时，服务的重启策略：
	   - START_NOT_STICKY：直到再次接收到新的Intent对象，这个服务才会被重新创建。
	   - START_STICKY：立刻重启，但不会重新传递最后的Intent对象，而会用null来调用onStartCommand方法。
	   - START_REDELIVER_INTENT：立刻重启，会重新传递最后的Intent对象给onStartCommand方法。
	-  Service类的所有生命周期方法都是在主线程中被调用的，这意味着如果你需要执行耗时操作的话，就必须自己开启一个线程，然后在线程中去执行。

<br>　　范例2：继承IntentService类。
``` java
package com.example.androidtest;
import android.app.IntentService;
import android.content.Intent;
public class MyIntentService extends IntentService {
    public MyIntentService() {
        // 由于IntentService类没有提供无参构造器，因此其子类必须显示调用有参构造器。
        super("MyIntentService");
    }

    // 此方法将在子线程中被调用，因此不要在其内部执行更新UI的操作。
    protected void onHandleIntent(Intent arg0) {
        long endTime = System.currentTimeMillis() + 5 * 1000;
        while (System.currentTimeMillis() < endTime) {
            synchronized (this) {
                try {
                    wait(endTime - System.currentTimeMillis());
                } catch (Exception e) { }
            }
        }
    }
}
```

    语句解释：
    -  IntentService是Service类的子类，它内部会把所有接到的请求交给一个工作线程去执行，帮我们省去创建线程的操作。
    -  若接到新请求时IntentService并没有被启动，则会像普通Service那样，调用onCreate方法，且在onCreate方法中会创建一个队列，并开始处理新请求。
    -  若服务正在处理某个请求时，又接到新的请求，则新请求会被放到队列中等待，队列中的所有请求都按照先进先出的原则被依次处理。当队列中的所有请求都被处理完毕后，IntentService就会调用stopSelf(int)方法来终止自己。
    -  由于IntentService类的各个生命周期方法中，都对Service做了扩展，因此除非必要你不应该重写这些方法，重写时应该先调用父类的实现，同时你不应该重写onStartCommand方法。

<br>　　以上就是你做的全部：`一个构造器`和`一个onHandleIntent()方法的实现`。

<br>　　**何时使用?**
　　使用`IntentService`类实现服务的特点如下：

	-  任务按照先进先出顺序依次执行，同一时间无法执行一个以上的任务。
	-  任务一旦被安排后，无法撤销、终止。
　　因此`IntentService`类适合于对上述两种缺点无要求的应用场合。

<br>　　范例3：配置服务。
``` xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
	package="com.example.androidtest" android:versionCode="1" android:versionName="1.0">
    <application 
        android:icon="@drawable/ic_launcher" 
        android:label="@string/app_name">
            <Service android:name="com.example.androidtest.MyService"/>
    </application>
</manifest>
```

    语句解释：
    -  由于服务是Android四大组件之一，因此需要在<application>标签内配置，服务和Activity一样，可以指定意图过滤器。当需要启动某个服务时，Android系统同样会依次和已在系统中注册的服务进行匹配，匹配成功的服务将会被启动。

<br>　　范例4：有两种方法可以启动服务，分别是调用`startService()`和`bindService()`两个方法。

　　本节只介绍通过`startService()`方法启动服务的知识，它有如下几个特点：

	-  Service一旦启动，它就能够无限期的在后台运行，除非下面的两种情况发生：
	   -  第一，系统内存不足，需要杀死Service所在的进程。
	   -  第二，调用stopSelf方法或stopService方法来明确停止服务。
	-  通过startService方法启动的服务，会涉及到三个生命周期方法：
	   -  首先，服务对象创建完毕后服务的onCreate方法被调用，执行一些初始化操作。
	   -  然后，onCreate方法调用后，服务的onStartCommand方法被调用，开始运行服务。  
	   -  最后，当服务被销毁时，服务的onDestroy方法被调用，执行收尾工作。


<br>　　范例5：关于停止服务。

	-  当调用stopSelf方法或stopService方法请求终止服务后，系统会尽快的销毁这个服务。
	-  但是，如果你的服务正在处理一个请求的时候，又接受到一个新的启动请求，那么在第一个请求结束时终止服务有可能会导致第二个请求不被处理。
	-  要避免这个问题，你能够使用stopSelf(int)方法来确保你请求终止的服务始终是基于最后启动的请求。
	-  也就是说，调用stopSelf(int)方法时，你要把那个要终止的服务ID传递给这个方法（这个ID是系统发送给onStartCommand方法的参数）。
	-  这样如果服务在你调用stopSelf(int)方法之前收到了一个新的启动请求，那么这个ID就会因不匹配而不被终止。

<br>
### 绑定Service ###
　　服务允许应用程序组件通过调用`bindService()`方法来启动服务。

　　**问：为什么要使用此种方式?**
　　答：很多时候，服务用来完成一个耗时的操作，在服务运行的时候，用户可以通过Activity提供的界面控件，来调用服务中提供的方法，从而控制服务的执行。

　　但是，若通过`startService`方法启动服务，是无法返回该服务对象的引用，也就意味着没法调用服务中的方法。
　　通过绑定方式启动服务可以获取服务对象的引用。
　　因此，在你想要`Activity`以及应用程序中的其他组件跟服务进行交互（方法调用、数据交换）时，或者要把应用程序中的某些功能通过进程间通信（IPC）暴露给其他应用程序时，就需要创建一个绑定类型的服务。

<br>**生命周期方法**
　　绑定启动方式的服务会涉及到四个生命周期方法：
	-  首先，服务对象被创建后，服务的onCreate方法被调用，执行一些初始化操作。
	-  然后，在第一个访问者和服务建立连接时，会调用服务的onBind方法。 
	-  接着，当最后一个访问者被摧毁，服务的onUnbind方法被调用。
	   -  最好不要在普通的广播接收者中通过绑定方式启动服务，因为广播接收者生命周期短暂。
	-  最后，当服务被销毁时，服务的onDestroy方法被调用，执行收尾工作。

<br>　　一个组件（客户端）可以调用`bindService()`方法来与服务建立连接，调用`unbindService()`方法来关闭与服务连接。多个客户端能够绑定到同一个服务，并且当所有的客户端都解绑以后，系统就会自动销毁这个服务（服务不需要终止自己）。
　　
<center>
![图的左边显示了用startService()方法创建服务时的生命周期，图的右边显示了用bindService()方法创建服务时的生命周期。](/img/android/android_2_11.png)
</center>

<br>　　范例1：创建绑定服务。
``` java
public class MyService extends Service {

    // 1、onBind方法只会在第一个访问者和服务建立连接时会调用。
    // 2、onBind方法的返回值IBinder代表服务的一个通信管道。访问者通过IBinder对象来访问服务中的方法。
    // 3、onBind方法返回的IBinder对象会被传送给ServiceConnection接口的onServiceConnected方法。
    public IBinder onBind(Intent intent) {
        return new MyBinder();
    }

    // Binder类实现了IBinder接口
    public class MyBinder extends Binder {
        public MyService getService() {
            return MyService.this;
        }
    }

    public int add(int a, int b) {
        return a + b;
    }
}
```
    语句解释：
    -  创建具有绑定能力的服务时，必须提供一个IBinder对象，它用于给客户端提供与服务端进行交互的编程接口。

<br>　　范例2：创建客户端。
``` java
public class MainActivity extends Activity {

    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        Intent intent = new Intent(this,MyService.class);
        // 1、bindService方法用来通过绑定方式来启动Service。
        // 2、bindService方法的三个参数：
        //    intent：要启动的服务
        //    conn：代表一个访问者指向service的连接。当服务被启动或停止时，会导致连接的建立与断开。当连接建立
        //          或断开时系统就会调用ServiceConnection内定义的回调方法。
        //    flags：设置服务启动的模式。
        bindService(intent, conn, BIND_AUTO_CREATE);
    }

    ServiceConnection conn = new ServiceConnection() {

        // 系统会在连接的服务突然丢失的时候调用这个方法，如服务崩溃时或被杀死时。在客户端解除服务绑定时不会调用。
        public void onServiceDisconnected(ComponentName name) { }

        // 此方法当访问者和服务之间成功建立连接后由系统回调。
        public void onServiceConnected(ComponentName name, IBinder service) {
            // 由于MyService和MainActivity定义在同一个项目中，所以MainActivity中可以直接向下转型IBinder对象。
            MyBinder binder = (MyBinder) service;
            System.out.println(binder.getService().add(5, 5));
        }
    }

    protected void onStop() {
        super.onStop();
        // 手工的解除绑定指定的服务。
        unbindService(conn);
    }
}
```

    语句解释：
    -  bindService方法的flags参数通常取值（后两个常量的具体作用暂不知道，一般会用第一个）：
       -  Context.BIND_AUTO_CREATE ：若服务当前未启动，则新建立一个服务对象。
       -  Context.BIND_DEBUG_UNBIND
       -  Context.BIND_NOT_FOREGROUND
    -  提示：bindService方法是异步方法，调用完此方法后，主线程会立刻返回。因此，程序中操作服务对象的那部分代码（第26行），应该写在成功建立连接之后。
    -  客户端（MainActivity）获得到Service的引用后，就可以调用其所提供的public方法了。 

<br>**管理绑定类型服务的生命周期：**
　　当服务从所有的客户端解除绑定时，`Android`系统会销毁它（除非它还被`startService()`方法启动了）。因此如果是纯粹的绑定类型的服务，你不需要管理服务的生命周期（`Android`系统会基于是否有客户端绑定了这个服务来管理它）。
　　但是，若你使用了`startService()`方法启动这个服务，那么你就必须明确的终止这个服务，因为它会被系统认为是启动类型的。这样服务就会一直运行到服务用`stopSelf()`方法或其他组件调用`stopService()`方法来终止服务。
　　另外，如果你的服务是启动类型的并且也接收绑定，那么当系统调用`onUnbind()`方法时，如果你想要在下次客户端绑定这个服务时调用`onRebind()`方法，你可以选择返回`true`。
<center>
![绑定服务的生命周期](/img/android/android_2_12.png)
</center>

## 系统内置服务 ##
　　在Android系统中内置了许多系统服务，它们各自对应不同的功能。常见的服务有：通知管理器、窗口管理器、包管理器等。
<br>
### 包管理器 ###
　　包管理器(PackageManager)可以获取当前用户手机中已安装`APK`文件中的信息（如`AndroidManifest`文件）。

<br>　　范例1：获取基本信息。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 使用Context对象获取一个全局的PackageManager对象。
        PackageManager mgr = this.getPackageManager();
        PackageInfo info;
        try {
            // 为getPackageInfo()方法指定一个应用程序的包名，可以获取该应用程序AndriodManifest文件中的信息。
            // Context的getPackageName()方法可以获取当前应用的包名。
            // 因此下面这行代码的含义为：获取当前应用程序的AndriodManifest文件中的信息。
            info = mgr.getPackageInfo(this.getPackageName(), 0);
            System.out.println("当前应用程序的包名称：" + info.packageName);
            System.out.println("当前应用程序的版本号：" + info.versionCode);
            System.out.println("当前应用程序的版本名称：" + info.versionName);
        } catch (NameNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

    语句解释：
    -  PackageInfo类描述的是某个应用程序中的AndroidManifest文件中的信息。 
    -  PackageManager的getPackageInfo()方法的第二个参数用来指明获取哪些信息常用取值：
       -  0 ：获取基本信息。
       -  PackageManager.GET_SERVICES：获取基本信息+所有Service的信息。
       -  PackageManager.GET_ACTIVITIES：获取基本信息+所有Activity的信息。
       -  PackageManager.GET_PERMISSIONS：获取基本信息+所有自定义权限的信息。
       -  若需要获取其它类型的信息，请自行查阅API 。
    -  PackageInfo类的常用属性： 
       -  int packageName：获取PackageInfo所代表的应用程序的<manifest>标签的package属性的值。 
       -  String versionCode：获取PackageInfo所代表的应用程序的<manifest>标签的versionCode属性的值。
       -  String versionName：获取PackageInfo所代表的应用程序的<manifest>标签的versionName属性的值。
       -  PermissionInfo[] permissions：获取PackageInfo所代表的应用程序内所有的<permission>标签。
       -  ActivityInfo[] activities：获取PackageInfo所代表的应用程序内所有的<Activity>标签。
       -  若需要获取其它信息，请自行查阅API。

<br>　　范例2：获取所有的Activity。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        PackageManager mgr = this.getPackageManager();
        PackageInfo info;
        try {
            info = mgr.getPackageInfo(this.getPackageName(), PackageManager.GET_ACTIVITIES);
            System.out.println(info.activities);
        } catch (NameNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
    语句解释：
    -  若getPackageInfo方法的第二个参数设置为0，则调用info.activities获得的将是null。

<br>　　范例3：获取ActivityInfo的基本信息。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        PackageManager mgr = getPackageManager();
        PackageInfo info;
        try {
            // 当前应用程序的所有Activity信息。
            info = mgr.getPackageInfo(this.getPackageName(), PackageManager.GET_ACTIVITIES);
            for (ActivityInfo ainfo : info.activities) {
                System.out.println("Activity名称：" + ainfo.name);
                System.out.println("Activity图标的资源ID：" + ainfo.icon);
                System.out.println("Activity标题的资源ID：" + ainfo.labelRes);
                System.out.println("标题：" + getResources().getString(ainfo.labelRes));
            }

            // 系统gallery程序的所有Activity信息。
            info = mgr.getPackageInfo("com.android.gallery", PackageManager.GET_ACTIVITIES);
            for (ActivityInfo ainfo : info.activities) {
                System.out.println("Activity名称：" + ainfo.name);
                System.out.println("Activity图标的资源ID：" + ainfo.icon);
                System.out.println("Activity标题的资源ID：" + ainfo.labelRes);
                // 必须使用loadLabel()方法,才能导入其他应用程序的资源。
                System.out.println("标题：" + ainfo.loadLabel(mgr));
            }
        } catch (NameNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```
    语句解释：
    -  ActivityInfo类描述的是AndroidManifest文件中的<activity>标签。
    -  ActivityInfo类除了上面列出的name、icon、labelRes三个属性外还有其他属性，比较常用的还有一个launchMode属性，它用来指明Activity的启动模式：
       -  ActivityInfo.LAUNCH_MULTIPLE ：相当于standard模式。
       -  ActivityInfo.LAUNCH_SINGLE_TOP ：你懂的。
       -  ActivityInfo.LAUNCH_SINGLE_TASK ：你懂的。
       -  ActivityInfo.LAUNCH_SINGLE_INSTANCE ：你懂的。

<br>　　范例4：元数据。
　　在`AndroidManifest`文件里面为`Activity`、`BroadcastReceiver`、`Service`配置参数，这些参数被称为“元数据”。
``` xml
<activity android:name=".MainActivity" android:label="@string/app_name">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    <meta-data android:name="name" android:value="张三" />
    <meta-data android:name="age" android:value="20"/>
    <meta-data android:name="sex" android:value="@string/sex"/>
    <meta-data android:name="isStudent" android:value="false"/>
</activity>
```
    语句解释：
    -  使用<meta-data>标签为组件配置元数据，元数据是一个key/value。 
    -  使用android:name属性指出元数据的key，使用android:value属性指出元数据的value，其中元数据的value可以是一个常量，也可以从资源文件中引用。
    -  元数据value的类型可以是：boolean、int、String、float类型的。

<br>　　范例5：访问元数据。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        PackageManager mgr = this.getPackageManager();
        ComponentName component = new ComponentName(this, MainActivity.class);
        try {
            // 获取元数据。
            ActivityInfo activityInfo = mgr.getActivityInfo(component, PackageManager.GET_META_DATA);
            Bundle metaData = activityInfo.metaData;
            System.out.println(metaData.getString("name"));
            System.out.println(metaData.getInt("age")); 
            System.out.println(metaData.getString("sex"));
            System.out.println(metaData.getBoolean("isStudent"));
        } catch (NameNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

    语句解释：
    -  在程序运行的时候，通过ActivityInfo的metaData属性可以获取组件的元数据，元数据被保存在一个Bundle对象中。

<br>　　范例6：元数据 -- 图片资源。
``` c
// 假设在Activity标签内配置了这个元数据：
<meta-data android:name="img" android:value="@drawable/icon"/>

// 那么在程序中能够这么访问：
System.out.println(metaData.get("img"));

// 但是程序输出的结果将是：
res/drawable-mdpi/icon.png
```

    语句解释：
    -  直接给元数据的value赋图片资源的ID是不可以的，当程序运行时从Bundle对象中获取到的数据，是该图片资源所在的路径，这个路径是String类型的。

<br>　　范例7：resource属性。
``` java
// 在Activity标签内配置：
<meta-data android:name="img" android:resource="@drawable/icon"/>

// 那么在程序中能够这么访问：
ImageView img = new ImageView(this);
img.setImageResource(metaData.getInt("img"));
setContentView(img);
```

    语句解释：
    -  元数据的value也可以使用android:resource属性为其赋值，android:resource保存资源数据的ID。在程序运行时，使用Bundle对象的getInt方法可以获取该资源数据的ID。

<br>　　范例8：用户程序和系统程序。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        PackageManager mgr = this.getPackageManager();
        // 获取当前系统中已安装的应用程序的全部信息。
        List<ApplicationInfo> list = mgr.getInstalledApplications(PackageManager.GET_UNINSTALLED_PACKAGES);
        for (ApplicationInfo app : list) {
            if ((app.flags & ApplicationInfo.FLAG_UPDATED_SYSTEM_APP) != 0) {
                // 用户升级了原来的系统的程序。
                System.out.println(app.packageName + " 用户程序!");
            } else if ((app.flags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                // 用户自己安装的app。
                System.out.println(app.packageName + " 用户程序!");
            } else {
                System.out.println(app.packageName + " 系统内置程序!");
            }
        }
    }
}
```
    语句解释：
	-  ApplicationInfo类描述的是应用程序的<application>标签。
    -  ApplicationInfo的flags表示其所代表的Application标签的一个标记，通常用它来判断该应用程序是否是系统程序。

<br>　　PackageManager类还有一个比较常用的方法：
``` android
public abstract Intent getLaunchIntentForPackage (String packageName)
```
　　为它指定一个应用程序(包名)，它将返回一个能打开该包名对应的应用程序的入口`Activity`的`Intent`对象。
<br>
### 活动管理器 ###
　　本节内容主要是讲解`ActivityManager`的使用，通过它我们可以获得内存信息、进程信息，还可以终止某个进程。当然只能终止用户的进程，系统的进程是杀死不了的。
<br>　　ActivityManager常用的静态内部类如下：

	-  ActivityManager.MemoryInfo： 系统可用内存信息
	-  ActivityManager.RecentTaskInfo： 最近的任务信息
	-  ActivityManager.RunningAppProcessInfo： 正在运行的进程信息
	-  ActivityManager.RunningServiceInfo： 正在运行的服务信息
	-  ActivityManager.RunningTaskInfo： 正在运行的任务信息

<br>　　范例1：获取当前系统中所有Task。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // 调用Context的getSystemService()方法获取一个ActivityManager对象。
        ActivityManager manager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        // 从当前系统中，按照Task出现在前后台的顺序，获取maxNum个正在运行的Task。
        // 若当前系统中运行的Task少于100，则返回所有Task 。处于前台的Task将被放到List的第一个位置。
        // ActivityManager.RunningTaskInfo类描述一个正在运行的Task有关的信息。
        RunningTaskInfo stack = manager.getRunningTasks(100).get(0);
        System.out.println(stack.topActivity.getClassName());
        System.out.println(stack.topActivity.getShortClassName());
        System.out.println(stack.topActivity.getPackageName());
        System.out.println(stack.topActivity.toShortString());
        System.out.println(stack.topActivity);
    }
}
```
    语句解释：
    -  想要正确的运行本范例获取系统中Task的信息，你需要在<manifest>标签下面申请权限：
       -  <uses-permission android:name="android.permission.GET_TASKS" />
    -  RunningTaskInfo类常用属性
       -  baseActivity：当前Task的栈底Activity 。
       -  numActivities：当前Task中的Activity的个数。
       -  topActivity：当前Task的栈顶Activity 。

<br>　　范例2：获取当前系统的内存使用情况。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ActivityManager manager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        // 将当前系统内存信息保存到ActivityManager.MemoryInfo对象中。
        ActivityManager.MemoryInfo info = new ActivityManager.MemoryInfo();
        manager.getMemoryInfo(info);
        // 当前系统的可用内存。 当前系统中打开的程序越多其值将越小。
        System.out.println(formateFileSize(info.availMem));
        // 如果系统处于低内存状态下，此变量的值将为true。
        System.out.println(info.lowMemory);
        // 内存低的阀值。当availMem到达threshold以下时，系统进入低内存状态，并将会开始关闭一些后台Service以及进程。
        System.out.println(formateFileSize(info.threshold));
    }
	
    //调用系统函数，将一个long变量转换为String类型的表示形式：KB/MB。  
    private String formateFileSize(long size){  
        return Formatter.formatFileSize(MainActivity.this, size);   
    }  
}
```
    语句解释：
    -  android.text.format.Formatter类是提供的用于格式化各类文本的工具类。

<br>　　范例3：获取当前系统中的所有进程的信息。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ActivityManager manager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        // 获取当前系统中所有正在运行的进程。若无任何进程运行，则返回null 。此List中的元素是无序的。
        List<RunningAppProcessInfo> list = manager.getRunningAppProcesses();
        for (RunningAppProcessInfo p : list) {
            // 三个字段依次表示：进程的名称、进程的id、进程的重要性。
            System.out.println(p.processName +","+ p.pid + "," + p.importance);
        }
    }
}
```
    语句解释：
    -  RunningAppProcessInfo用来描述当前内存中的一个进程的信息。
    -  importance值越小进程的优先级越高，常用取值为：
       -  RunningAppProcessInfo.IMPORTANCE_FOREGROUND：100
       -  RunningAppProcessInfo.IMPORTANCE_VISIBLE：200
       -  RunningAppProcessInfo.IMPORTANCE_SERVICE：300
       -  RunningAppProcessInfo.IMPORTANCE_BACKGROUND：400
       -  RunningAppProcessInfo.IMPORTANCE_EMPTY：500
       -  RunningAppProcessInfo.IMPORTANCE_GONE：1000
    -  需要注意的是，不同Android版本的划分规则会有不同，比如8.1的源码中使用 IMPORTANCE_CACHED 来代替 IMPORTANCE_BACKGROUND 。

<br>　　问：进程的优先级有什么用呢？

	-  当系统内存低并且其它进程要求立即服务用户的时候，Android系统可以决定在某个时点关掉一个进程，运行在这个进程中的应用程序组件也会因进程被杀死而销毁。
	-  当要决定要杀死哪个进程时，Android系统会权衡它们对用户的重要性，即依赖于那个进程的优先级。
	-  通常情况下，按照优先级从高到底排列，系统将进程划分为五个级别：
	   -  前台进程、可见进程、服务进程、后台进程、空进程。

<br>**前台进程**
　　一个进程满足以下任何条件都被认为是前台进程：

	A.  进程持有一个正在跟用户交互的Activity对象（它的onResume方法已经被调用）。
	B.  进程持有一个正在前台运行的Service对象，这个Service已经调用了startForeground方法。
	C.  进程持有一个正在执行生命周期方法的Service对象（onCreate、onStart、或onDestroy）。
	D.  进程持有一个正在执行onReceive方法的BroadcastReceiver对象。
　　一般情况，在给定的时间内只有很少的前台进程存在，只有当设备已到达内存的饱和状态时，系统才会杀死一些前台进程来保持对用户界面的响应。

<br>**可见进程**
　　可见进程是指没有任何前台组件的一个进程，但是仍然能够影响用户在屏幕上看到的内容。如果满足下列条件之一，进程被认为是可见的：

	A.  进程持有一个不在前台的Activity，但是这个Activity依然对用户可见（它的onPause方法已经被调用）。
	B.  进程持有一个Service对象，该对象被一个正在跟用户交互的Activity所绑定。

<br>**服务进程**
　　服务进程是指持有了一个用`startService`方法启动的服务的进程，并且这个进程没有被归入上面介绍的两个分类。
　　虽然服务进程不直接跟用户看到的任何东西捆绑，但是，通常他们都在做用户关心的事情（如在后台播放音乐，或下载网络上的数据），除非前台和可见进程的内存不足，否则系统会一直保留它们。

<br>**后台进程**
　　后台进程是指持有一个用户当前不可见的`Activity`的进程（即`onStop`方法已经被调用）。
　　这些进程不会直接影响用户体验，并且为了给前台、可见、或服务进程回收内存，系统能够在任何时候杀死它们。
　　通常会有很多后台进程在运行，因此它们被保留在一个`LRU`（Least recently used）列表中以确保用户最近看到的那个带有`Activity`的进程最后被杀死。如果一个`Activity`正确的实现了它的生命周期方法，并且保存了当前的状态，那么杀死它的进程将不会影响用户的视觉体验，因为当用户返回到这个`Activity`时，这个`Activity`会恢复所有的可见状态。

<br>**空进程**
　　空进程是指不持有任何活动的应用程序组件的一个进程。保留这种进程存活的唯一原因是为了缓存的目的，以便提高需要在其中运行的组件的下次启动时间。
　　为了平衡进程缓存和基础内核缓存之间的整体系统资源，系统会经常杀死这些进程。

<br>特殊说明：

	-  Android会选择其中优先级最高的组件，来作为进程的级别。
	-  例如，一个进程持有一个Service和一个可见的Activity，那么这个进程就会被当作可见进程而不是服务进程。


<br>　　范例4：获取各个进程所占用的内存大小。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ActivityManager manager = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        List<RunningAppProcessInfo> list = manager.getRunningAppProcesses();
        for (RunningAppProcessInfo p : list) {
            int size = manager.getProcessMemoryInfo(new int[]{p.pid})[0].getTotalPrivateDirty() * 1024;
            System.out.println(p.processName +","+ Formatter.formatFileSize(this, size));
        }
    }
}
```
    语句解释：
    -  调用Formatter.formatFileSize来格式化显示字节数量。

<br>　　范例5：关闭进程。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ActivityManager am = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        List<RunningAppProcessInfo> runningAppProcessInfos = am.getRunningAppProcesses();
        if (runningAppProcessInfos == null || runningAppProcessInfos.size() == 0){
            for (RunningAppProcessInfo info : runningAppProcessInfos) {
                if (info.processName.equals("com.cutler.template")) {
                    android.os.Process.killProcess(info.pid);
                    am.killBackgroundProcesses(info.processName);
                }
            }
        }
    }
}
```
    语句解释：
    -  关闭名称为com.cutler.template的进程。
    -  执行本范例的代码需要在清单文件中申请如下两个权限：
       -  <uses-permission android:name="android.permission.KILL_BACKGROUND_PROCESSES" />
       -  <uses-permission android:name="android.permission.RESTART_PACKAGES"/>
<br>
### 传感器 ###
　　传感器是一种检测装置，能感受到被测量的信息，并能将检测感受到的信息，按一定规律变换成为电信号或其他所需形式的信息输出，以满足信息的传输、处理、存储、显示、记录和控制等要求。

　　目前，在Android中提供了6种传感器，这些传感器可以检测出手机当前所处的环境中的信息。

<br>　　范例1：android.hardware.Sensor类。

	-  方向传感器：Sensor.TYPE_ORIENTATION。
	-  重力传感器：Sensor.TYPE_ACCELEROMETER。
	-  光线传感器：Sensor.TYPE_LIGHT。
	-  磁场传感器：Sensor.TYPE_MAGNETIC_FIELD。
	-  距离传感器：Sensor.TYPE_PROXIMITY。
	-  温度传感器：Sensor.TYPE_TEMPERATURE。

<br>　　在`Sensor`类中没有提供`public`的构造器，若想构造一个`Sensor`对象，则可以使用系统提供的一个类`SensorManager`。

<br>　　范例1：获取方向传感器。
``` java
public class MainActivity extends Activity {
    SensorManager manager;
    Sensor orientationSensor;
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        // 获取SensorManager对象。
        manager = (SensorManager)getSystemService(Context.SENSOR_SERVICE);
        // manager.getDefaultSensor()方法指定传感器的类型，获取一个传感器对象。
        orientationSensor = manager.getDefaultSensor(Sensor.TYPE_ORIENTATION);
    }
}
```
<br>　　由于手机所处的环境是不断变化的，因此在代码中通常会定义一个监听器，当外界环境改变时`SensorManager`就会调用监听器中的方法。

<br>　　范例2：监听传感器的数据变化。
``` java
public class MainActivity extends Activity {
    SensorManager manager;
    Sensor orientationSensor;
    MySensorEventListener myListener;

    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        // 获取SensorManager对象。
        manager = (SensorManager)getSystemService(Context.SENSOR_SERVICE);
        // manager.getDefaultSensor()方法指定传感器的类型，获取一个传感器对象。
        orientationSensor = manager.getDefaultSensor(Sensor.TYPE_ORIENTATION);
        // MySensorEventListener的定义在下面，它是SensorEventListener的子类。
        myListener = new MySensorEventListener();
    }

    protected void onResume() {
        // 当Activity获得焦点的时候，注册一个事件监听器。
        manager.registerListener(myListener, orientationSensor, SensorManager.SENSOR_DELAY_NORMAL);
        super.onResume();
    }

    protected void onPause() {
        // 当Activity失去焦点时，解除注册。
        manager.unregisterListener(this.myListener);
        super.onPause();
    }

    private final class MySensorEventListener implements SensorEventListener {
        public void onAccuracyChanged(Sensor sensor, int accuracy) { }

        // 当传感器监听到的值改变时调用此方法。
        public void onSensorChanged(SensorEvent event) {
            float currentDegree = -event.values[0]; // 当前角度。
        }
    }
}
```

    语句解释：
    -  在SensorManager.registerListener()方法的第三个参数用来指明传感器每秒钟采样的频率。常见的取值为：
       -  SensorManager.SENSOR_DELAY_FASTEST：每秒80次。
       -  SensorManager.SENSOR_DELAY_GAME：每秒60次。
       -  SensorManager.SENSOR_DELAY_UI：每秒40次。
       -  SensorManager.SENSOR_DELAY_NORMAL：每秒20次。
    -  当传感器的数据发生变化时，系统会将相关的数据封装成一个SensorEvent对象并传递给onSensorChanged方法。
    -  SensorEvent有一个常用字段float[] values，传感器监听到的数据都保存在此数组中。对于不同的传感器，此数组的长度是不同的。
       -  对于方向传感器来说，values[0]表示手机当前指向的方向相对于北方的偏移角度。0北方、90东方、180南方、270西方。

# 第二节 广播接收者 #
　　在现实世界中发生一个新闻后，广播电台会广播这个新闻给打开收音机的人，对这个新闻感兴趣的人会关注。
　　`Android`也提供了类似的机制，它将`系统开关机`、`时间变更`、`屏幕变暗`、`电池电量不足通知`、`抓图通知`等等事件封装成一个广播，当这些事件发生时它就会通知所有关注该事件的应用软件。

　　`Broadcast Receivers` （广播接收者）是应用程序中用来接收广播的一个组件。

<br>**广播接收者的特点：**

	-  广播接收者是静止的，它不会主动做任何事情，而总是等待着广播的到来。
	   -  可以简单的理解：广播接收者是一个时刻等待冲锋的士兵，而广播就是一个信号，士兵接到信号后就可以做一些事情了。
	-  广播接收者可以接收来自用户自定义的广播和来自系统的广播。
	-  广播接收者并不是一个随便的人，它也会对外界传来的广播进行筛选，它所不需要的，它不会去接收。

## 基础知识 ##
　　若一个类继承了`BroadcastReceiver`那么它就是一个广播接收者。
　　我们使用一个`Intent`对象来代表`广播`，用户或系统发送广播时，发送的就是`Intent`对象，与广播相关的数据都被保存在`Intent`对象中了。

<br>　　范例1：简单的广播接收者。
``` java
public class MyBroadcastReceiver extends BroadcastReceiver {

    /**
     * BroadcastReceiver唯一一个需要子类重写的方法，当接到外界的广播时系统会调用此方法。
     * @param context: 上下文对象。 
     * @param intent: 广播相关的数据都保存在这个Intent对象中。
     */
    public void onReceive(Context context, Intent intent) {
        // 在屏幕上弹出一个消息，关于Toast的具体用法后面章节会有介绍。
        Toast.makeText(context, "Hi Tom", Toast.LENGTH_SHORT).show();
    }
}
```
    语句解释：
    -  当一个广播产生时，系统会拿广播依次和所有已经注册到系统中的广播接收者匹配。
    -  当某个广播接收者匹配成功后，系统都会创建该广播接收者的实例，然后调用该实例的onReceive方法，同时将广播Intent传入给onReceive方法。
    -  当onReceive方法执行完毕后，广播接收者的实例会被销毁。
    -  onReceive方法由主线程调用，因此在此方法中不要做一些耗时超过10秒的操作，否则系统将提示ANR(Application Not Responding)。
    -  使用onReceive方法中的context参数不能弹出一个Dialog，但是可以Toast。 

<br>　　做为`Android`中的四大组件之一，广播接收者同样需要在清单文件中进行声明。

<br>　　范例2：配置接收者。
``` xml
<receiver android:name="com.example.androidtest.MyBroadcastReceiver">
    <intent-filter>
        <action android:name="aaa.abc" />
    </intent-filter>
</receiver>
```
    语句解释：
    -  在<application>标签内部使用<receiver>标签来声明一个广播接收者。
	-  属性android:name：指明当前BroadcastReceiver所对应的类。
	-  当广播产生后，系统会依次拿广播Intent和所有的广播接收者的intent-filter进行匹配，若匹配成功，则将广播交给该广播接收者。一个广播可以同时被多个广播接收者接收。

<br>　　范例3：发送广播。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        Intent intent = new Intent();
        intent.setAction("aaa.abc");
        // Activity类是Context类的子类，调用Context的sendBroadcast()方法，发送一个无序广播。
        // 此时变量intent代表一个广播，广播的主要职责就是与广播接收者的意图过滤器(IntentFilter)进行匹配。
        this.sendBroadcast(intent);
    }
}
```
    语句解释：
    -  当MainActivity被启动后就可以在屏幕上看到一个Toast消息“Hi Tom”。

## 无序广播和有序广播 ##
　　广播分为：无序广播(`Normal broadcasts`)和有序广播(`Ordered broadcasts`)。

　　**无序广播：**
　　广播的发送者会将广播同时发送给所有符合条件的广播接收者。

　　**有序广播：**
　　与无序广播不同的是，广播最初会被发送给符合条件的、优先级最高的接收者，然后广播会从优先级最高的接收者手中开始，在所有符合条件组成的接收者集合中，按照优先级的高低，被依次传递下去。若多个接收者具有相同的优先级，则广播会被先传递给Android系统最先找到的那个接收者。

　　有序广播和无序广播最明显的区别在于发送它们时所调用的方法不同：

	-  无序广播：sendBroadcast(intent)
	-  有序广播：sendOrderedBroadcast(intent, receiverPermission)
　　除此之外，无序广播相对有序广播消息传递的效率比较高，但各个接收者无法终止和修改广播。而有序广播的某个接收者在中途可以终止、修改广播。

<br>　　范例1：广播接收者的优先级。
``` xml
<receiver android:name="com.example.androidtest.MyBroadcastReceiver2" >
    <intent-filter android:priority="1000" >
        <action android:name="aaa.abc" />
    </intent-filter>
</receiver>
```
    语句解释：
    -  用同样的方法再创建一个MyBroadcastReceiver2，并在配置它的时候使用android:priority属性指定它的优先级。
    -  优先级的取值范围是 -1000 ~ 1000 ，最高优先级为1000。

<br>　　范例2：发送广播。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        Intent intent = new Intent();
        intent.setAction("aaa.abc");
        // 发送有序广播
        this.sendOrderedBroadcast(intent, null);
    }
}
```
    语句解释：
    -  程序运行时MyBroadcastReceiver2会先接到广播，然后MyBroadcastReceiver才会接到。
    -  如果MyBroadcastReceiver2在接到广播后把广播给拦截了（让广播不再往下继续传递），那么MyBroadcastReceiver将无法接到广播。

<br>　　范例3：自定义权限。
``` xml
<permission android:name="cxy.mypermisson"/>
```
    语句解释：
    -  在<manifest>标签内部以及<application>标签的外部，使用<permission>标签可以自定义一个权限。
	-  属性android:name：指出权限的名称。权限的名称最少要是二级的。若权限的名称仅为：cxy 则是错误的。必须是二级以上：cxy.mypermisson。
	-  定义完权限后，在其他应用程序中就可以通过<uses-permission>标签直接使用这个权限。

<br>　　范例4：发送广播。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        Intent intent = new Intent();
        intent.setAction("abc");
        // 发送有序广播，只有申请了cxy.mypermisson权限的应用程序里所定义的广播接收者才能接到这个广播。
        this.sendOrderedBroadcast(intent, "cxy.mypermisson");
    }
}
```
    语句解释：
    -  此时只有在申请了cxy.mypermisson权限的应用程序中定义的BroadcastReceiver才可以接收到广播。 
    -  当程序运行时上面定义的两个广播接收者都接不到广播，若想让它们接到广播，则需要在<manifest>标签内部以及<application>标签的外部使用<uses-permission android:name="cxy.mypermisson" />申请权限，即便这个权限是它自己定义的，也需要申请。
	-  提示：一般情况下，当系统发送广播时，若广播接收者所在的应用程序并没有运行，则系统会自动将其运行。以保证广播能被顺利接收。

<br>　　范例5：终止广播。
``` java
public class MyBroadcastReceiver2 extends BroadcastReceiver{
    public void onReceive(Context context, Intent intent) {
        Toast.makeText(context, "Hi Tom2", Toast.LENGTH_SHORT).show();
        // 在某个广播接收者中调用此方法可以终止广播继续向下传递。
        // 如果广播被前面的接收者终止，后面的接收者就再也无法获取到广播。
        // 此方法仅对通过sendOrderedBroadcast方法发送的有序广播有效。
        abortBroadcast();
    }
}
```
    语句解释：
    -  由于MyBroadcastReceiver2的优先级比MyBroadcastReceiver大，所以后者将不会再接到广播。


<br>　　范例6：哥不是你可以终止的。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        Intent intent = new Intent();
        intent.setAction("abc");
        /*
         * 发送一个有序广播，同时指定一个广播接收者的对象，这个对象一定会接到广播，即便广播在中途被其他接收者终止了。
         * 第三个参数：必须接收当前广播的接收者。若不需要此设置，可以传递null。
         * 第四个参数：此参数为null，则接收者将在Context所在的主线程被调用。
         * 第五个参数：用于标识结果数据的结果码。通常为：Activity.RESULT_OK。
         * 第六个参数：传递给各个广播接收的一个String类型的数据。
         * 第七个参数：若String类型的数据，不能满足使用，则可以使用此参数，将其他数据附加到广播中。
         */
        sendOrderedBroadcast(intent,"cxy.mypermisson",
				new MyBroadcastReceiver(), null, Activity.RESULT_OK, null, null); 
    }
}
```
    语句解释：
    -  这种确保某个广播接收者一定能够接到广播的方式仅适用于你自己的广播接受者，因为我们是无法实例化其他应用程序的接收者的。
    -  如果广播在传播的过程中未被任何接收者终止，则最终MyBroadcastReceiver将会接收到2次广播，一次是正常传递来的调用，另一次则是为了确保一定接到广播导致的调用。


<br>　　范例7：发送数据。
``` java
public class MainActivity extends Activity {
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
		
        Intent intent = new Intent();
        intent.putExtra("string", "from intent");
        intent.setAction("abc");
        Bundle data = new Bundle();
        data.putString("string", "from bundle");
        sendOrderedBroadcast(intent,"cxy.mypermisson",
				new MyBroadcastReceiver(), null, Activity.RESULT_OK, "Hello World", data); 
    }
}
```
    语句解释：
    -  发送任何广播时都可以往Intent中设置附加数据，以供接收者使用。 
    -  发送有序广播时除了可以往Intent对象中设置数据外，还可以将数据放在一个Bundle对象中，然后通过sendOrderedBroadcast方法发送数据。

<br>　　范例8：接收数据。
``` java
public class MyBroadcastReceiver extends BroadcastReceiver {

    public void onReceive(Context context, Intent intent) {
        // 接收sendOrderedBroadcast()方法第六个参数的值。
        System.out.println(getResultData());
        // 接收intent对象设置的值，即范例7设置的：“from intent”。
        System.out.println(intent.getStringExtra("string"));
        // 接收sendOrderedBroadcast()方法第七个参数里的值，即范例7设置的：“from bundle”。
        System.out.println(getResultExtras(true).getString("string"));
    }
}
```
    语句解释：
    -  发送广播的时候，虽然我们可以确保某个接收者一定会接到广播，但是无法保证它接到的数据是最初的。 
    -  getResultExtras()方法的boolean类型参数用来设置当没有找到Bundle对象时，是否返回一个空的Bundle对象。 若置为false则当未设置Bundle对象时，此方法将返回null。


<br>　　范例9：妈的，还有谁?
``` java
public class MyBroadcastReceiver2 extends BroadcastReceiver{
    public void onReceive(Context context, Intent intent) {
        setResultData("from receiver 2");
        intent.putExtra("string", "from receiver 2");
        getResultExtras(true).putString("string", "from receiver 2");
        abortBroadcast();
    }
}
```
    语句解释：
    -  经过测试发现，修改intent中的值是无效的，之后的接收者接到的仍然是最初设置的值。
    -  但是调用setResultData()修改值和修改getResultExtras()方法获取的Bundle对象的值却是可以影响到后面的接收者的。

<br>　　范例10：广播的发送者，你也得证明你是个好人。
``` xml
<receiver 
    android:name="com.example.androidtest.MyBroadcastReceiver2"
    android:permission="org.cxy.permission.TEST">
    <intent-filter android:priority="888">
        <action android:name="myAction" />
    </intent-filter>
</receiver>
```
    语句解释：
    -  在声明BroadcastReceiver时，可以在<receiver>标签中指定android:permission属性。此属性表明，当前广播接收者仅接收，来自于拥有org.cxy.permission.TEST权限的应用程序中发出的广播。

## 动态注册 ##
　　事实上，注册广播的方法有两种，一种是静态注册，一种是动态注册：

	-  静态注册：通过在AndroidManifest.xml文件中添加<receiver>元素来将广播接收者注册到Android系统中。
	-  动态注册：在程序运行的时候，先创建一个你的BroadcastReceiver的对象，然后通过调用ContextWrapper类的registerReceiver方法将该对象注册到Android系统中。
<br>　　在Android的广播机制中，动态注册的优先级是要高于静态注册优先级的，而且有的广播只接收动态注册广播接收者，因此我们需要学习如果动态注册广播接收器。

<br>　　范例1：动态注册广播接收者。
``` java
public class MainActivity extends Activity {
    private static final String ACTION = "cn.etzmico.broadcastreceiverregister.SENDBROADCAST"; 
    private BroadcastReceiver myReceiver = new BroadcastReceiver() {
        public void onReceive(Context context, Intent intent) {
            Toast.makeText(context, "myReceiver receive", Toast.LENGTH_SHORT).show();
        }
    };
	
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        IntentFilter filter = new IntentFilter();  
        filter.addAction(ACTION);
        registerReceiver(myReceiver, filter);  
    }

    protected void onDestroy() {
        super.onDestroy();
        // 在Activity退出时解除注册。
        unregisterReceiver(myReceiver);
    }
}
```
　　动态注册广播接收器有一个特点，就是必须要程序运行的时候才可以注册，因此如果程序未运行就接不到广播了。

<br>**本节参考阅读：**
- [【Android】动态注册广播接收器 - CSDN博主 伊茨米可](http://blog.csdn.net/etzmico/article/details/7317528)

## 本地广播 ##

<br>　　在 Google 的[ 开发指南 ](https://developer.android.com/guide/components/broadcasts.html)中清楚的描述了，广播接受者是用于接受来自系统的消息，例如：系统启动、开始充电、应用安装等，但实际上，它被大量的误用于应用内部通信，这就违背了它最初的设计用途及理念。

　　直接使用 BroadcastReceiver 进行应用内部通信有如下的问题：

    -  广播接受者默认情况下是可以被任何第三方App唤醒的，数据的安全性不高。虽然可以通过设置权限等方式来禁止别人这么做。
    -  广播接收者的运行机制需要进程间通信的，内部通信通常很频繁且不需要跨进程。

　　基于这个现实的问题，Google 推出了一个新的API，叫做`LocalBroadcastManager`，专门用来实现应用内部通信。

<br>　　范例1：基础用法。
``` java
public class MainActivity extends AppCompatActivity {

    public static final String ACTION = "com.cutler.demo";
    private MyBroadcastReceiver mBroadcastReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // 像往常一样创建广播接受者对象
        mBroadcastReceiver = new MyBroadcastReceiver();
        // 将这个对象动态注册到LocalBroadcastManager中
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(ACTION);
        LocalBroadcastManager.getInstance(this).registerReceiver(mBroadcastReceiver, intentFilter);
    }

    /**
     * 界面中有个按钮，按钮被点击的时候会调用方法，发送一个广播
     */
    public void onClick(View view) {
        Intent intent = new Intent();
        intent.setAction(ACTION);
        LocalBroadcastManager.getInstance(this).sendBroadcast(intent);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        // 界面关闭的时候，解除注册
        LocalBroadcastManager.getInstance(this).unregisterReceiver(mBroadcastReceiver);
    }

    class MyBroadcastReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            Toast.makeText(context, "收到", Toast.LENGTH_SHORT).show();
        }
    }
}
```
    语句解释：
    -  LocalBroadcastManager内部没有什么高深的技术，一共就300多行代码，它其实就是观察者模式的实现。
    -  它的应用场景和观察者模式一样，如果您不了解观察者模式，后续博文中会有介绍。
    -  事实上Github上有一个名为EventBus的开源库，此类能实现的功能它都能实现，且更棒，所以此类并不常用，了解即可。


<br>　　需要注意的是，Google官方已经不再推荐使用这个类了，原文如下：

    LocalBroadcastManager is an application-wide event bus and embraces layer violations in your app: any component may listen events from any other.
    You can replace usage of LocalBroadcastManager with other implementation of observable pattern, depending on your usecase suitable options may be LiveData or reactive streams.

<br>**本节参考阅读：**
- [为什么应该使用本地广播](http://effmx.com/articles/wei-shi-yao-ying-gai-shi-yong-ben-di-yan-bo-localbroadcastmanager/)
- [Android本地广播和全局广播的区别及实现原理](https://blog.csdn.net/u010126792/article/details/82417190)

## 监听系统广播 ##

<br>　　范例1：以开机广播为例，我们需要监听`BOOT_COMPLETED`动作：
``` xml
<receiver android:name="com.example.androidtest.BootReceiver" >
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```
　　接收开机启动广播所需的权限：
``` xml
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
```


<br>　　范例2：时间改变广播：
``` java
public class MainActivity extends AppCompatActivity {

    private IntentFilter intentFilter;

    private TimeChangeReceiver timeChangeReceiver;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        intentFilter = new IntentFilter();
        //每分钟变化
        intentFilter.addAction(Intent.ACTION_TIME_TICK);
        //设置了系统时区
        intentFilter.addAction(Intent.ACTION_TIMEZONE_CHANGED);
        //设置了系统时间
        intentFilter.addAction(Intent.ACTION_TIME_CHANGED);
        timeChangeReceiver = new TimeChangeReceiver();
        registerReceiver(timeChangeReceiver, intentFilter);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        unregisterReceiver(timeChangeReceiver);
    }

    class TimeChangeReceiver extends BroadcastReceiver {
        @Override
        public void onReceive(Context context, Intent intent) {
            switch (intent.getAction()) {
                case Intent.ACTION_TIME_TICK:
                    //每过一分钟 触发
                    System.out.println("____ 1 min passed");
                    break;
                case Intent.ACTION_TIME_CHANGED:
                    //设置了系统时间
                    System.out.println("____ system time changed");
                    break;
                case Intent.ACTION_TIMEZONE_CHANGED:
                    //设置了系统时区的action
                    System.out.println("____ system time zone changed");
                    break;
                default:
                    break;
            }
        }
    }
}
```
    语句解释：
    -  其中ACTION_TIME_TICK事件要求必须动态注册，另外两种事件可以静态注册。

　　静态注册时，设置如下action即可：
``` xml
  <intent-filter>
    <!-- 设置时间、时区时触发 -->
    <action android:name="android.intent.action.TIME_SET"></action>
    <action android:name="android.intent.action.TIMEZONE_CHANGED"></action>
   </intent-filter>
```

<br>　　范例3：开屏、关屏、解锁广播：
``` java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        IntentFilter filter = new IntentFilter();
        // 开屏
        filter.addAction(Intent.ACTION_SCREEN_ON);
        // 关屏
        filter.addAction(Intent.ACTION_SCREEN_OFF);
        // 解锁屏幕，若用户没有设置任何方式的解锁，则会紧随开屏广播之后发出
        filter.addAction(Intent.ACTION_USER_PRESENT);
        receiver = new MyReceiver();
        registerReceiver(receiver, filter);
    }
}
```
    语句解释：
    -  其中开屏和关屏广播要求必须动态注册，ACTION_USER_PRESENT事件在Android O (8.0 api 26)以上则只能动态注册。
    -  为了让动态注册的广播接收者有更长的生命周期，可以在Application里注册它。

　　静态注册时，设置如下action即可：
``` xml
<intent-filter>
    <action android:name="android.intent.action.USER_PRESENT" />
</intent-filter>
```


<br>　　范例4：网络状态改变广播：
``` java
// WIFI总开关的打开和关闭事件
filter.addAction(WifiManager.WIFI_STATE_CHANGED_ACTION);

// 监听WIFI的连接状态即是否连上了一个有效无线路由
filter.addAction(WifiManager.NETWORK_STATE_CHANGED_ACTION);

// 监听网络连接的设置，包括WIFI和移动数据的打开和关闭
filter.addAction(ConnectivityManager.CONNECTIVITY_ACTION);
```
    语句解释：
    -  其中CONNECTIVITY_ACTION在Android7.0(api 24)以后必须动态注册，另外两个则在8.0以后才需要。
    -  另外需要注意的是，CONNECTIVITY_ACTION的最大弊端是比另外两个广播的反应要慢。
    -  如果仅仅是接收系统的广播（比如用来做进程保活），是可以不用申请权限的。

　　静态注册时，设置如下action即可：
``` xml
<intent-filter>
    <action android:name="android.net.wifi.WIFI_STATE_CHANGED" />
    <action android:name="android.net.wifi.STATE_CHANGE" />
    <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
</intent-filter>
```


<br>　　范例5：安装、更新、卸载广播：
``` java
// 一个新应用已经安装在设备上（监听所在的app安装时，接不到此广播。）
filter.addAction(Intent.ACTION_PACKAGE_ADDED);

// 一个新版本的应用覆盖安装到设备上。会先收到remove的再收到replace的，监听所在的app的更新也能收到。
// 所谓的更新就是versionCode或者versionName发生了变化
filter.addAction(Intent.ACTION_PACKAGE_REPLACED);

// 一个已存在的app已经从设备上卸载（卸载监听所在的app，监听不到自己的卸载）
filter.addAction(Intent.ACTION_PACKAGE_REMOVED);
```
    语句解释：
    -  这三个事件在Android O (8.0 api 26)以上则只能动态注册，8.0以下则可以静态注册。

　　静态注册时，设置如下action即可：
``` xml
<intent-filter>
    <action android:name="android.intent.action.PACKAGE_ADDED" />
    <action android:name="android.intent.action.PACKAGE_REPLACED" />
    <action android:name="android.intent.action.PACKAGE_REMOVED" />
    <data android:scheme="package" />
</intent-filter>
```

<br>　　范例6：充电状态广播：
``` java
// 每当设备连接或断开电源时，BatteryManager就会发送下面两个广播
filter.addAction(Intent.ACTION_POWER_CONNECTED);
filter.addAction(Intent.ACTION_POWER_DISCONNECTED);

// 电量发生改变时会发出此广播，此事件只能动态注册
filter.addAction(Intent.ACTION_BATTERY_CHANGED);

// 每当设备电池电量不足或退出不足状态时，便会触发下面两个广播
filter.addAction(Intent.ACTION_BATTERY_OKAY);
filter.addAction(Intent.ACTION_BATTERY_LOW);
```
    语句解释：
    -  前两个事件在Android O (8.0 api 26)以上则只能动态注册，8.0以下则可以静态注册。
    -  需要注意的是，ACTION_BATTERY_CHANGED只能动态注册。
    -  后两个广播不太好重现，估计和其它广播类似，8.0以后会受限制。

　　静态注册时，设置如下action即可：
``` xml
<intent-filter>
    <action android:name="android.intent.action.ACTION_POWER_CONNECTED"/>
    <action android:name="android.intent.action.ACTION_POWER_DISCONNECTED"/>
    <action android:name="android.intent.action.ACTION_BATTERY_LOW"/>
    <action android:name="android.intent.action.ACTION_BATTERY_OKAY"/>
</intent-filter>
```

# 第三节 各版本变化 #

## 广播 ##

　　本节将列出广播在各个Android版本中的表现。

### Android 3.1 ###
<br>　　从`Android 3.1`开始的Android加入了一种保护机制，这个机制导致程序接收不到系统广播。

　　系统为Intent添加了两个flag，`FLAG_INCLUDE_STOPPED_PACKAGES`和`FLAG_EXCLUDE_STOPPED_PACKAGES`，用来控制Intent是否要对处于`stopped`状态的App起作用，如果一个App`安装后未启动过`或者`被用户在管理应用中手动停止`（强行停止）的话，那么该App就处于`stopped`状态了。

　　顾名思义：

    -  FLAG_INCLUDE_STOPPED_PACKAGES：表示包含stopped的App
    -  FLAG_EXCLUDE_STOPPED_PACKAGES：表示不包含stopped的App

　　从`Android 3.1`开始，系统会在所有广播上添加了一个flag（`FLAG_EXCLUDE_STOPPED_PACKAGES`），这样的结果就是一个处于`stopped`状态的App就不能接收系统的广播，这样就可以防止病毒木马之类的恶意程序。

　　正常情况下，如果你的App没有处于`stopped`状态，那么当用户重启手机的时候，你的App就可以接收到开机启动广播了。
　　但事实并非这么简单，国内的各大手机厂商对开机启动的权限，除了上面的限制外，还有自己的策略。

    -  比如小米自己维护了一个白名单，默认情况下只有白名单内的App才可以被设置为开机启动（即便你的应用程序没有处于stopped状态也不会接收到开机启动广播）。这个白名单中通常包含一些使用比较广泛的App，比如微信、QQ等。当然事情还是有转机的，小米针对每一个App都提供了一个设置，如果用户在小米手机中手动设置允许你的程序开机启动的话，那么你的App就可以接到开机广播了。


### Android 7.0 ###

　　如果应用程序注册接收广播，则应用程序的接收器每次发送广播时都会消耗资源。如果有太多应用程序注册接收基于系统事件的广播，则会导致问题；触发广播的系统事件可能导致所有这些应用快速连续消耗资源，从而影响用户体验。
　　为了缓解此问题Android 7.0（API级别24）应用以下限制：

    -  从7.0开始应用在清单文件中静态注册的 CONNECTIVITY_ACTION 的广播接收器，不再会接到广播，但动态注册仍然可以。
    -  应用无法发送，接收 ACTION_NEW_PICTURE 或 ACTION_NEW_VIDEO 广播。此优化会影响所有应用，而不仅仅是针对API级别24的应用。

　　Android框架提供了多种解决方案来减轻对这些隐式广播的需求。例如`JobScheduler`或者`WorkManager`。


### Android 8.0 ###

　　为了进一步缓解资源消耗的问题，Android 8.0（API级别26）中实行了更加严格的限制：

    -  除了有限的例外之外，应用无法使用清单注册（静态注册）的方式来接收大部分的隐式广播。
　　但是：
    -  这些隐式广播，依然可以通过运行时注册（动态注册）的方式注册。
    -  对于显式广播，则依然可以通过清单注册（静态注册）的方式监听。

　　在Android8.0上，除了以下隐式广播外，其他所有隐式广播均无法通过在`AndroidManifest.xml`中注册监听：
``` java
/**
开机广播
 Intent.ACTION_LOCKED_BOOT_COMPLETED
 Intent.ACTION_BOOT_COMPLETED
*/
"保留原因：这些广播只在首次启动时发送一次，并且许多应用都需要接收此广播以便进行作业、闹铃等事项的安排。"

/**
增删用户
Intent.ACTION_USER_INITIALIZE
"android.intent.action.USER_ADDED"
"android.intent.action.USER_REMOVED"
*/
"保留原因：这些广播只有拥有特定系统权限的app才能监听，因此大多数正常应用都无法接收它们。"
    
/**
时区、ALARM变化
"android.intent.action.TIME_SET"
Intent.ACTION_TIMEZONE_CHANGED
AlarmManager.ACTION_NEXT_ALARM_CLOCK_CHANGED
*/
"保留原因：时钟应用可能需要接收这些广播，以便在时间或时区变化时更新闹铃"

/**
语言区域变化
Intent.ACTION_LOCALE_CHANGED
*/
"保留原因：只在语言区域发生变化时发送，并不频繁。 应用可能需要在语言区域发生变化时更新其数据。"

/**
Usb相关
UsbManager.ACTION_USB_ACCESSORY_ATTACHED
UsbManager.ACTION_USB_ACCESSORY_DETACHED
UsbManager.ACTION_USB_DEVICE_ATTACHED
UsbManager.ACTION_USB_DEVICE_DETACHED
*/
"保留原因：如果应用需要了解这些 USB 相关事件的信息，目前尚未找到能够替代注册广播的可行方案"

/**
蓝牙状态相关
BluetoothHeadset.ACTION_CONNECTION_STATE_CHANGED
BluetoothA2dp.ACTION_CONNECTION_STATE_CHANGED
BluetoothDevice.ACTION_ACL_CONNECTED
BluetoothDevice.ACTION_ACL_DISCONNECTED
*/
"保留原因：应用接收这些蓝牙事件的广播时不太可能会影响用户体验"

/**
Telephony相关
CarrierConfigManager.ACTION_CARRIER_CONFIG_CHANGED
TelephonyIntents.ACTION_*_SUBSCRIPTION_CHANGED
TelephonyIntents.SECRET_CODE_ACTION
TelephonyManager.ACTION_PHONE_STATE_CHANGED
TelecomManager.ACTION_PHONE_ACCOUNT_REGISTERED
TelecomManager.ACTION_PHONE_ACCOUNT_UNREGISTERED
*/
"保留原因：设备制造商 (OEM) 电话应用可能需要接收这些广播"

/**
账号相关
AccountManager.LOGIN_ACCOUNTS_CHANGED_ACTION
*/
"保留原因：一些应用需要了解登录帐号的变化，以便为新帐号和变化的帐号设置计划操作"

/**
应用数据清除
Intent.ACTION_PACKAGE_DATA_CLEARED
*/
"保留原因：只在用户显式地从 Settings 清除其数据时发送，因此广播接收器不太可能严重影响用户体验"
    
/**
软件包被移除
Intent.ACTION_PACKAGE_FULLY_REMOVED
*/
"保留原因：一些应用可能需要在另一软件包被移除时更新其存储的数据；对于这些应用，尚未找到能够替代注册此广播的可行方案"

/**
外拨电话
Intent.ACTION_NEW_OUTGOING_CALL
*/
"保留原因：执行操作来响应用户打电话行为的应用需要接收此广播"
    
/**
当设备所有者被设置、改变或清除时发出
DevicePolicyManager.ACTION_DEVICE_OWNER_CHANGED
*/
"保留原因：此广播发送得不是很频繁；一些应用需要接收它，以便知晓设备的安全状态发生了变化"
    
/**
日历相关
CalendarContract.ACTION_EVENT_REMINDER
*/
"保留原因：由日历provider发送，用于向日历应用发布事件提醒。因为日历provider不清楚日历应用是什么，所以此广播必须是隐式广播。"
    
/**
安装或移除存储相关广播
Intent.ACTION_MEDIA_MOUNTED
Intent.ACTION_MEDIA_CHECKING
Intent.ACTION_MEDIA_EJECT
Intent.ACTION_MEDIA_UNMOUNTED
Intent.ACTION_MEDIA_UNMOUNTABLE
Intent.ACTION_MEDIA_REMOVED
Intent.ACTION_MEDIA_BAD_REMOVAL
*/
"保留原因：这些广播是作为用户与设备进行物理交互的结果：安装或移除存储卷或当启动初始化时（当可用卷被装载）的一部分发送的，因此它们不是很常见，并且通常是在用户的掌控下"

/**
短信、WAP PUSH相关
Telephony.Sms.Intents.SMS_RECEIVED_ACTION
Telephony.Sms.Intents.WAP_PUSH_RECEIVED_ACTION

注意：需要申请以下权限才可以接收
"android.permission.RECEIVE_SMS"
"android.permission.RECEIVE_WAP_PUSH"
*/
"保留原因：SMS短信应用需要接收这些广播"

```

　　默认情况下，这个更改仅会影响`targetSDK >= 26`的应用。但是，用户可以从“设置”界面为任何应用启用这些限制，即使应用的目标是低于26的API级别。


<br>**本节参考阅读：**
- [咦，Oreo怎么收不到广播了？](https://juejin.im/post/5aefd27f6fb9a07ab45889cc)
- [后台执行限制](https://developer.android.com/about/versions/oreo/background#broadcasts)
- [Android 3.1 APIs](http://developer.android.com/about/versions/android-3.1.html)


## 服务 ##


### Android 8.0 ###

　　在后台中运行的服务会消耗设备资源，这可能降低用户体验。 为了缓解这一问题，系统对这些服务施加了一些限制:

    -  第一，系统不再允许后台应用创建后台服务，当后台应用中尝试调用startService方法时会抛出IllegalStateException。
    -  第二，而即使是在应用前台状态时启动的Service，在应用退到后台的一小段时间后（几分钟）也会被系统停止(等同于stopSelf方法被调用)，不过绑定服务不会受此影响。

　　上面提到了`前台应用`和`后台应用`的概念，需要注意的是，用于服务限制目的的后台定义与用于内存管理目的的后台应用的定义不同；一个应用按照内存管理的定义可能处于后台，但按照能够启动服务的定义却可能处于前台。

　　对于服务限制来说，如果满足以下任意条件，应用将被视为处于前台，否则视为处于后台：

    -  具有可见 Activity（不管该 Activity 已启动还是已暂停）。
    -  具有前台服务。
    -  另一个前台应用已关联到该应用（不管是通过绑定到其中一个服务，还是通过使用其中一个内容提供程序）。 例如，如果另一个应用绑定到该应用的服务，那么该应用处于前台：
       - IME、壁纸服务、通知侦听器、语音或文本服务


　　现在就有一个问题了，如果想在后台应用中创建前台服务该怎么办？

    -  因为，在8.0之前，创建前台服务的方式通常是先创建一个后台服务，然后将该服务推到前台。
    -  但是，在8.0及其之后，如果后台应用startService将导致crash。

　　现在 Android O 中提供了新的`startForegroundService`方法，该方法本质上和`startService`相同，只是要求`Service`中必须调用`startForeground()`，它和`startService`相比，可以随时进行调用，即使当前应用不在前台。
　　我们可以这样来实现：
 
    -  首先，调用 startForegroundService 并传入要执行 Service 的 Intent。这时会创建后台服务，我们需要马上将其提升到前台。
    -  然后，在被启动的 Service 中创建一个通知，注意优先级需要至少是 PRIORITY_LOW。
    -  最后，在服务内部调用 startForeground(id, notification)，注意在创建服务后，有五秒钟来调用 startForeground()，如果没有及时调用，系统将终止 Service 并声明应用 ANR。


<br>　　接下来我们来验证一下：当应用被切到后台时，绑定方式启动的服务是否会在几分钟后被停止。验证的步骤为：

    -  首先，创建一个BaseService，它只负责定时打印一些Log，和一些公用代码。
    -  然后，创建两个继承了BaseService的Service类，一个接受普通方式启动，一个接受绑定方式启动。
    -  接着，在Activity_A中分别通过两种方式启动它们。
    -  最后，点击Home键将程序切到后台，观察Log输出。

<br>　　范例1-1：创建一个父类
``` java
public class BaseService extends Service {
    protected Handler handler;
    public void onCreate() {
        super.onCreate();
        handler = new Handler();
        System.out.println("______________ onCreate " + getName());
    }
    public String getName() {
        return "BaseService";
    }
    protected Runnable runnable = new Runnable() {
        public void run() {
            System.out.println("______________ tick " + getName());
            handler.postDelayed(this, 5000);
        }
    };
    public void onDestroy() {
        super.onDestroy();
        handler.removeCallbacks(runnable);
        System.out.println("______________ onDestroy " + getName());
    }
    public IBinder onBind(Intent intent) {
        return null;
    }
}
```

<br>　　范例1-2：创建两个子Service
``` java
public class NormalService extends BaseService {
    public int onStartCommand(Intent intent, int flags, int startId) {
        System.out.println("______________ onStartCommand " + getName());
        handler.post(runnable);
        return START_STICKY;
    }
    public String getName() {
        return "普通服务";
    }
}

public class BindService extends BaseService {
    public IBinder onBind(Intent intent) {
        handler.post(runnable);
        System.out.println("______________ onBind " + getName());
        return new MyBinder();
    }
    public class MyBinder extends Binder {
        public BindService getService() {
            return BindService.this;
        }
    }
    public String getName() {
        return "绑定服务";
    }
}
```

<br>　　范例1-3：启动两个服务
``` java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        startService(new Intent(this, NormalService.class));
        bindService(new Intent(this, BindService.class), new ServiceConnection() {
            public void onServiceConnected(ComponentName name, IBinder service) {
                System.out.println("_____ 链接建立");
            }
            public void onServiceDisconnected(ComponentName name) {
                System.out.println("_____ 链接断开");
            }
        }, BIND_AUTO_CREATE);
    }
}
```

<br>　　通过观察输出得出结论：
 
    -  程序切到后台1分钟后，普通方式启动的服务会被杀掉，绑定仍然存活。
    -  绑定方式启动的服务会随着绑定它的组件销毁而销毁，除非是在Application中执行绑定。

<br>　　所以如果想让你的Service在8.0上活的更久，该怎么做不用再说了吧？


<br>**本节参考阅读：**
- [后台服务限制](https://developer.android.google.cn/about/versions/oreo/background.html#services)
- [内存管理](https://developer.android.google.cn/topic/performance/memory-overview.html)
- [Android Oreo 中对后台任务的限制](https://zhuanlan.zhihu.com/p/29289780)


