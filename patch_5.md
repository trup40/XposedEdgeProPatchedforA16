# Patch 5: Fix for Broken Partial Screenshot on Android 14+

This patch addresses the issue where the Partial Screenshot feature silently fails on newer Android versions (specifically Android 14 and above).

## The Bug

The issue originates from Google removing or heavily modifying the native partial screenshot UI components in Android 14+. 

When XEdgePro intercepts a partial screenshot trigger (type `2`) and forwards it to the system's `handleScreenShot` method, the system silently drops the request because the underlying system activity is either missing or heavily restricted. Furthermore, attempting to bypass this by executing a root shell command (`su -c am start...`) directly from within `w1.smali` silently fails due to strict SELinux policies (`neverallow` rules prevent the `SystemUI` background process from executing shell commands).

## Applied Changes

### `smali/com/jozein/xedgepro/xposed/w1.smali` (Native Intent Injection)
Injected a conditional check at the very beginning of the `D4(I)V` method. 
- If the requested screenshot type is `1` (Full Screenshot), it bypasses our injected code and gracefully falls back to the original system behavior.
- If the requested screenshot type is `2` (Partial Screenshot), it completely intercepts the broken system method. Instead, it grabs the `SystemUI`'s active Application Context and uses a native Android `Intent` to launch our standalone, lightweight helper application (`com.trup40.xedgecrop`). This perfectly bypasses SELinux restrictions by utilizing standard Java APIs rather than shell execution. Added robust `try-catch` blocks with `Log.e` for safe failure handling to prevent `SystemUI` crashes.

**BEFORE:**
```smali
.method private D4(I)V
    .locals 4

    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I

    const/4 v1, 0x1

    const/4 v2, 0x0

    const/16 v3, 0x21
    
    # ... (Proceeds directly to original system screenshot handling)
```

**AFTER:**
```smali
.method private D4(I)V
    .locals 4

    const/4 v0, 0x2
    if-ne p1, v0, :cond_do_full_screenshot

    const-string v1, "XEdgeCrop"
    const-string v2, "Partial screenshot requested. Dispatching helper intent..."
    invoke-static {v1, v2}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I

    :try_start_partial
    invoke-static {}, Landroid/app/ActivityThread;->currentApplication()Landroid/app/Application;
    move-result-object v0

    if-nez v0, :continue_intent
    goto :end_partial

    :continue_intent
    new-instance v1, Landroid/content/Intent;
    invoke-direct {v1}, Landroid/content/Intent;-><init>()V

    const-string v2, "com.trup40.xedgecrop"
    const-string v3, "com.trup40.xedgecrop.MainActivity"
    invoke-virtual {v1, v2, v3}, Landroid/content/Intent;->setClassName(Ljava/lang/String;Ljava/lang/String;)Landroid/content/Intent;

    const/high16 v2, 0x10000000
    invoke-virtual {v1, v2}, Landroid/content/Intent;->addFlags(I)Landroid/content/Intent;

    invoke-virtual {v0, v1}, Landroid/content/Context;->startActivity(Landroid/content/Intent;)V
    :try_end_partial
    .catch Ljava/lang/Exception; {:try_start_partial .. :try_end_partial} :catch_partial

    goto :end_partial

    :catch_partial
    move-exception v0
    const-string v1, "XEdgeCrop"
    const-string v2, "Helper execution failed!"
    invoke-static {v1, v2, v0}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;Ljava/lang/Throwable;)I

    :end_partial
    return-void

    :cond_do_full_screenshot
    # --- ORIGINAL FULL SCREENSHOT CODE CONTINUES HERE ---
    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I

    const/4 v1, 0x1

    const/4 v2, 0x0

    const/16 v3, 0x21
    # ...
```