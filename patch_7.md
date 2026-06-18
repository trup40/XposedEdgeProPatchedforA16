# Patch 7: 3-Stage Fallback Mechanism for freezeRotation (Android 16 / ZUXOS)

## The Bug
On certain Android 16 devices, the traditional `WindowManagerService#freezeRotation(int)` method has been restricted or hidden. Furthermore, the newer `freezeDisplayRotation(int displayId, int rotation)` method introduced in Android 11 has had its signature altered in Android 15+ to include a caller string: `freezeDisplayRotation(int displayId, int rotation, String caller)`. When the XEdgePro module attempts to invoke these via reflection without providing exact parameter matches, a `NoSuchMethodError` is caused.

## The Solution
The `D(I)V` method in `smali/com/jozein/xedgepro/xposed/m2.smali` has been updated with a highly robust 3-stage fallback strategy. 
1. **First Attempt:** `freezeRotation(int)` is invoked (preserving compatibility with older Android versions).
2. **Second Attempt:** If the first attempt throws an exception, `freezeDisplayRotation(0, int)` is called (using `0` as the default `displayId` for standard Android 11-14).
3. **Third Attempt:** If the second attempt fails, `freezeDisplayRotation(0, int, "XEdgePro")` is called to fulfill the new 3-parameter signature requirement introduced in Android 15+.
4. **Error Logging:** If all attempts fail, the final exception is cleanly caught and logged using the module's own `f.v.d()` logger. A system crash is completely prevented.

---

### File: `smali/com/jozein/xedgepro/xposed/m2.smali`
**Method:** `.method D(I)V`

#### BEFORE
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

#### AFTER
```smali
.method D(I)V
    .locals 4

    :try_start_0
    const/4 v0, 0x1

    new-array v0, v0, [Ljava/lang/Object;

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object v1

    const/4 v2, 0x0

    aput-object v1, v0, v2

    const-string v1, "freezeRotation"

    invoke-virtual {p0, v1, v0}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;
    :try_end_0
    .catchall {:try_start_0 .. :try_end_0} :catchall_0

    goto :goto_0

    :catchall_0
    move-exception v0

    :try_start_1
    const/4 v0, 0x2

    new-array v0, v0, [Ljava/lang/Object;

    const/4 v1, 0x0

    invoke-static {v1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object v2

    aput-object v2, v0, v1

    const/4 v1, 0x1

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object v2

    aput-object v2, v0, v1

    const-string v1, "freezeDisplayRotation"

    invoke-virtual {p0, v1, v0}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;
    :try_end_1
    .catchall {:try_start_1 .. :try_end_1} :catchall_1

    goto :goto_0

    :catchall_1
    move-exception v0

    :try_start_2
    const/4 v0, 0x3

    new-array v0, v0, [Ljava/lang/Object;

    const/4 v1, 0x0

    invoke-static {v1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object v2

    aput-object v2, v0, v1

    const/4 v1, 0x1

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object v2

    aput-object v2, v0, v1

    const/4 v1, 0x2

    const-string v2, "XEdgePro"

    aput-object v2, v0, v1

    const-string v1, "freezeDisplayRotation"

    invoke-virtual {p0, v1, v0}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;
    :try_end_2
    .catchall {:try_start_2 .. :try_end_2} :catchall_2

    goto :goto_0

    :catchall_2
    move-exception v0

    invoke-static {v0}, Lf/v;->d(Ljava/lang/Throwable;)V

    :goto_0
    return-void
.end method
```
