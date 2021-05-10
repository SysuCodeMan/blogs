---
title: LeakCanary源码学习（基于LeakCanary2.7）
date: 2021-05-08 17:28:54
tags: 内存泄漏检测工具
---
### Overview
> LeakCanary is a memory leak detection library for Android.

![](https://raw.githubusercontent.com/SysuCodeMan/PicBed/main/20210508173009.png)
                                                        ——官方文档
                                                        
官方介绍言简意赅，LeakCanary是Android平台的一款内存泄漏检测工具，可以检测出应用当前存在内存泄漏的地方。

### 用法
``` gradle
dependencies {
  // debugImplementation because LeakCanary should only run in debug builds.
  debugImplementation 'com.squareup.leakcanary:leakcanary-android:2.7'
}
```
只需要在工程的依赖中加上这么一行即可，这里需要注意的是，添加依赖使用的API是**debugImplementation**——仅对debug包生效。
看到LeakCanary的用法时感觉很强大，仅仅引入依赖，没有添加任何java代码，即可以完成整个应用的内存泄漏检测。

### 概念
在学习LeakCanary进行内存泄漏检测的原理前，首先明确内存泄露的含义：不再使用的对象没有被正确销毁，导致内存不能被释放。
在java世界中，通常发生内存泄漏的原因是长声明周期的对象持有了短声明周期对象的引用。

### 基本原理
从LeakCanary的官方文档中可以总结出其工作原理：LeakCanary在Activity/Fragment/View/ViewModel这些对象生命周期终点时，会弱引用这些对象，然后等待5秒并进行垃圾回收，如果此后弱引用的对象仍然存在，则说明这些对象可能发生了内存泄漏；此时需要进行一次heap dump，分析出这些对象的被引用情况，并将结果呈现给用户。

### 1.初始化
LeakCanary2.7这个版本将初始化从显示的用户代码调用，改成了在ContentProvider的onCreate()中执行，这是一个很巧妙的设计，避免了对用户代码的侵入；当用户app依赖了LeakCanary之后，应用的application初始化之后自动执行LeakCanary里的ContentProvider会在onCreate()，从而触发LeakCanary的初始化操作。
负责初始化的ContentProvider在leakcanary-object-watcher-android这个库内，实现为AppWatcherInstaller
``` kotlin
internal sealed class AppWatcherInstaller : ContentProvider() {

  /**`
   * [MainProcess] automatically sets up the LeakCanary code that runs in the main app process.
   */
  internal class MainProcess : AppWatcherInstaller()

  /**
   * When using the `leakcanary-android-process` artifact instead of `leakcanary-android`,
   * [LeakCanaryProcess] automatically sets up the LeakCanary code
   */
  internal class LeakCanaryProcess : AppWatcherInstaller()

  override fun onCreate(): Boolean {
    val application = context!!.applicationContext as Application
    AppWatcher.manualInstall(application)
    return true
  }

  override fun query(
    uri: Uri,
    strings: Array<String>?,
    s: String?,
    strings1: Array<String>?,
    s1: String?
  ): Cursor? {
    return null
  }

  override fun getType(uri: Uri): String? {
    return null
  }

  override fun insert(
    uri: Uri,
    contentValues: ContentValues?
  ): Uri? {
    return null
  }

  override fun delete(
    uri: Uri,
    s: String?,
    strings: Array<String>?
  ): Int {
    return 0
  }

  override fun update(
    uri: Uri,
    contentValues: ContentValues?,
    s: String?,
    strings: Array<String>?
  ): Int {
    return 0
  }
}
```
这个ContentProvider的实现里，query/getType/insert/delete/update方法都是空实现，因为本来也不是真正的ContentProvider，仅仅是利用了ContentProvider的onCreate()机制来自动触发我们库的初始化，这个思路很巧妙，我们在进行debug工具开发的时候也可以借鉴这个写法。
在onCreate()中拿到了用户应用的application，然后调用AppWatcher的manualInstall()方法：
``` kotlin
  @JvmOverloads
  fun manualInstall(
    application: Application,
    retainedDelayMillis: Long = TimeUnit.SECONDS.toMillis(5),
    watchersToInstall: List<InstallableWatcher> = appDefaultWatchers(application)
  ) {
    checkMainThread()
    if (isInstalled) {
      throw IllegalStateException(
        "AppWatcher already installed, see exception cause for prior install call", installCause
      )
    }
    check(retainedDelayMillis >= 0) {
      "retainedDelayMillis $retainedDelayMillis must be at least 0 ms"
    }
    installCause = RuntimeException("manualInstall() first called here")
    this.retainedDelayMillis = retainedDelayMillis
    if (application.isDebuggableBuild) {
      LogcatSharkLog.install()
    }
    // Requires AppWatcher.objectWatcher to be set
    LeakCanaryDelegate.loadLeakCanary(application)

    watchersToInstall.forEach {
      it.install()
    }
  }
