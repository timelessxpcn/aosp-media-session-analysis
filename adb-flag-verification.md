# ADB 快速验证 Flag 能力指南

## 概述

本文档提供通过 ADB 命令快速验证设备上 aconfig flag 状态的方法，适用于 CTS 测试失败时的快速诊断。

---

## 一、快速诊断流程（30 秒）

### 1.1 一键诊断脚本

```bash
#!/bin/bash
# 快速诊断 flag 状态

echo "===== 1. 检查 flag 服务 ====="
adb shell service list | grep -iE "flag|config"

echo ""
echo "===== 2. 检查 list 命令 ====="
adb shell cmd device_config list 2>&1 | grep -E "enable_screen_off_scanning|enable_full_scan_with_media_content_control"

echo ""
echo "===== 3. 检查 get 命令 ====="
echo "enable_screen_off_scanning: $(adb shell cmd device_config get media_solutions enable_screen_off_scanning 2>&1)"
echo "enable_full_scan_with_media_content_control: $(adb shell cmd device_config get media_better_together enable_full_scan_with_media_content_control 2>&1)"

echo ""
echo "===== 4. 诊断结论 ====="
LIST_COUNT=$(adb shell cmd device_config list 2>&1 | grep -c "enable_screen_off_scanning")
GET_RESULT=$(adb shell cmd device_config get media_solutions enable_screen_off_scanning 2>&1)

if [ "$LIST_COUNT" -gt 0 ] && [ "$GET_RESULT" = "null" ]; then
    echo "⚠️  Flag 定义存在（list 可见），但运行时未加载（get 返回 null）"
    echo ""
    echo "建议执行："
    echo "  adb shell cmd device_config put media_solutions enable_screen_off_scanning true"
    echo "  adb shell cmd device_config put media_better_together enable_full_scan_with_media_content_control true"
elif [ "$GET_RESULT" = "true" ] || [ "$GET_RESULT" = "false" ]; then
    echo "✅ Flag 运行时状态正常"
else
    echo "❌ 无法确定 flag 状态，请检查设备连接"
fi
```

### 1.2 快速判断表

| 命令 | 预期结果 | 异常含义 |
|---|---|---|
| `service list \| grep device_config` | 显示服务 | 服务不存在 → FlagService 未启动 |
| `cmd device_config list \| grep flag` | 显示 flag 定义 | 无输出 → aconfig 文件缺失 |
| `cmd device_config get ns flag` | `true` 或 `false` | `null` → flag 未加载到运行时 |
| `ls /system/etc/aconfig_flags.pb` | 文件存在 | 不存在 → flag 文件未打包 |

---

## 二、检查 Flag 服务是否可用

### 2.1 查看 flag 相关服务

```bash
# 检查有哪些 flag 相关服务
adb shell service list | grep -iE "flag|config|aconfig"
```

**正常设备应该看到：**
```
device_config: []
feature_flags: [android.flags.IFeatureFlags]   ← Android 15+ 才有
```

### 2.2 确认使用哪个命令

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

## 三、读取 Flag 值

### 3.1 读取单个 flag

```bash
# 语法：cmd <service> get <namespace> <flag_name>

# 读取 MediaRouter2 相关的 flag
adb shell cmd device_config get media_solutions enable_screen_off_scanning
adb shell cmd device_config get media_better_together enable_full_scan_with_media_content_control
```

**返回值说明：**
- `true` → flag 已启用
- `false` → flag 已禁用
- `null` → flag 未定义或未加载到运行时
- 报错 → 服务异常

### 3.2 列出某个 namespace 下所有 flag

```bash
adb shell cmd device_config list media_solutions
adb shell cmd device_config list media_better_together
```

### 3.3 列出所有 namespace

```bash
adb shell cmd device_config list_namespaces
```

### 3.4 列出所有 flag（可能很多）

```bash
adb shell cmd device_config list
```

### 3.5 搜索特定 flag

```bash
adb shell cmd device_config list | grep -i "screen_off"
adb shell cmd device_config list | grep -i "media_content"
```

---

## 四、诊断 list 与 get 不一致问题

### 4.1 问题现象

```bash
# list 能看到 flag，且值为 true
$ cmd device_config list | grep enable_screen_off_scanning
media_solutions/com.android.media.flags.enable_screen_off_scanning=true

# get 却返回 null
$ cmd device_config get media_solutions enable_screen_off_scanning
null
```

### 4.2 原因分析

| 命令 | 数据来源 | 说明 |
|---|---|---|
| `list` | aconfig proto 文件 + DeviceConfig property | 能看到所有定义的 flag |
| `get` | 只从 DeviceConfig property 存储中读取 | 只能读到运行时已加载的值 |

**根本原因：** Flag 定义存在（在 proto 文件中），但没有被加载到运行时存储中。

