title: Android 关闭任务栈（Activity）的方法 [Android 源码修改]
date: 
tags: [android]
---

许多初学者刚开始学习Android的时候最初接触的都是Activity。而始终围绕Android开发者的一个问题是：Activity的跳转、显示、销毁以及更多的其他操作。而更难的是实现比较复杂界面控制的需求。

本文并不会说明android task栈和activity的关系，这类文章在Baidu、Google上到处都是。本文从将介绍关闭Activity的几种情况。

## **1. 退出整个应用，关闭当前应用的所有Activity。**
一种比较流行的Activity管理是通过一个Activity管理类，将本应用所有Activity打开和关闭都放到自定义栈中。以方便对每一个Activity的管理和控制。可以任意关闭和结束Activity。在必要的时候可以将所有Activity都一次性结束掉。以下是一个比较通用的Activity管理类。
**优点：**可以自由控制所有Activity
**缺点：**只能将本应用的Activity进行管理。

```java
public class MyActivityManager {
    private static MyActivityManager instance;
    private Stack<Activity> activityStack;// activity栈

    private MyActivityManager() {
    }

    // 单例模式
    public static MyActivityManager getInstance() {
        if (instance == null) {
            instance = new MyActivityManager();
        }
        return instance;
    }

    // 把一个activity压入栈中
    public void pushOneActivity(Activity actvity) {
        if (activityStack == null) {
            activityStack = new Stack<Activity>();
        }
        activityStack.add(actvity);
        Log.d("MyActivityManager ", "size = " + activityStack.size());
    }

    // 获取栈顶的activity，先进后出原则
    public Activity getLastActivity() {
        return activityStack.lastElement();
    }

    // 移除一个activity
    public void popOneActivity(Activity activity) {
        if (activityStack != null && activityStack.size() > 0) {
            if (activity != null) {
                activity.finish();
                activityStack.remove(activity);
                activity = null;
            }
        }
    }

    // 退出所有activity
    public void finishAllActivity() {
        if (activityStack != null) {
            while (activityStack.size() > 0) {
                Activity activity = getLastActivity();
                if (activity == null)
                    break;
                popOneActivity(activity);
            }
        }
    }
}
```

## **2. 退出所有taskAffinity的Activity**
finishAffinity
作用：关闭启动的所有的activity(这些activity都有android:taskAffinity=":finishing");

就比如：
列表界面--详情界面(taskaffinity)--评论界面(taskaffinity)--他人资料界面(taskaffinity)；然后调用 **finishAffinity();** 这样就直接回到列表界面，同时关闭详情界面，评论界面，他人资料界面。

使用：
```xml
    <activity
        android:name=".FinishAffinity"
        android:taskAffinity=":finishing" >
    </activity>
```
## **3. 退出一个Activity，且关闭其打开的所有Activity**

当我们的应用打开了第三方应用的界面过程中，如何关闭所用Activity呢？

比如：“用户详情”界面--“第三方图片选择头像”界面。此时，可能因为某些后台状态的变化，“用户详情”界面不得不退出，那么如何在退出的同时能够关闭“第三方图片选择头像”界面呢？

通过观察此时系统的任务栈（观察任务栈的方式：**adb dumpsys activity**），发现此时两个Activity属于统一个task中，那么系统是否提供了这样的API，可以直接关闭一个Task呢？答案是肯定的。

API如下：

    ActivityManager.removeTask(taskId, ActivityManager.REMOVE_TASK_KILL_PROCESS);

经过封装之后，我们就可以轻易实现上述需求了。

```
public class ActivityUtil {
    /**
     * 关闭所在component所在的task的所有Activity
     */
    public static void removeTaskByBaseComponent(Context context, ComponentName component) {
        final ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        final List<ActivityManager.RunningTaskInfo> tasks = am.getRunningTasks(100);
        int taskId = -1;
        if (tasks != null && tasks.size() > 0) {
            int len = tasks.size();
            for (int i=0;i<len;i++) {
                ActivityManager.RunningTaskInfo task = tasks.get(i);
                if (task != null && task.baseActivity.equals(component)) {
                    taskId = task.id;
                    break;
                }
            }
        }

        am.removeTask(taskId, ActivityManager.REMOVE_TASK_KILL_PROCESS);
    }
}

```

## **4. 关闭这个Activity所打开的其他Activity。**