```
这个方法分成以下部分：
- 1.调用appDefaultWatchers()获得watchersToInstall对象集合
- 2.调用LeakCanaryDelegate.loadLeakCanary(application)
- 3.触发所有watchersToInstall的install()函数

### 1.1 AppWatcher.appDefaultWatchers()
``` kotlin
  fun appDefaultWatchers(
    application: Application,
    reachabilityWatcher: ReachabilityWatcher = objectWatcher
  ): List<InstallableWatcher> {
    return listOf(
      ActivityWatcher(application, reachabilityWatcher),
      FragmentAndViewModelWatcher(application, reachabilityWatcher),
      RootViewWatcher(reachabilityWatcher),
      ServiceWatcher(reachabilityWatcher)
    )
  }
```
默认的Watcher是以上4个。

### 1.2 LeakCanaryDelegate.loadLeakCanary(application)
``` kotlin
  val loadLeakCanary by lazy {
    try {
      val leakCanaryListener = Class.forName("leakcanary.internal.InternalLeakCanary")
      leakCanaryListener.getDeclaredField("INSTANCE")
        .get(null) as (Application) -> Unit
    } catch (ignored: Throwable) {
      NoLeakCanary
    }
  }
```
完成InternalLeakCanary类的加载

### 1.3 触发所有watchersToInstall的install()函数，以ActivityWatcher为例：
``` kotlin
  override fun install() {
    application.registerActivityLifecycleCallbacks(lifecycleCallbacks)
  }
```
调用Application的registerActivityLifecycleCallbacks，监听Activity的生命周期，回调函数的实现是：
``` kotlin
private val lifecycleCallbacks =
    object : Application.ActivityLifecycleCallbacks by noOpDelegate() {
      override fun onActivityDestroyed(activity: Activity) {
        reachabilityWatcher.expectWeaklyReachable(
          activity, "${activity::class.java.name} received Activity#onDestroy() callback"
        )
      }
    }
```
在Activity执行onDestroy()时执行调用reachabilityWatcher的expectWeaklyReachable()方法，这里使用的reachabilityWatcher实现类是ObjectWatcher，expectWeaklyReachable()的实现如下：
``` kotlin
  @Synchronized override fun expectWeaklyReachable(
    watchedObject: Any,
    description: String
  ) {
    if (!isEnabled()) {
      return
    }
    removeWeaklyReachableObjects()
    val key = UUID.randomUUID()
      .toString()
    val watchUptimeMillis = clock.uptimeMillis()
    val reference =
      KeyedWeakReference(watchedObject, key, description, watchUptimeMillis, queue)
    SharkLog.d {
      "Watching " +
        (if (watchedObject is Class<*>) watchedObject.toString() else "instance of ${watchedObject.javaClass.name}") +
        (if (description.isNotEmpty()) " ($description)" else "") +
        " with key $key"
    }

    watchedObjects[key] = reference
    checkRetainedExecutor.execute {
      moveToRetained(key)
    }
  }
```
这个方法有以下3部分组成：
- 1.调用removeWeaklyReachableObjects()进行一次可达性检查
- 2.将当前需要被观察的watchedObject记录到watchedObjects这个集合里
- 3.调用checkRetainedExecutor触发moveToRetained()
下面分别展开看下removeWeaklyReachableObjects()和moveToRetained()的实现：

### 1.3.1 removeWeaklyReachableObjects()
``` kotlin
  private fun removeWeaklyReachableObjects() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    var ref: KeyedWeakReference?
    do {
      ref = queue.poll() as KeyedWeakReference?
      if (ref != null) {
        watchedObjects.remove(ref.key)
      }
    } while (ref != null)
  }