### 4.3 诊断步骤

```bash
# 1. 检查 list 命令
adb shell cmd device_config list | grep enable_screen_off_scanning

# 2. 检查 get 命令
adb shell cmd device_config get media_solutions enable_screen_off_scanning

# 3. 如果 list 有值但 get 返回 null，说明 flag 未加载到运行时
```

### 4.4 解决方案

```bash
# 临时方案：手动设置 flag 值
adb shell cmd device_config put media_solutions enable_screen_off_scanning true
adb shell cmd device_config put media_better_together enable_full_scan_with_media_content_control true

# 验证
adb shell cmd device_config get media_solutions enable_screen_off_scanning
# 应该返回 true 而不是 null
```

---

## 五、直接读取 Flag 文件

### 5.1 检查 aconfig proto 文件

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

### 5.2 查看文件内容（粗略）

```bash
# 用 strings 命令查看 flag 名称
adb shell strings /system/etc/aconfig_flags.pb | grep -i "screen_off\|media_content"

# 用 hexdump 查看（需要 root）
adb shell hexdump -C /system/etc/aconfig_flags.pb | head -50
```

---

## 六、通过 dumpsys 检查

### 6.1 dump 服务状态

```bash
# dump device_config 服务
adb shell dumpsys device_config

# dump feature_flags 服务（如果有）
adb shell dumpsys feature_flags
```

### 6.2 搜索特定 flag

```bash
# 搜索 MediaRouter2 相关 flag
adb shell dumpsys device_config | grep -i "screen_off_scanning"
adb shell dumpsys device_config | grep -i "full_scan_with_media"

# 查看所有 namespace
adb shell dumpsys device_config | grep "namespace"
```

---

## 七、检查 Flag 对应的功能是否生效

### 7.1 检查 MediaRouter2 功能

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

### 7.2 检查其他常见 flag 功能

```bash
# 检查 WindowManager 相关 flag
adb shell cmd device_config list window_manager_native_boot

# 检查 SystemUI 相关 flag
adb shell cmd device_config list systemui

# 检查 ActivityManager 相关 flag
adb shell cmd device_config list activity_manager_native_boot
```

---

## 八、完整诊断脚本

### 8.1 基础检查脚本

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
echo "--- list 命令 ---"
adb shell cmd $CMD list 2>&1 | grep -E "enable_screen_off_scanning|enable_full_scan_with_media_content_control"

echo ""
echo "--- get 命令 ---"
echo "enable_screen_off_scanning: $(adb shell cmd $CMD get media_solutions enable_screen_off_scanning 2>&1)"
echo "enable_full_scan_with_media_content_control: $(adb shell cmd $CMD get media_better_together enable_full_scan_with_media_content_control 2>&1)"

echo ""
echo "===== 4. 检查 aconfig 文件 ====="
adb shell ls -la /system/etc/aconfig_flags.pb 2>&1
adb shell ls -la /vendor/etc/aconfig_flags.pb 2>&1

echo ""
echo "===== 5. 诊断结论 ====="
LIST_COUNT=$(adb shell cmd $CMD list 2>&1 | grep -c "enable_screen_off_scanning")
GET_RESULT=$(adb shell cmd $CMD get media_solutions enable_screen_off_scanning 2>&1)

if [ "$LIST_COUNT" -gt 0 ] && [ "$GET_RESULT" = "null" ]; then
    echo "⚠️  Flag 定义存在（list 可见），但运行时未加载（get 返回 null）"
    echo ""
    echo "建议执行："
    echo "  adb shell cmd $CMD put media_solutions enable_screen_off_scanning true"
    echo "  adb shell cmd $CMD put media_better_together enable_full_scan_with_media_content_control true"
elif [ "$GET_RESULT" = "true" ] || [ "$GET_RESULT" = "false" ]; then
    echo "✅ Flag 运行时状态正常"
else
    echo "❌ 无法确定 flag 状态"
fi
```

**使用方法：**
```bash
chmod +x flag_check.sh
./flag_check.sh
```

### 8.2 完整诊断脚本

```bash
#!/bin/bash
# flag_diagnose.sh - 完整的 flag 诊断脚本

set -e

echo "╔════════════════════════════════════════════════════════════╗"
echo "║         ADB Flag 能力诊断工具                              ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""

# 检查设备连接
echo "【1/8】检查设备连接..."
if ! adb devices | grep -q "device$"; then
    echo "❌ 设备未连接"
    exit 1
fi
echo "✅ 设备已连接"
echo ""

# 检查服务
echo "【2/8】检查 flag 相关服务..."
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
echo "【3/8】检查 aconfig 文件..."
for file in /system/etc/aconfig_flags.pb /vendor/etc/aconfig_flags.pb; do
    if adb shell ls -la "$file" 2>/dev/null | grep -q "$file"; then
        echo "✅ $file 存在"
    else
        echo "⚠️  $file 不存在"
    fi
