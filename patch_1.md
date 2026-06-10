# Patch 1: Fix for Full Screen Triggers

## Applied Changes

### 1. `smali/com/jozein/xedgepro/xposed/n0$m.smali` (Type Mapping Fix)
We updated the `.locals` count to accommodate a new variable and injected a version check block right after fetching the `mType` value. If the device is running Android 14+ (API 34 / `0x22`), the new type values (`1` and `2`) are mapped back to the old expected values (`0` and `1`).

**BEFORE:**
```smali
.method protected afterHookedMethod(Lde/robv/android/xposed/XC_MethodHook$MethodHookParam;)V
    .locals 2

    # ... (other code) ...

    :cond_4
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->c:Ljava/lang/reflect/Field;

    invoke-virtual {v0, p1}, Ljava/lang/reflect/Field;->getInt(Ljava/lang/Object;)I

    move-result v0

    if-nez v0, :cond_5

    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->f:Lcom/jozein/xedgepro/xposed/n0;
    
    # ... (rest of the method) ...
```

**AFTER:**
```smali
.method protected afterHookedMethod(Lde/robv/android/xposed/XC_MethodHook$MethodHookParam;)V
    .locals 3

    # ... (other code) ...

    :cond_4
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->c:Ljava/lang/reflect/Field;

    invoke-virtual {v0, p1}, Ljava/lang/reflect/Field;->getInt(Ljava/lang/Object;)I

    move-result v0

    sget v1, Landroid/os/Build$VERSION;->SDK_INT:I

    const/16 v2, 0x22

    if-lt v1, v2, :cond_patch_old

    const/4 v1, 0x1

    if-ne v0, v1, :cond_patch_nav

    const/4 v0, 0x0

    goto :cond_patch_old

    :cond_patch_nav
    const/4 v1, 0x2

    if-ne v0, v1, :cond_patch_done

    const/4 v0, 0x1

    goto :cond_patch_old

    :cond_patch_done
    goto :goto_0

    :cond_patch_old
    if-nez v0, :cond_5

    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->f:Lcom/jozein/xedgepro/xposed/n0;

    # ... (rest of the method) ...
```

### 2. `smali/com/jozein/xedgepro/xposed/n0.smali` (Hook Signature Fix)
Because the parameters of the `updateVisibility` method changed, `findAndHookMethod` was failing to attach the hook. We replaced it with `hookAllMethods` to ensure the hook catches the method regardless of its parameter variations on newer Android versions.

**BEFORE:**
```smali
    const-string p3, "updateVisibility"

    new-array v1, v2, [Ljava/lang/Object;

    new-instance v4, Lcom/jozein/xedgepro/xposed/n0$m;

    invoke-direct {v4, p0, p2}, Lcom/jozein/xedgepro/xposed/n0$m;-><init>(Lcom/jozein/xedgepro/xposed/n0;Ljava/lang/Class;)V

    aput-object v4, v1, p1

    invoke-static {p2, p3, v1}, Lde/robv/android/xposed/XposedHelpers;->findAndHookMethod(Ljava/lang/Class;Ljava/lang/String;[Ljava/lang/Object;)Lde/robv/android/xposed/XC_MethodHook$Unhook;
```

**AFTER:**
```smali
    const-string p3, "updateVisibility"

    new-instance v4, Lcom/jozein/xedgepro/xposed/n0$m;

    invoke-direct {v4, p0, p2}, Lcom/jozein/xedgepro/xposed/n0$m;-><init>(Lcom/jozein/xedgepro/xposed/n0;Ljava/lang/Class;)V

    invoke-static {p2, p3, v4}, Lde/robv/android/xposed/XposedBridge;->hookAllMethods(Ljava/lang/Class;Ljava/lang/String;Lde/robv/android/xposed/XC_MethodHook;)Ljava/util/Set;
```