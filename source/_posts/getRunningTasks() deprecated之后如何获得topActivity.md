---
title: getRunningTasks() deprecated之后如何获得topActivity
date: 2016-05-26 13:58:29
tags: Android
categories: Android

---
#### 前言
在Android sdk 22之前，要想获得topActivity，只需要用getRunningTasks(int maxNum)即可返回当前正在运行的任务栈，从任务栈中取出第一个即可，如下：
```java
ActivityManager am = ((ActivityManager) getSystemService(ACTIVITY_SERVICE));
List<ActivityManager.RunningTaskInfo> taskInfo = am.getRunningTasks(5);
String packageName = taskInfo.get(0).topActivity.getPackageName();
```
如上即可获得topActivity的包名，有了包名之后想干什么就干什么了。不过，如果就这么简单，就不值得记录学习了。
#### getRunningTasks() was deprecated
在Android L(也就是sdk 22)之后，getRunningTasks()方法以及被抛弃，原因是出于保护用户隐私之类的。如下：
>for L we do plan to have a new solution that will address some of the existing use cases of getRecentTasks(): we hope to expose the system’s internal usage stats service in a new and expanded form that is available to third party apps through the SDK.
>The new usage stats service should have two major pieces of information. First, it will provided aggregated stats of the time the user spent in each app, and the last launch time of those apps, rolled up over days, weeks, months, and years. Second, it will include detailed usage events over the last few days, describing the time and ComponentName when any activity transitions to the foreground or background.
Access to this new usage data will require that the user explicitly grant the app access to usage stats through a new settings UI. They will be able to see all apps that have access and revoke access at any point.

