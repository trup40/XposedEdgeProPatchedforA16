# Patch 3: Decoupling Bar Visibility from Full Screen Triggers

This patch resolves an issue where the "Status Bar Shown" and "Navigation Bar Shown" triggers were delayed. The delay occurred because the visibility state was being artificially masked as `false` during transient events. The logic was decoupled so that specific bar triggers fire immediately based on true visibility, while the "Full Screen Exit" logic selectively uses the transient state to prevent false exits.

## Applied Changes

### 1. Addition of Transient State Field (`smali/com/jozein/xedgepro/xposed/n0.smali`)
A new public boolean field was introduced to maintain the transient status independently from the visibility state.

**BEFORE:**
```smali
.field private f0:I

.field private final g0:Ljava/lang/Runnable;
```

**AFTER:**
```smali
.field private f0:I

.field private final g0:Ljava/lang/Runnable;

.field public mTransientState:Z
```

### 2. State Export Refactoring (`smali/com/jozein/xedgepro/xposed/n0$m.smali`)
The transient detection logic was modified. Instead of overwriting the true visibility value (`v3`) with `0` (false), the true visibility is now preserved. The transient state is evaluated into a separate register (`v4`) and directly assigned to the newly created `mTransientState` field.

**BEFORE:**
```smali
    :cond_4
    iget-object v3, p0, Lcom/jozein/xedgepro/xposed/n0$m;->d:Ljava/lang/reflect/Field;
    invoke-virtual {v3, v1}, Ljava/lang/reflect/Field;->getBoolean(Ljava/lang/Object;)Z
    move-result v3

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
    const/4 v3, 0x0
    :try_end_transient
    .catchall {:try_start_transient .. :try_end_transient} :catch_transient
    goto :cond_transient_done
    :catch_transient
    nop
    :cond_transient_done
```

**AFTER:**
```smali
    :cond_4
    iget-object v3, p0, Lcom/jozein/xedgepro/xposed/n0$m;->d:Ljava/lang/reflect/Field;
    invoke-virtual {v3, v1}, Ljava/lang/reflect/Field;->getBoolean(Ljava/lang/Object;)Z
    move-result v3

    const/4 v4, 0x0

    if-eqz v3, :cond_transient_done
    :try_start_transient
    const-string v5, "mInsetsPolicy"
    invoke-static {v0, v5}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v0
    const-string v5, "mTransientControlTarget"
    invoke-static {v0, v5}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v0
    iget-object v5, p1, Lde/robv/android/xposed/XC_MethodHook$MethodHookParam;->thisObject:Ljava/lang/Object;
    const-string v4, "mControlTarget"
    invoke-static {v5, v4}, Lde/robv/android/xposed/XposedHelpers;->getObjectField(Ljava/lang/Object;Ljava/lang/String;)Ljava/lang/Object;
    move-result-object v4
    if-ne v0, v4, :cond_not_transient
    const/4 v4, 0x1
    goto :cond_transient_done
    :cond_not_transient
    const/4 v4, 0x0
    :try_end_transient
    .catchall {:try_start_transient .. :try_end_transient} :catch_transient
    goto :cond_transient_done
    :catch_transient
    const/4 v4, 0x0
    :cond_transient_done
    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0$m;->f:Lcom/jozein/xedgepro/xposed/n0;
    iput-boolean v4, v0, Lcom/jozein/xedgepro/xposed/n0;->mTransientState:Z
```

### 3. Decoupling Logic in Update Methods (`smali/com/jozein/xedgepro/xposed/n0.smali`)
The `B0(Z)` (Status Bar) and `y0(Z)` (Navigation Bar) methods were rewritten. The true visibility is immediately passed to the individual bar triggers. The `mTransientState` field is then evaluated to conditionally block the "Full Screen Exit" trigger if the bars are only shown transiently.

**BEFORE (`B0(Z)`):**
```smali
.method private B0(Z)V
    .locals 1

    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z

    if-eq v0, p1, :cond_1

    iput-boolean p1, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z

    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0;->H:Lcom/jozein/xedgepro/xposed/w1;

    invoke-virtual {v0, p1}, Lcom/jozein/xedgepro/xposed/w1;->o3(Z)V

    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->R:Z

    if-nez v0, :cond_0

    if-nez p1, :cond_0

    const/4 p1, 0x1

    goto :goto_0

    :cond_0
    const/4 p1, 0x0

    :goto_0
    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->S:Z

    if-eq v0, p1, :cond_1

    iput-boolean p1, p0, Lcom/jozein/xedgepro/xposed/n0;->S:Z

    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0;->H:Lcom/jozein/xedgepro/xposed/w1;

    invoke-virtual {v0, p1}, Lcom/jozein/xedgepro/xposed/w1;->g3(Z)V

    :cond_1
    return-void
.end method
```

**AFTER (`B0(Z)`):**
```smali
.method private B0(Z)V
    .locals 3

    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z

    if-eq v0, p1, :cond_update_fs

    iput-boolean p1, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z

    iget-object v0, p0, Lcom/jozein/xedgepro/xposed/n0;->H:Lcom/jozein/xedgepro/xposed/w1;

    invoke-virtual {v0, p1}, Lcom/jozein/xedgepro/xposed/w1;->o3(Z)V

    :cond_update_fs
    iget-boolean v0, p0, Lcom/jozein/xedgepro/xposed/n0;->P:Z

    if-eqz v0, :cond_check_transient

    iget-boolean v1, p0, Lcom/jozein/xedgepro/xposed/n0;->mTransientState:Z

    if-eqz v1, :cond_check_transient

    const/4 v0, 0x0

    :cond_check_transient
    iget-boolean v1, p0, Lcom/jozein/xedgepro/xposed/n0;->R:Z

    if-eqz v1, :cond_check_transient2

    iget-boolean v2, p0, Lcom/jozein/xedgepro/xposed/n0;->mTransientState:Z

    if-eqz v2, :cond_check_transient2

    const/4 v1, 0x0

    :cond_check_transient2
    const/4 v2, 0x1

    if-nez v1, :cond_not_fs

    if-nez v0, :cond_not_fs

    goto :cond_apply_fs

    :cond_not_fs
    const/4 v2, 0x0

    :cond_apply_fs
    iget-boolean v1, p0, Lcom/jozein/xedgepro/xposed/n0;->S:Z

    if-eq v1, v2, :cond_end

    iput-boolean v2, p0, Lcom/jozein/xedgepro/xposed/n0;->S:Z

    iget-object v1, p0, Lcom/jozein/xedgepro/xposed/n0;->H:Lcom/jozein/xedgepro/xposed/w1;

    invoke-virtual {v1, v2}, Lcom/jozein/xedgepro/xposed/w1;->g3(Z)V

    :cond_end
    return-void
.end method
```
*(The identical structural logic was applied to `y0(Z)V` for the Navigation Bar update.)*
