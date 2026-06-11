# Patch 4: Fix for False Full Screen Triggers

This patch addresses an issue where the `FullScreen` trigger would fire unexpectedly in two scenarios:
1. During window transitions (e.g., opening an app or returning to the launcher).
2. When pulling down the notification shade (Status Bar expansion) while on the home screen or inside a non-fullscreen app.

## The Bug

The issue stemmed from how the system's "Transient State" (temporary visibility of system bars) was detected and handled:

**1. The Null Comparison Bug (App Transitions):**
During window transitions, the WindowManager dynamically changes the `mControlTarget` (the window controlling the system bars). During this brief transition, both `mTransientControlTarget` and `mControlTarget` can evaluate to `null`. Our previous logic checked if `mTransientControlTarget == mControlTarget`. Since `null == null` evaluates to true in Smali, the code mistakenly believed the system bars were in a "Transient State."

**2. The Unconditional Masking Bug (Notification Shade):**
When the user pulls down the notification shade, the system temporarily assigns it as the `mTransientControlTarget`, putting the system into a legitimate "Transient State" (`mTransientState = true`). However, our logic blindly assumed that *any* transient state meant the bars were hidden to prevent false exits. Because of this, pulling the shade on the Home screen forced XEdgePro to pretend the natively visible bars were hidden, causing it to incorrectly evaluate `FullScreen = true`.

## Applied Changes

### 1. `smali/com/jozein/xedgepro/xposed/n0$m.smali` (Null Check Fix)
Injected a null-check (`if-eqz v0`) immediately before the equality comparison. If `mTransientControlTarget` is null, the state can never be transient, thus preventing the `null == null` false positive during transitions.

**BEFORE:**
```smali
    const-string v4, "mControlTarget"
    invoke-static {v5, v4}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v4
    if-ne v0, v4, :cond_not_transient
```

**AFTER:**
```smali
    const-string v4, "mControlTarget"
    invoke-static {v5, v4}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v4
    if-eqz v0, :cond_not_transient
    if-ne v0, v4, :cond_not_transient
```

### 2. `smali/com/jozein/xedgepro/xposed/n0.smali` (Conditional Transient Masking)
Modified the `B0` (Status Bar update) and `y0` (Navigation Bar update) methods. The transient mask is now conditional: it will **only** pretend the bars are hidden if the device was **already** in Full-Screen mode (`S == true`). If the device is not in Full-Screen mode (e.g., Home screen), pulling the notification shade still triggers a transient state, but it is ignored for visibility masking.

**BEFORE:**
```smali
    :cond_update_fs
    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z
    if-eqz v0, :cond_check_transient
    iget-boolean v1, p0, Lcom/jozein/xedgepro/xposed/n0;->mTransientState:Z
    if-eqz v1, :cond_check_transient
    const/4 v0, 0x0
    :cond_check_transient
```

**AFTER:**
```smali
    :cond_update_fs
    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z
    if-eqz v0, :cond_check_transient
    iget-boolean v1, p0, Lcom/jozein/xedgepro/xposed/n0;->mTransientState:Z
    if-eqz v1, :cond_check_transient
    iget-boolean v1, p0, Lcom/jozein/xedgepro/xposed/n0;->S:Z
    if-eqz v1, :cond_check_transient
    const/4 v0, 0x0
    :cond_check_transient
```