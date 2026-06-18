# Patch 6: Fix for Android 16 (ZUXOS) Compatibility Issues

This patch addresses several `NoSuchFieldError` and `NoSuchMethodError` crashes occurring on devices running Android 16 (specifically reported on Lenovo ZUXOS 1.5.10). 

## The Bug

In Android 16, certain internal framework APIs and fields that XEdgePro relies on via Reflection have been modified or removed. When the module attempts to read these non-existent fields, it crashes the System Server or prevents core module functionalities from initializing.

The reported issues are:
1. `AudioService#mStreamVolumeAlias` (NoSuchFieldError): Causes audio control initialization to fail.
2. `WindowManagerService#freezeRotation(int)` (NoSuchMethodError): Causes a crash when attempting to lock screen rotation.
3. `InputMethodManagerService#mCurInputContext` (NoSuchFieldError): Breaks IME (keyboard) state detection.

## Applied Changes

We implement robust fallback mechanisms (try-catch blocks and default values) for these reflection calls. If the Android 16 API fails, the application will gracefully default to safe values instead of crashing.

### 1. `smali/com/jozein/xedgepro/xposed/e.smali` (`mStreamVolumeAlias` Fallback)
Modified the `f()` method which attempts to read the volume alias array. If the reflection fails, it now returns a standard fallback integer array instead of `null`.

**BEFORE:**
```smali
.method f()[I
    .locals 2

    :try_start_0
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/e;->d:Ljava/lang/Object;

    const-string v1, "mStreamVolumeAlias"

    invoke-static {v0, v1}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;

    move-result-object v0

    check-cast v0, [I
    :try_end_0
    .catchall {:try_start_0 .. :try_end_0} :catchall_0

    return-object v0

    :catchall_0
    move-exception v0

    invoke-static {v0}, Lf/v;->d(Ljava/lang/Throwable;)V

    const/4 v0, 0x0

    return-object v0
.end method
```

**AFTER:**
```smali
.method f()[I
    .locals 2

    :try_start_0
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/e;->d:Ljava/lang/Object;

    const-string v1, "mStreamVolumeAlias"

    invoke-static {v0, v1}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;

    move-result-object v0

    check-cast v0, [I
    :try_end_0
    .catchall {:try_start_0 .. :try_end_0} :catchall_0

    return-object v0

    :catchall_0
    const/16 v0, 0xc

    new-array v0, v0, [I

    fill-array-data v0, :array_fallback

    return-object v0

    :array_fallback
    .array-data 4
        0x0
        0x1
        0x2
        0x3
        0x4
        0x5
        0x6
        0x7
        0x8
        0x9
        0xa
        0xb
    .end array-data
.end method
```

### 2. `smali/com/jozein/xedgepro/xposed/m2.smali` (`freezeRotation` Safely Catch)
Modified the `D(I)` method. Wrapped the reflective call to `freezeRotation` in a try-catch block. If the method does not exist on Android 16, it catches the `Throwable` and prevents the crash.

**BEFORE:**
```smali
.method D(I)V
    .locals 2

    const/4 v0, 0x1

    new-array v0, v0, [Ljava/lang/Object;

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object p1

    const/4 v1, 0x0

    aput-object p1, v0, v1

    const-string p1, "freezeRotation"

    invoke-virtual {p0, p1, v0}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;

    return-void
.end method
```

**AFTER:**
```smali
.method D(I)V
    .locals 2

    :try_start_0
    const/4 v0, 0x1

    new-array v0, v0, [Ljava/lang/Object;

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object p1

    const/4 v1, 0x0

    aput-object p1, v0, v1

    const-string p1, "freezeRotation"

    invoke-virtual {p0, p1, v0}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;
    :try_end_0
    .catchall {:try_start_0 .. :try_end_0} :catchall_0

    :catchall_0
    return-void
.end method
```

### 3. `smali/com/jozein/xedgepro/xposed/p0.smali` (`mCurInputContext` Safe Return)
Modified the `F()` method. It previously threw an explicit exception (`p0$b`) if the context was null. Changed it to catch potential reflection errors and simply return `null` if it fails, preventing the module from crashing the IME tracking.

**BEFORE:**
```smali
.method private F()Ljava/lang/Object;
    .locals 1

    const-string v0, "mCurInputContext"

    invoke-virtual {p0, v0}, Lcom/jozein/xedgepro/xposed/j2;->u(Ljava/lang/String;)Ljava/lang/Object;

    move-result-object v0

    if-eqz v0, :cond_0

    return-object v0

    :cond_0
    new-instance v0, Lcom/jozein/xedgepro/xposed/p0$b;

    invoke-direct {v0}, Lcom/jozein/xedgepro/xposed/p0$b;-><init>()V

    throw v0
.end method
```

**AFTER:**
```smali
.method private F()Ljava/lang/Object;
    .locals 1

    :try_start_0
    const-string v0, "mCurInputContext"

    invoke-virtual {p0, v0}, Lcom/jozein/xedgepro/xposed/j2;->u(Ljava/lang/String;)Ljava/lang/Object;

    move-result-object v0
    :try_end_0
    .catchall {:try_start_0 .. :try_end_0} :catchall_0

    return-object v0

    :catchall_0
    const/4 v0, 0x0

    return-object v0
.end method
```