**相比（3）更加麻烦的情况出现了～～**

关闭这个界面打开的其他界面（包括第三方Activity），而不关闭自己。
此时，**removeTask** 接口已经无法满足我们的需求了。

下面的封装是我们希望的一个接口。可是SDK中并没有提供 **removeActivityFromTask** 接口。
```
public class ActivityUtil {
    /**
     * 关闭所有component打开的Activity，除了自己
     */
    public static void removeActivityFromTaskBasedOnComponents(Context context, ComponentName component) {
        final ActivityManager am = (ActivityManager) context.getSystemService(Context.ACTIVITY_SERVICE);
        final List<ActivityManager.RunningTaskInfo> tasks = am.getRunningTasks(100);
        int taskId = -1;
        int num = 0;
        if (tasks != null && tasks.size() > 0) {
            int len = tasks.size();
            for (int i=0;i<len;i++) {
                ActivityManager.RunningTaskInfo task = tasks.get(i);
                if (task != null && task.baseActivity.equals(component)) {
                    taskId = task.id;
                    num = task.numActivities;
                    break;
                }
            }
        }

        for (int i = num - 1; i > 0; i--) {
            boolean ret = am.removeActivityFromTask(taskId, i);
        }
    }
}
```

**怎么办呢？**
于是想到，改造一下 removeTask(taskId, ActivityManager.REMOVE_TASK_KILL_PROCESS);
实现 removeActivityFromTask(taskId, index); 其中index为该task中Activity的位置。

框架修改的patch

