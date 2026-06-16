# ADB 快速验证 Flag 能力指南

## 概述

本文档提供通过 ADB 命令快速验证设备上 aconfig flag 状态的方法，适用于 CTS 测试失败时的快速诊断。

---

## 一、检查 Flag 服务是否可用

### 1.1 查看 flag 相关服务

```bash
# 检查有哪些 flag 相关服务
adb shell service list | grep -iE "flag|config|aconfig"
```

**正常设备应该看到：**
```
device_config: []
feature_flags: [android.flags.IFeatureFlags]   ← Android 15+ 才有
```

### 1.2 确认使用哪个命令

```bash
# 新版命名（Android 15+）
adb shell cmd flag help 2>&1

# 旧版命名（Android 14 及以下）
adb shell cmd device_config help 2>&1
```

**判断规则：**
- 如果 `cmd flag help` 成功 → 使用 `cmd flag`
- 如果 `cmd device_config help` 成功 → 使用 `cmd device_config`
- 如果都失败 → Flag 服务异常

---

## 二、读取 Flag 值

### 2.1 读取单个 flag

```bash
# 语法：cmd <service> get <namespace> <flag_name>

# 读取 MediaRouter2 相关的 flag
adb shell cmd device_config get media_solutions enable_screen_off_scanning
adb shell cmd device_config get media_better_together enable_full_scan_with_media_content_control
```

**返回值说明：**
- `true` → flag 已启用
- `false` → flag 已禁用
- `null` → flag 未定义
- 报错 → 服务异常

### 2.2 列出某个 namespace 下所有 flag

```bash
adb shell cmd device_config list media_solutions
adb shell cmd device_config list media_better_together
```

### 2.3 列出所有 namespace

```bash
adb shell cmd device_config list_namespaces
```

### 2.4 列出所有 flag（可能很多）

```bash
adb shell cmd device_config list
```

---

## 三、直接读取 Flag 文件

### 3.1 检查 aconfig proto 文件

```bash
# 检查系统分区的 flag 文件
adb shell ls -la /system/etc/aconfig_flags.pb
adb shell ls -la /system_ext/etc/aconfig_flags.pb
adb shell ls -la /product/etc/aconfig_flags.pb
adb shell ls -la /vendor/etc/aconfig_flags.pb

# 检查 APEX 中的 flag 文件
adb shell ls -la /apex/*/etc/aconfig_flags/ 2>/dev/null

# 检查运行时 override 文件
adb shell ls -la /data/misc/aconfig/ 2>/dev/null
```

### 3.2 查看文件内容（粗略）

```bash
# 用 strings 命令查看 flag 名称
adb shell strings /system/etc/aconfig_flags.pb | grep -i "screen_off\|media_content"

# 用 hexdump 查看（需要 root）
adb shell hexdump -C /system/etc/aconfig_flags.pb | head -50
```

---

## 四、通过 dumpsys 检查

### 4.1 dump 服务状态

```bash
# dump device_config 服务
adb shell dumpsys device_config

# dump feature_flags 服务（如果有）
adb shell dumpsys feature_flags
```

### 4.2 搜索特定 flag

```bash
# 搜索 MediaRouter2 相关 flag
adb shell dumpsys device_config | grep -i "screen_off_scanning"
adb shell dumpsys device_config | grep -i "full_scan_with_media"

# 查看所有 namespace
adb shell dumpsys device_config | grep "namespace"
```

---

## 五、检查 Flag 对应的功能是否生效

### 5.1 检查 MediaRouter2 功能

```bash
# 检查 MediaRouter2 是否支持息屏扫描 API
adb shell cmd media.router help 2>&1 | grep -i scan

# 检查 MediaRouterService 状态
adb shell dumpsys media.router | head -50

# 检查相关权限
adb shell dumpsys package android.media.router.cts.bluetoothpermissionsapp | grep -i "MEDIA_CONTENT_CONTROL"

# 检查设备是否支持相关 feature
adb shell pm list features | grep -i "media\|router\|leanback"
```

### 5.2 检查其他常见 flag 功能