```
这里的queue是ReferenceQueue<Any>()，在构建弱引用时，是通过WeakReference<Any>(referent, referenceQueue)的方式构建的，传入的第二个参数referenceQueue就是这个queue对象
**构建WeakReference时，如果referent的可达性更改成只有弱引用，则会将这个referent添加到传入的referenceQueue中**
理解了上面的这个referenceQueue的作用，便不难理解这段方法的作用：
不断从queue中取出对象，因为这些对象是只剩弱引用的，在下次GC必然会被回收，因此我们没有必要再继续观察它们，所以把它们从watchedObjects中移除

### 1.3.2 moveToRetained()
``` kotlin
  @Synchronized private fun moveToRetained(key: String) {
    removeWeaklyReachableObjects()
    val retainedRef = watchedObjects[key]
    if (retainedRef != null) {
      retainedRef.retainedUptimeMillis = clock.uptimeMillis()
      onObjectRetainedListeners.forEach { it.onObjectRetained() }
    }
  }
```
这里再一次调用了removeWeaklyReachableObjects()检查当前这个对象是否真的处于Retained状态，如果是的话，则触发onObjectRetainedListeners的onObjectRetained()方法；这里比较重要的
onObjectRetainedListener是InternalLeakCanary中的实现：
``` kotlin
  override fun onObjectRetained() = scheduleRetainedObjectCheck()
``` 
会调用到scheduleRetainedObjectCheck()
``` kotlin
  fun scheduleRetainedObjectCheck() {
    if (this::heapDumpTrigger.isInitialized) {
      heapDumpTrigger.scheduleRetainedObjectCheck()
    }
  }
```
转到HeapDumpTrigger的scheduleRetainedObjectCheck()方法：
``` kotlin
fun scheduleRetainedObjectCheck(
    delayMillis: Long = 0L
  ) {
    val checkCurrentlyScheduledAt = checkScheduledAt
    if (checkCurrentlyScheduledAt > 0) {
      return
    }
    checkScheduledAt = SystemClock.uptimeMillis() + delayMillis
    backgroundHandler.postDelayed({
      checkScheduledAt = 0
      checkRetainedObjects()
    }, delayMillis)
  }
```
然后执行checkRetainedObjects()方法
``` kotlin
  private fun checkRetainedObjects() {
    val iCanHasHeap = HeapDumpControl.iCanHasHeap()

    val config = configProvider()

    // 这个条件语句是用于判断能否进行DumpHeap，正常情况不会走进去，暂时略过这里面的细节
    if (iCanHasHeap is Nope) {
      if (iCanHasHeap is NotifyingNope) {
        // Before notifying that we can't dump heap, let's check if we still have retained object.
        var retainedReferenceCount = objectWatcher.retainedObjectCount

        if (retainedReferenceCount > 0) {
          gcTrigger.runGc()
          retainedReferenceCount = objectWatcher.retainedObjectCount
        }

        val nopeReason = iCanHasHeap.reason()
        val wouldDump = !checkRetainedCount(
          retainedReferenceCount, config.retainedVisibleThreshold, nopeReason
        )

        if (wouldDump) {
          val uppercaseReason = nopeReason[0].toUpperCase() + nopeReason.substring(1)
          onRetainInstanceListener.onEvent(DumpingDisabled(uppercaseReason))
          showRetainedCountNotification(
            objectCount = retainedReferenceCount,
            contentText = uppercaseReason
          )
        }
      } else {
        SharkLog.d {
          application.getString(
            R.string.leak_canary_heap_dump_disabled_text, iCanHasHeap.reason()
          )
        }
      }
      return
    }

    var retainedReferenceCount = objectWatcher.retainedObjectCount

    if (retainedReferenceCount > 0) {
      // 如果此时的retained的对象大于0，主动触发一次gc
      gcTrigger.runGc()
      retainedReferenceCount = objectWatcher.retainedObjectCount
    }

    if (checkRetainedCount(retainedReferenceCount, config.retainedVisibleThreshold)) return

    val now = SystemClock.uptimeMillis()
    val elapsedSinceLastDumpMillis = now - lastHeapDumpUptimeMillis
    // 这个条件语句作用是判断当前时间和上一次进行dump操作隔了多久，如果在WAIT_BETWEEN_HEAP_DUMPS_MILLIS区间内，则这时候不会马上进行dump操作，而是延迟一段时间再操作
    if (elapsedSinceLastDumpMillis < WAIT_BETWEEN_HEAP_DUMPS_MILLIS) {
      onRetainInstanceListener.onEvent(DumpHappenedRecently)
      showRetainedCountNotification(
        objectCount = retainedReferenceCount,
        contentText = application.getString(R.string.leak_canary_notification_retained_dump_wait)
      )
      scheduleRetainedObjectCheck(
        delayMillis = WAIT_BETWEEN_HEAP_DUMPS_MILLIS - elapsedSinceLastDumpMillis
      )
      return
    }

    dismissRetainedCountNotification()
    val visibility = if (applicationVisible) "visible" else "not visible"
    // 触发dumpHeap()方法进行dump操作
    dumpHeap(
      retainedReferenceCount = retainedReferenceCount,
      retry = true,
      reason = "$retainedReferenceCount retained objects, app is $visibility"
    )
  }