done
echo ""

# 检查 list 命令
echo "【4/8】检查 list 命令..."
LIST_RESULT=$(adb shell cmd $CMD list 2>&1 | grep "enable_screen_off_scanning")
if [ -n "$LIST_RESULT" ]; then
    echo "✅ list 命令正常: $LIST_RESULT"
else
    echo "⚠️  list 命令未找到相关 flag"
fi
echo ""

# 检查 get 命令
echo "【5/8】检查 get 命令..."
GET_RESULT=$(adb shell cmd $CMD get media_solutions enable_screen_off_scanning 2>&1)
echo "get 结果: $GET_RESULT"
echo ""

# 诊断
echo "【6/8】诊断..."
if [ -n "$LIST_RESULT" ] && [ "$GET_RESULT" = "null" ]; then
    echo "⚠️  Flag 定义存在（list 可见），但运行时未加载（get 返回 null）"
    echo ""
    echo "建议执行："
    echo "  adb shell cmd $CMD put media_solutions enable_screen_off_scanning true"
    echo "  adb shell cmd $CMD put media_better_together enable_full_scan_with_media_content_control true"
elif [ "$GET_RESULT" = "true" ] || [ "$GET_RESULT" = "false" ]; then
    echo "✅ Flag 运行时状态正常"
else
    echo "❌ 无法确定 flag 状态"
fi
echo ""

# 检查功能
echo "【7/8】检查 MediaRouter2 功能..."
if adb shell dumpsys media.router 2>/dev/null | grep -q "MediaRouterService"; then
    echo "✅ MediaRouterService 运行中"
else
    echo "⚠️  MediaRouterService 状态异常"
fi
echo ""

# 总结
echo "【8/8】诊断总结..."
echo "════════════════════════════════════════════════════════════"
echo "设备 flag 能力诊断完成。"
echo ""
echo "如果 list 有值但 get 返回 null，说明 flag 未加载到运行时。"
echo "这会导致 CTS 测试失败，因为 CTS 使用 get 命令读取 flag 值。"
echo "════════════════════════════════════════════════════════════"
```

**使用方法：**
```bash
chmod +x flag_diagnose.sh
./flag_diagnose.sh
```

---

## 九、常见问题排查

### 9.1 `cmd: Can't find service: flag`

**原因：** 设备上没有 `flag` 服务

**解决：**
```bash
# 使用旧版命令
adb shell cmd device_config get <namespace> <flag_name>
```

### 9.2 flag 返回 `null`

**原因：** flag 未定义或未加载到运行时

**排查：**
```bash
# 检查 list 命令是否能看到 flag
adb shell cmd device_config list | grep <flag_name>

# 如果 list 能看到但 get 返回 null，说明 flag 未加载到运行时
# 手动设置 flag 值
adb shell cmd device_config put <namespace> <flag_name> true
```

### 9.3 aconfig 文件不存在

**原因：** 系统编译时未包含 flag 文件

**排查：**
```bash
# 检查所有可能的路径
adb shell find / -name "aconfig_flags.pb" 2>/dev/null

# 检查 APEX 模块
adb shell ls /apex/*/etc/aconfig_flags/ 2>/dev/null
```

### 9.4 flag 值为 false，但测试期望为 true

**原因：** flag 被禁用

**解决：**
```bash
# 临时启用 flag（需要 root）
adb shell cmd device_config put <namespace> <flag_name> true

# 或者通过 override
adb shell cmd device_config override <namespace> <flag_name> true
```

---

## 十、针对 CTS 失败的快速诊断流程

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
    │   │   ├─ 执行：adb shell cmd device_config list | grep flag_name
    │   │   │   ├─ 有输出 → flag 定义存在
    │   │   │   └─ 无输出 → flag 定义缺失
    │   │   │
    │   │   └─ 执行：adb shell cmd device_config get namespace flag_name
    │   │       ├─ 返回 true/false → flag 正常
    │   │       ├─ 返回 null → flag 未加载到运行时
    │   │       │   └─ 解决：手动 put flag 值
    │   │       └─ 报错 → 服务异常
    │   │
    │   └─ 其他异常 → 参考其他诊断文档
    │
    └─ 执行完整诊断脚本
        └─ ./flag_diagnose.sh
```

---

## 十一、参考资源

- [AOSP aconfig 文档](https://source.android.com/docs/core/architecture/modular-system/aconfig)
- [DeviceConfig API](https://developer.android.com/reference/android/provider/DeviceConfig)
- [CTS 测试框架](https://source.android.com/docs/compatibility/cts)

---

*文档创建时间：2026-06-16*  
*最后更新：2026-06-16*  
*适用于 AOSP Android 14/15*