core/java/android/app/ActivityManager.java
```
diff --git a/core/java/android/app/ActivityManager.java b/core/java/android/app/ActivityManager.java
index 7ca3459..46c4b86 100644
--- a/core/java/android/app/ActivityManager.java
+++ b/core/java/android/app/ActivityManager.java
@@ -827,6 +827,16 @@ public class ActivityManager {
         }
     }
 
+    public boolean removeActivityFromTask(int taskId, int activityIndex)
+            throws SecurityException {
+        try {
+            return ActivityManagerNative.getDefault().removeActivityFromTask(taskId, activityIndex);
+        } catch (RemoteException e) {
+            // System dead, we will be dead too soon!
+            return false;
+        }
+    }
+
      /*
       * If set, the process of the root activity of the task will be killed
       * as part of removing the task.
```
core/java/android/app/ActivityManagerNative.java
```
diff --git a/core/java/android/app/ActivityManagerNative.java b/core/java/android/app/ActivityManagerNative.java
index 397f0eb..2ed36a3 100755
--- a/core/java/android/app/ActivityManagerNative.java
+++ b/core/java/android/app/ActivityManagerNative.java
@@ -1769,6 +1769,17 @@ public abstract class ActivityManagerNative extends Binder implements IActivityM
             return true;
         }
 
+        case REMOVE_ACTIVITY_OF_TASK_TRANSACTION:
+        {
+            data.enforceInterface(IActivityManager.descriptor);
+            int taskId = data.readInt();
+            int activityIndex = data.readInt();
+            boolean result = removeActivityFromTask(taskId, activityIndex);
+            reply.writeNoException();
+            reply.writeInt(result ? 1 : 0);
+            return true;
+        }
+
         case REMOVE_TASK_TRANSACTION:
         {
             data.enforceInterface(IActivityManager.descriptor);
@@ -4334,6 +4345,20 @@ class ActivityManagerProxy implements IActivityManager
         return result;
     }
 
+    public boolean removeActivityFromTask(int taskId, int activityIndex) throws RemoteException {
+        Parcel data = Parcel.obtain();
+        Parcel reply = Parcel.obtain();
+        data.writeInterfaceToken(IActivityManager.descriptor);
+        data.writeInt(taskId);
+        data.writeInt(activityIndex);
+        mRemote.transact(REMOVE_ACTIVITY_OF_TASK_TRANSACTION, data, reply, 0);
+        reply.readException();
+        boolean result = reply.readInt() != 0;
+        reply.recycle();
+        data.recycle();
+        return result;
+    }
+
     public boolean removeTask(int taskId, int flags) throws RemoteException {
         Parcel data = Parcel.obtain();
         Parcel reply = Parcel.obtain();
```
core/java/android/app/IActivityManager.java
```
diff --git a/core/java/android/app/IActivityManager.java b/core/java/android/app/IActivityManager.java
index aa2fff1..9ed66e5 100755
--- a/core/java/android/app/IActivityManager.java
+++ b/core/java/android/app/IActivityManager.java
@@ -355,6 +355,7 @@ public interface IActivityManager extends IInterface {
     public int[] getRunningUserIds() throws RemoteException;
 
     public boolean removeSubTask(int taskId, int subTaskIndex) throws RemoteException;
+    public boolean removeActivityFromTask(int taskId, int activityIndex) throws RemoteException;
 
     public boolean removeTask(int taskId, int flags) throws RemoteException;
 
@@ -697,4 +698,5 @@ public interface IActivityManager extends IInterface {
     int GET_PERSISTED_URI_PERMISSIONS_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+181;
     int APP_NOT_RESPONDING_VIA_PROVIDER_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+182;
        int SEND_BOOT_FAST_COMPLETE_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION + 183;
+    int REMOVE_ACTIVITY_OF_TASK_TRANSACTION = IBinder.FIRST_CALL_TRANSACTION+184;
 }
```
services/java/com/android/server/am/ActivityManagerService.java
```
diff --git a/services/java/com/android/server/am/ActivityManagerService.java b/services/java/com/android/server/am/ActivityManagerService.java
index 0157f33..ef93bcb 100755
--- a/services/java/com/android/server/am/ActivityManagerService.java
+++ b/services/java/com/android/server/am/ActivityManagerService.java
@@ -7088,6 +7088,24 @@ public final class ActivityManagerService extends ActivityManagerNative
         }
     }
 
+    @Override
+    public boolean removeActivityFromTask(int taskId, int activityIndex) {
+        synchronized (this) {
+            enforceCallingPermission(android.Manifest.permission.REMOVE_TASKS,
+                    "removeActivityFromTask()");
+            long ident = Binder.clearCallingIdentity();
+            try {
+                TaskRecord tr = recentTaskForIdLocked(taskId);
+                if (tr != null) {
+                    return tr.removeTaskActivityLocked(activityIndex, true) != null;
+                }
+                return false;
+            } finally {
+                Binder.restoreCallingIdentity(ident);
+            }
+        }
+    }
+
     private void killUnneededProcessLocked(ProcessRecord pr, String reason) {
         if (!pr.killedByAm) {
             Slog.i(TAG, "Killing " + pr.toShortString() + " (adj " + pr.setAdj + "): " + reason);
```
services/java/com/android/server/am/TaskRecord.java
```
diff --git a/services/java/com/android/server/am/TaskRecord.java b/services/java/com/android/server/am/TaskRecord.java
index 3d568ff..8d2233e 100644
--- a/services/java/com/android/server/am/TaskRecord.java
+++ b/services/java/com/android/server/am/TaskRecord.java
@@ -206,6 +206,19 @@ final class TaskRecord extends ThumbnailHolder {
         return mActivities.size() == 0;
     }
 
+    ActivityRecord removeActivity(int index) {
+        int numActivities = mActivities.size();
+        if (index < numActivities) {
+            ActivityRecord r = mActivities.get(index);
+            if (stack.finishActivityLocked(r, Activity.RESULT_CANCELED, null, "clear", false)) {
+                removeActivity(r);
+                return r;
+            }
+            return null;
+        }
+        return null;
+    }
+
     /**
      * Completely remove all activities associated with an existing
      * task starting at a specified index.
```
```
@@ -316,6 +329,18 @@ final class TaskRecord extends ThumbnailHolder {
         return info.subtasks.get(info.numSubThumbbails-1).holder.lastThumbnail;
     }
 
+    public ActivityRecord removeTaskActivityLocked(int activityIndex, boolean taskRequired) {
+        if (activityIndex < 0 || activityIndex >= mActivities.size()) {
+            if (taskRequired) {
+                Slog.w(TAG, "removeTaskActivityLocked: unknown activityIndex " + activityIndex);
+            }
+            return null;
+        }
+
+        // Remove this task's activity at the activityIndex of the task.
+        return removeActivity(activityIndex);
+    }
+
     public ActivityRecord removeTaskActivitiesLocked(int subTaskIndex,
             boolean taskRequired) {
         TaskAccessInfo info = getTaskAccessInfoLocked(false);
```

