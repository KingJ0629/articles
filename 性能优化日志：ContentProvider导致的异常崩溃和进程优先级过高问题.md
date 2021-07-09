### 性能优化日志：ContentProvider导致的异常崩溃和进程优先级过高问题

#### 场景1介绍
应用A应用有这么个崩溃场景，当另外一个应用B崩溃的时候，应用A应用也随之崩溃

#### 场景1分析
应用A通过ContentProvider和应用B有跨进程通信，当使用ContentResolver.query的时候，系统会尝试去获取unstableProvider，如果ContentProvider所在进程死亡，则系统就会尝试去获取stableProvider，然后执行query操作。业务侧拿到的是unstableProvider还是stable是个盲盒。然后再说说stable与unstable的区别，采用unstable类型的ContentProvider的app不会因为远程ContentProvider进程的死亡而被杀，stable则恰恰相反。这便是ContentProvider坑爹之处，对于app无法事先决定创建的ContentProvider是stable，还是unstable 类型的，也便无法得知自己的进程是否会依赖于远程ContentProvider的生死。故上面的场景就是发生了应用B进程带崩了无辜的应用A进程。

#### 解决方案
解决方案就是通过反射主动去拿unstableProvider,自己掌握自己的命运。代码如下：
```java
ContentResolver resolver = context.getContentResolver();

// ReflectionUnit是个反射工具类 封装了些catch操作
Method acquireUnstableProvider = ReflectionUnit.maybeGetMethod(resolver.getClass(),"acquireUnstableProvider", Uri.class);
// 获取 unstableProvider
IContentProvider unstableProvider = (IContentProvider) ReflectionUnit.invoke(acquireUnstableProvider, resolver, uri);
if (unstableProvider == null) {
    Log.e(TAG, "fail to acquire unstable provider. use default query");
    return resolver.query(uri, projection, selection, selectionArgs, sortOrder);
}

Cursor qCursor = null;
try {
    Class<?> ICancellationSignalClass = ReflectionUnit.maybeForName("android.os.ICancellationSignal");
    if (ICancellationSignalClass == null){
        // 异常情况都返回原始的query
        return resolver.query(uri, projection, selection, selectionArgs, sortOrder);
    }
    Method getPackageName = ReflectionUnit.maybeGetMethod(resolver.getClass(), "getPackageName");
    String packageName = (String)ReflectionUnit.invoke(getPackageName, resolver);
    Method createSqlQueryBundle = ReflectionUnit.maybeGetMethod(resolver.getClass(), "createSqlQueryBundle", String.class, String[].class, String.class);
    if (createSqlQueryBundle == null){
        return resolver.query(uri, projection, selection, selectionArgs, sortOrder);
    }
    Bundle bundle = (Bundle) ReflectionUnit.invoke(createSqlQueryBundle, resolver, selection, selectionArgs, sortOrder);
    Method query = ReflectionUnit.maybeGetMethod(unstableProvider.getClass(), "query", String.class, Uri.class, String[].class, Bundle.class, ICancellationSignalClass);
    if (query == null){
        return resolver.query(uri, projection, selection, selectionArgs, sortOrder);
    }
    // 执行query方法
    qCursor = (Cursor) ReflectionUnit.invoke(query, unstableProvider, packageName, uri, projection, bundle, null);
    if (qCursor == null) {
        return null;
    }
    // Force query execution.  Might fail and throw a runtime exception here.
    int count = qCursor.getCount();
    Log.d(TAG, "use unstableQuery retrun query. count = " + count);

    return qCursor;
}
```

### 场景2介绍
一段时间后性能部门找过来，说是应用A的进程优先级异常的高，想要定位下问题

### 场景2分析
进程优先级最高的是前台应用，但我们应用不在前台的时候也拥有和前台进程一样的优先级，这样的情况只有可能应用正在与前台交互的过程中，持有一个正在和前台交互的ContentProvider是一种可能。结合性能组反馈我们的ContentProvider内部计数异常，我们再回头看我们上面的代码，结合ContentProvider源码中获取unstableProvider的代码，发现上面的代码只有`acquireUnstableProvider`过程，没有用完释放的过程，故我们模仿源码，在上面`try catch`的`finally`中通过反射调用`releaseUnstableProvider`。

### 解决方案
源码中`releaseUnstableProvider`是个抽象方法，我们要去找到真正的实现类`ContextImpl$ApplicationContentResolver`。

```java
Class<?> ApplicationContentResolverClass = Class.forName("android.app.ContextImpl$ApplicationContentResolver");
Method releaseUnstableProviderMethod = ApplicationContentResolverClass.getMethod("releaseUnstableProvider", IContentProvider.class);
releaseUnstableProviderMethod.setAccessible(true);
releaseUnstableProviderMethod.invoke(resolver, unstableProvider);
```
