# Patch 8: Fix Full Screenshot Execution via AccessibilityService

## Problem Description
Inconsistent behavior was observed for the full screenshot action. The action was executed successfully when triggered via a launcher shortcut, but a silent failure occurred when it was triggered via an in-app gesture or broadcast.

Upon failure of the reflection call, a `try-catch` fallback mechanism was originally triggered. This fallback simulated a hardware `Print Screen` keypress (`KEYCODE_SYSRQ` / `120`). 
* When a launcher shortcut was used, the injected key was processed successfully by the OS because the screen was no longer being touched.
* When an in-app gesture was used, the injected key event was dropped by the Android InputDispatcher to prevent input conflicts, because the screen was actively being touched during the swipe.

## Proposed Solution
The deprecated reflection method (`handleScreenShot`) and the unreliable key injection fallback (`Print Screen`) are completely replaced. A native, reliable, and public API for taking screenshots via Accessibility Services is utilized for Android 9.0 (API 28) and above.

The full screenshot branch within the `D4(I)V` method is rewritten. The module's active `AccessibilityService` instance (`v3.smali`) is fetched, and `performGlobalAction(9)` (`GLOBAL_ACTION_TAKE_SCREENSHOT`) is invoked. This native approach is immune to active touch states and is executed flawlessly across all invocation methods.

## Implementation Details

### Step 1: Replace Full Screenshot Logic in `D4(I)V`
The reflection-based code block responsible for full screenshots is removed. It is replaced with logic that calls `performGlobalAction(9)` on the `AccessibilityService`. The legacy `m2(120)` fallback method is retained purely as a safety net in case the accessibility service is unavailable.

**File:** `smali/com/jozein/xedgepro/xposed/w1.smali`
**Action:** Replace block within `.method private D4(I)V`

**Before:**
```smali
    :cond_do_full_screenshot
    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I

    const/4 v1, 0x1

    const/4 v2, 0x0

    const/16 v3, 0x21

    if-lt v0, v3, :cond_0

    const/4 v0, 0x2

    new-array v3, v0, [Ljava/lang/Object;

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object p1

    aput-object p1, v3, v2

    invoke-static {v0}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object p1

    aput-object p1, v3, v1

    const-string p1, "handleScreenShot"

    invoke-virtual {p0, p1, v3}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;

    goto :goto_0

    :cond_0
    const/16 v3, 0x1b

    if-gt v0, v3, :cond_1

    new-array v0, v1, [Ljava/lang/Object;

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object p1

    aput-object p1, v0, v2

    const-string p1, "takeScreenshot"

    invoke-virtual {p0, p1, v0}, Lcom/jozein/xedgepro/xposed/j2;->r(Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;

    goto :goto_0

    :cond_1
    const-string v0, "mScreenshotRunnable"

    invoke-virtual {p0, v0}, Lcom/jozein/xedgepro/xposed/j2;->u(Ljava/lang/String;)Ljava/lang/Object;

    move-result-object v0

    check-cast v0, Ljava/lang/Runnable;

    new-array v1, v1, [Ljava/lang/Object;

    invoke-static {p1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;

    move-result-object p1

    aput-object p1, v1, v2

    const-string p1, "setScreenshotType"

    invoke-static {v0, p1, v1}, Lde/robv/android/xposed/XposedHelpers;->callMethod(Ljava/lang/Object;Ljava/lang/String;[Ljava/lang/Object;)Ljava/lang/Object;

    invoke-interface {v0}, Ljava/lang/Runnable;->run()V

    :goto_0
```

**After:**
```smali
    :cond_do_full_screenshot
    sget v0, Landroid/os/Build$VERSION;->SDK_INT:I

    const/16 v1, 0x1c

    if-lt v0, v1, :fallback_screenshot

    invoke-direct {p0}, Lcom/jozein/xedgepro/xposed/w1;->P1()Lcom/jozein/xedgepro/xposed/v3;

    move-result-object v0

    if-eqz v0, :fallback_screenshot

    const/16 p1, 0x9

    invoke-virtual {v0, p1}, Landroid/accessibilityservice/AccessibilityService;->performGlobalAction(I)Z

    goto :goto_0

    :fallback_screenshot
    const/16 p1, 0x78

    invoke-direct {p0, p1}, Lcom/jozein/xedgepro/xposed/w1;->m2(I)V

    :goto_0
```