```bash
# 检查 WindowManager 相关 flag
adb shell cmd device_config list window_manager_native_boot

# 检查 SystemUI 相关 flag
adb shell cmd device_config list systemui

# 检查 ActivityManager 相关 flag
adb shell cmd device_config list activity_manager_native_boot
```

---

## 六、一键验证脚本

### 6.1 基础检查脚本

```bash
#!/bin/bash
# flag_check.sh - 快速检查设备 flag 能力

echo "===== 1. 检查 flag 服务 ====="
SERVICES=$(adb shell service list 2>/dev/null | grep -iE "flag|config")
echo "$SERVICES"

if echo "$SERVICES" | grep -q "device_config"; then
    CMD="device_config"
elif echo "$SERVICES" | grep -q "feature_flags"; then
    CMD="feature_flags"
else
    echo "❌ 没有找到 flag 相关服务"
    exit 1
fi

echo ""
echo "===== 2. 使用命令: cmd $CMD ====="

echo ""
echo "===== 3. 检查 MediaRouter2 相关 flag ====="
echo "--- enable_screen_off_scanning ---"
adb shell cmd $CMD get media_solutions enable_screen_off_scanning 2>&1

echo "--- enable_full_scan_with_media_content_control ---"
adb shell cmd $CMD get media_better_together enable_full_scan_with_media_content_control 2>&1

echo ""
echo "===== 4. 检查 aconfig 文件 ====="
adb shell ls -la /system/etc/aconfig_flags.pb 2>&1
adb shell ls -la /vendor/etc/aconfig_flags.pb 2>&1

echo ""
echo "===== 5. 检查 namespace 列表 ====="
adb shell cmd $CMD list_namespaces 2>&1 | grep -iE "media|router"

echo ""
echo "===== 完成 ====="
```

**使用方法：**
```bash
chmod +x flag_check.sh
./flag_check.sh
```

### 6.2 完整诊断脚本

```bash
#!/bin/bash
# flag_diagnose.sh - 完整的 flag 诊断脚本

set -e

echo "╔════════════════════════════════════════════════════════════╗"
echo "║         ADB Flag 能力诊断工具                              ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""

# 检查设备连接
echo "【1/7】检查设备连接..."
if ! adb devices | grep -q "device$"; then
    echo "❌ 设备未连接"
    exit 1
fi
echo "✅ 设备已连接"
echo ""

# 检查服务
echo "【2/7】检查 flag 相关服务..."
SERVICES=$(adb shell service list 2>/dev/null | grep -iE "flag|config")
if [ -z "$SERVICES" ]; then
    echo "❌ 没有找到 flag 相关服务"
    exit 1
fi
echo "$SERVICES"
echo ""

# 确定命令
if echo "$SERVICES" | grep -q "device_config"; then
    CMD="device_config"
elif echo "$SERVICES" | grep -q "feature_flags"; then
    CMD="feature_flags"
else
    echo "❌ 无法确定使用哪个命令"
    exit 1
fi
echo "✅ 使用命令: cmd $CMD"
echo ""

# 检查文件
echo "【3/7】检查 aconfig 文件..."
for file in /system/etc/aconfig_flags.pb /vendor/etc/aconfig_flags.pb; do
    if adb shell ls -la "$file" 2>/dev/null | grep -q "$file"; then
        echo "✅ $file 存在"
    else
        echo "⚠️  $file 不存在"
    fi
done
echo ""

# 检查 namespace
echo "【4/7】检查 media 相关 namespace..."
adb shell cmd $CMD list_namespaces 2>&1 | grep -iE "media|router" || echo "⚠️  没有找到 media 相关 namespace"
echo ""

# 检查具体 flag
echo "【5/7】检查 MediaRouter2 相关 flag..."
FLAGS=(
    "media_solutions/enable_screen_off_scanning"
    "media_better_together/enable_full_scan_with_media_content_control"
)

for flag in "${FLAGS[@]}"; do
    NS=$(echo $flag | cut -d'/' -f1)
    NAME=$(echo $flag | cut -d'/' -f2)
    VALUE=$(adb shell cmd $CMD get "$NS" "$NAME" 2>&1)
    echo "  $flag = $VALUE"
done
echo ""

# 检查功能
echo "【6/7】检查 MediaRouter2 功能..."
if adb shell dumpsys media.router 2>/dev/null | grep -q "MediaRouterService"; then
    echo "✅ MediaRouterService 运行中"
else
    echo "⚠️  MediaRouterService 状态异常"
fi
echo ""

# 总结
echo "【7/7】诊断总结..."
echo "════════════════════════════════════════════════════════════"
echo "设备 flag 能力诊断完成。"
echo ""
echo "如果所有检查都通过，但 CTS 仍然失败，可能是："
echo "  1. CTS 版本与设备版本不匹配"
echo "  2. 特定 flag 的值不符合测试要求"
echo "  3. 其他系统配置问题"
echo "════════════════════════════════════════════════════════════"
```