```
这个方法乍一看还有点长，但实际上正常情况下里面的一些条件语句都是不用走进去的，已经在注释中说明，简单来说，就是判断当前是不是真的需要进行dumpHeap操作
下面看下dumpHeap()操作的具体实现：
``` kotlin
  private fun dumpHeap(
    retainedReferenceCount: Int,
    retry: Boolean,
    reason: String
  ) {
    saveResourceIdNamesToMemory()
    val heapDumpUptimeMillis = SystemClock.uptimeMillis()
    KeyedWeakReference.heapDumpUptimeMillis = heapDumpUptimeMillis
    when (val heapDumpResult = heapDumper.dumpHeap()) {
      is NoHeapDump -> {
        if (retry) {
          SharkLog.d { "Failed to dump heap, will retry in $WAIT_AFTER_DUMP_FAILED_MILLIS ms" }
          scheduleRetainedObjectCheck(
            delayMillis = WAIT_AFTER_DUMP_FAILED_MILLIS
          )
        } else {
          SharkLog.d { "Failed to dump heap, will not automatically retry" }
        }
        showRetainedCountNotification(
          objectCount = retainedReferenceCount,
          contentText = application.getString(
            R.string.leak_canary_notification_retained_dump_failed
          )
        )
      }
      is HeapDump -> {
        lastDisplayedRetainedObjectCount = 0
        lastHeapDumpUptimeMillis = SystemClock.uptimeMillis()
        objectWatcher.clearObjectsWatchedBefore(heapDumpUptimeMillis)
        HeapAnalyzerService.runAnalysis(
          context = application,
          heapDumpFile = heapDumpResult.file,
          heapDumpDurationMillis = heapDumpResult.durationMillis,
          heapDumpReason = reason
        )
      }
    }
  }
```
分成两步：
- 调用heapDumper.dumpHeap()对堆栈进行真正的dump操作，得到当前堆栈情况，最重要的产物是heapDumpFile
- 通过HeapAnalyzerService.runAnalysis()来对当前堆栈情况heapDumpFile进行分析
再往后的操作就是通过shark分析heapDumpFile，找出完成的引用链并展示。

### 总结
- LeakCanary2.7版本已经不需要用户代码再显式初始化，而是通过ContentProvider的机制
- LeakCanary对Activity进行内存泄漏检测的原理是，通过application来监听应用所有Activity的生命周期，在onDestory()执行时开始观察它；假如onDestory()执行一段时间后这个对象仍然存活，则处于retained状态；当retained状态的对象数量达到阈值（默认为5），则会进行一次dumpHeap操作
- 通过dumpHeap操作得到的heapDumpFile，调用shark进行引用链的分析