大概意思就是该方法以及被抛弃，然后用了一种新的更好地方式来代替，不过这种方式需要用户授予权限。这种方式就是用`UsageStatsManager`，详细请点击[here](https://developer.android.com/reference/android/app/usage/UsageStatsManager.html)。
下面直接上代码，在Android L之后获取topActivity包名：
```java
/**
 * 获得top activity的包名
 * @return
 */
public String getTopPackage(){
    long ts = System.currentTimeMillis();
    UsageStatsManager mUsageStatsManager = (UsageStatsManager)getSystemService(Context.USAGE_STATS_SERVICE);
    List<UsageStats> usageStats = mUsageStatsManager.queryUsageStats(UsageStatsManager.INTERVAL_BEST, ts-1000, ts);
    if (usageStats == null || usageStats.size() == 0) {//如果为空则返回""
        return "";
    }
    Collections.sort(usageStats, mRecentComp);//mRecentComp = new RecentUseComparator()
    return usageStats.get(0).getPackageName();
}
```
下面是`RecentUseComparator`的代码
```java
static class RecentUseComparator implements Comparator<UsageStats> {
    @Override
    public int compare(UsageStats lhs, UsageStats rhs) {
        return (lhs.getLastTimeUsed() > rhs.getLastTimeUsed()) ? -1 : (lhs.getLastTimeUsed() == rhs.getLastTimeUsed()) ? 0 : 1;
    }
}
```
使用UsageStatsManager需要如下权限：
```xml
<uses-permission xmlns:tools="http://schemas.android.com/tools"
    android:name="android.permission.PACKAGE_USAGE_STATS"
    tools:ignore="ProtectedPermissions" />
```
此外还需要用户授权去请求数据，可以用一下代码引导用户去设置页面开启
```java
Intent intent = new Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS);
startActivity(intent);
```
当前还可以通过一下代码来检测用户是否已经授权，如果已经授权过了，则不用重复去引导用户
```java
/**
 * 使用UsageStatsManager需要用户允许开启，该方法用于判断用户是否已经授权
 * @param context
 * @return true:还没有授权 false:已经授权
 */
public static boolean needPermissionForBlocking(Context context) {
    try {
        PackageManager packageManager = context.getPackageManager();
        ApplicationInfo applicationInfo = packageManager.getApplicationInfo(context.getPackageName(), 0);
        AppOpsManager appOpsManager = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);
        int mode = appOpsManager.checkOpNoThrow(AppOpsManager.OPSTR_GET_USAGE_STATS, applicationInfo.uid, applicationInfo.packageName);
        return  (mode != AppOpsManager.MODE_ALLOWED);
    } catch (PackageManager.NameNotFoundException e) {
        return true;
    }
}
```
如此，我们就可以在全平台中获得topActivity的包名了，也就可以“为所欲为”啦。
#### demo
下面是我做的一个应用锁小应用，目前在Android 4.3和Android 5.1完美运行。话不多说，show you the code !
在service中开启服务来监听应用程序的启动

```java
public class AppLockService extends Service {

    /**
     * 用来让当WatchDogService被stop时停止循环遍历任务栈的topActivity
     * true:循环遍历
     * false:停止遍历
     */
    private boolean flag = false;

    /**
     * 任务栈topActivity的包名
     */
    private String packageName = "";

    public AppLockService() {
    }

    @Override
    public IBinder onBind(Intent intent) {
        // TODO: Return the communication channel to the service.
        throw new UnsupportedOperationException("Not yet implemented");
    }

    @Override
    public void onCreate() {
        //分线程中循环遍历
        new Thread() {
            @Override
            public void run() {
                flag = true;
                while (flag) {
                    //获取包名
                    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {//版本小于lollipop的
                        ActivityManager am = ((ActivityManager) getSystemService(ACTIVITY_SERVICE));
                        List<ActivityManager.RunningTaskInfo> taskInfo = am.getRunningTasks(5);
                        packageName = taskInfo.get(0).topActivity.getPackageName();
                    } else { //版本为Lollipop及以上
                        if (needPermissionForBlocking(getApplicationContext())) {
                            //如果用户没有授权，引导用户去设置页面授权
                            Intent intent = new Intent(Settings.ACTION_USAGE_ACCESS_SETTINGS);
                            //在service中开启activity需要为intent添加FLAG_ACTIVITY_NEW_TASK的flag
                            intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                            startActivity(intent);
                        } else {
                            //得到包名
                            packageName = getTopPackage();
                        }
                        Log.e("TAG", "top app = " + packageName);
                        //有了包名之后就可以与应用锁数据库里面的包名做匹配
                        //如果匹配上就弹出输入密码的页面，酱紫就可以实现应用锁了(是不是有点想得太简单了= =)
                    }
                    SystemClock.sleep(50);
                }
            }
        }.start();

    }

    /**
     * 获得top activity的包名
     *
     * @return
     */
    @TargetApi(21)
    public String getTopPackage() {
        long ts = System.currentTimeMillis();
        UsageStatsManager mUsageStatsManager = (UsageStatsManager) getSystemService(Context.USAGE_STATS_SERVICE);
        List<UsageStats> usageStats = mUsageStatsManager.queryUsageStats(UsageStatsManager.INTERVAL_BEST, ts - 1000, ts);
        if (usageStats == null || usageStats.size() == 0) {//如果为空则返回""
            return "";
        }
        Collections.sort(usageStats, new RecentUseComparator());
        return usageStats.get(0).getPackageName();
    }

    /**
     *
     */
    @TargetApi(21)
    static class RecentUseComparator implements Comparator<UsageStats> {
        @Override
        public int compare(UsageStats lhs, UsageStats rhs) {
            return (lhs.getLastTimeUsed() > rhs.getLastTimeUsed()) ? -1 : (lhs.getLastTimeUsed() == rhs.getLastTimeUsed()) ? 0 : 1;
        }
    }

    /**
     * 使用UsageStatsManager需要用户允许开启，该方法用于判断用户是否已经授权
     *
     * @param context
     * @return true:还没有授权 false:已经授权
     */
    @TargetApi(19)
    public static boolean needPermissionForBlocking(Context context) {
        try {
            PackageManager packageManager = context.getPackageManager();
            ApplicationInfo applicationInfo = packageManager.getApplicationInfo(context.getPackageName(), 0);
            AppOpsManager appOpsManager = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);
            int mode = appOpsManager.checkOpNoThrow(AppOpsManager.OPSTR_GET_USAGE_STATS, applicationInfo.uid, applicationInfo.packageName);
            return (mode != AppOpsManager.MODE_ALLOWED);
        } catch (PackageManager.NameNotFoundException e) {
            return true;
        }
    }

    @Override
    public void onDestroy() {
        flag = false;
    }
}
```

#### 结语
至此，在getRunningTasks方法被抛弃后，终于完成了一个能够适应于多版本sdk的应用锁。不过，也只是仅仅能用而已，以上肯定不是最优的解决方案。由于自己功力尚浅，如果有错误，看到的小伙伴请多多指教！