**使用方法：**
```bash
chmod +x flag_diagnose.sh
./flag_diagnose.sh
```

---

## 七、快速判断表

| 命令 | 预期结果 | 异常含义 |
|---|---|---|
| `cmd flag help` | 显示帮助信息 | 服务不存在 → 用 `cmd device_config` |
| `cmd device_config get <ns> <flag>` | `true` 或 `false` | `null` → flag 未定义 |
| `cmd device_config list_namespaces` | 列出所有 namespace | 空 → aconfig 系统未初始化 |
| `ls /system/etc/aconfig_flags.pb` | 文件存在 | 不存在 → flag 文件未打包 |
| `dumpsys device_config` | 显示所有 flag 状态 | 空 → 服务异常 |
| `dumpsys media.router` | 显示 MediaRouter 状态 | 空 → MediaRouterService 异常 |

---

## 八、常见问题排查

### 8.1 `cmd: Can't find service: flag`

**原因：** 设备上没有 `flag` 服务

**解决：**
```bash
# 使用旧版命令
adb shell cmd device_config get <namespace> <flag_name>
```

### 8.2 flag 返回 `null`

**原因：** flag 未定义或 namespace 不存在

**排查：**
```bash
# 检查 namespace 是否存在
adb shell cmd device_config list_namespaces | grep <namespace>

# 检查该 namespace 下有哪些 flag
adb shell cmd device_config list <namespace>
```

### 8.3 aconfig 文件不存在

**原因：** 系统编译时未包含 flag 文件

**排查：**
```bash
# 检查所有可能的路径
adb shell find / -name "aconfig_flags.pb" 2>/dev/null

# 检查 APEX 模块
adb shell ls /apex/*/etc/aconfig_flags/ 2>/dev/null
```

### 8.4 flag 值为 false，但测试期望为 true

**原因：** flag 被禁用

**解决：**
```bash
# 临时启用 flag（需要 root）
adb shell cmd device_config put <namespace> <flag_name> true

# 或者通过 override
adb shell cmd device_config override <namespace> <flag_name> true
```

---

## 九、针对 CTS 失败的快速诊断流程

```
CTS 测试失败
    │
    ├─ 看错误堆栈
    │   ├─ FlagReadException → flag 读取失败
    │   │   │
    │   │   ├─ 执行：adb shell cmd flag help
    │   │   │   ├─ 成功 → flag 服务正常，检查具体 flag 值
    │   │   │   └─ 失败 → 用 cmd device_config 替代
    │   │   │
    │   │   └─ 执行：adb shell cmd device_config get <ns> <flag>
    │   │       ├─ 返回 true/false → flag 存在，检查值是否正确
    │   │       ├─ 返回 null → flag 未定义
    │   │       └─ 报错 → 服务异常
    │   │
    │   └─ 其他异常 → 参考其他诊断文档
    │
    └─ 执行完整诊断脚本
        └─ ./flag_diagnose.sh
```

---

## 十、参考资源

- [AOSP aconfig 文档](https://source.android.com/docs/core/architecture/modular-system/aconfig)
- [DeviceConfig API](https://developer.android.com/reference/android/provider/DeviceConfig)
- [CTS 测试框架](https://source.android.com/docs/compatibility/cts)

---

*文档创建时间：2026-06-16*  
*适用于 AOSP Android 14/15*
