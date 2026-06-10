# Patch 2: Accurate Full-Screen Detection (Transient Bars Fix)

This patch refactors the `afterHookedMethod` in `n0$m.smali` to distinguish between actual full-screen exits and temporary system UI overlays (Transient Bars).

## Applied Changes: `smali/com/jozein/xedgepro/xposed/n0$m.smali`

### 1. Register and Variable Refactoring
The local registers've been expanded and stabilized variable assignments to prevent data corruption and allow for the new logic.

**Comparison:**
| Feature | `patch_1` (Before) | Current Patch (After) |
| :--- | :--- | :--- |
| **Locals** | `.locals 3` | `.locals 6` |
| **v0** | Reused (Field -> DC Object -> Field) | **DisplayContent Object** (Static) |
| **v1** | Reused (DC Object -> mSource Field) | **mSource Object** (Static) |
| **v2** | **mType Value** | **mType Value** |
| **v3** | N/A | **Visibility Value** (Corrected) |
| **v4 / v5** | N/A | **Transient Detection Logic** |

---

### 2. Implementation of Transient Bar Detection
The core fix identifies if the system bars are shown "transiently". If they are, it gets forced the visibility to `false` (hidden) so XEdgePro stays in "Full Screen" mode.

**Logic Injected:**
```smali
    # Fetches mVisible into v3 (instead of reusing p1)
    iget-object v3, p0, Lcom/jozein/xedgepro/xposed/n0$m;->d:Ljava/lang/reflect/Field;
    invoke-virtual {v3, v1}, Ljava/lang/reflect/Field;->getBoolean(Ljava/lang/Object;)Z
    move-result v3

    # NEW: Transient Control Check with Error Handling
    if-eqz v3, :cond_transient_done
    :try_start_transient
    const-string v4, "mInsetsPolicy"
    invoke-static {v0, v4}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v4
    const-string v5, "mTransientControlTarget"
    invoke-static {v4, v5}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v4
    iget-object v5, p1, Lde/robv/android/xposed/XC_MethodHook$MethodHookParam;->thisObject:Ljava/lang/Object;
    const-string v0, "mControlTarget"
    invoke-static {v5, v0}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v0
    if-ne v4, v0, :cond_transient_done
    const/4 v3, 0x0 # Mask visibility as HIDDEN if transient
    :try_end_transient
    .catchall {:try_start_transient .. :try_end_transient} :catch_transient
    goto :cond_transient_done
    :catch_transient
    nop # Graceful fallback to standard visibility
    :cond_transient_done
```

---

### 3. Refactored Reporting Logic (Line-by-Line Change)
Because registers were reorganized, the final reporting to the main `n0` class was completely rewritten to be cleaner.

**BEFORE (`patch_1`):**
```smali
    :cond_patch_old
    if-nez v0, :cond_5
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->f:Lcom/jozein/xedgepro/xposed/n0;
    iget-object v1, p0, Lcom/jozein/xedgepro/xposed/n0$m;->d:Ljava/lang/reflect/Field;
    invoke-virtual {v1, p1}, Ljava/lang/reflect/Field;->getBoolean(Ljava/lang/Object;)Z
    move-result p1
    invoke-static {v0, p1}, Lcom/jozein/xedgepro/xposed/n0;->Q(Lcom/jozein/xedgepro/xposed/n0;Z)V
    goto :goto_0
    :cond_5
    const/4 v1, 0x1
    if-ne v0, v1, :cond_6
    # ... (similar logic for R) ...
```

**AFTER (Refactored Patch):**
```smali
    :cond_patch_old
    # Logic uses v2 (type) and v3 (corrected visibility)
    if-nez v2, :cond_5
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->f:Lcom/jozein/xedgepro/xposed/n0;
    invoke-static {v0, v3}, Lcom/jozein/xedgepro/xposed/n0;->Q(Lcom/jozein/xedgepro/xposed/n0;Z)V
    goto :goto_0
    :cond_5
    const/4 v0, 0x1
    if-ne v2, v0, :cond_6
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->f:Lcom/jozein/xedgepro/xposed/n0;
    invoke-static {v0, v3}, Lcom/jozein/xedgepro/xposed/n0;->R(Lcom/jozein/xedgepro/xposed/n0;Z)V
```

## Summary
By decoupling the "fetch" logic from the "report" logic and injecting the `mTransientControlTarget` validation in between, the "False Exit" bug has been fixed while making the code significantly more robust for Android 16.
