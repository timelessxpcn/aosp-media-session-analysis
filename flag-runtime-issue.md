# Flag 读取异常分析：list 能看到但 get 返回 null

## 问题现象

在 W 版本设备上执行以下命令：

```bash
# list 能看到 flag，且值为 true
$ cmd device_config list | grep enable_screen_off_scanning
media_solutions/com.android.media.flags.enable_screen_off_scanning=true

$ cmd device_config list | grep enable_full_scan_with_media_content_control
media_better_together/com.android.media.flags.enable_full_scan_with_media_content_control=true

# get 却返回 null
$ cmd device_config get media_solutions enable_screen_off_scanning
null

$ cmd device_config get media_better_together enable_full_scan_with_media_content_control
null
```

---

## 源码分析

### DeviceConfigService 实现

从 AOSP 源码 `DeviceConfigService.java` 可以看到：

**`list` 命令：**
```java
case LIST:
    if (namespace != null) {
        DeviceConfig.Properties properties = DeviceConfig.getProperties(namespace);
        List<String> keys = new ArrayList<>(properties.getKeyset());
        Collections.sort(keys);
        for (String name : keys) {
            pout.println(name + "=" + properties.getString(name, null));
        }
    } else {
        for (String line : listAll(iprovider)) {
            // 直接读取所有 flag，包括从 aconfig proto 文件
        }
    }
```

**`get` 命令：**
```java
case GET:
    pout.println(DeviceConfig.getProperty(namespace, key));
```

### 数据来源区别

| 命令 | 数据来源 | 说明 |
|---|---|---|
| `list` | aconfig proto 文件 + DeviceConfig property | 能看到所有定义的 flag |
| `get` | 只从 DeviceConfig property 存储中读取 | 只能读到运行时已加载的值 |

---

## aconfig Flag 生命周期

```
┌─────────────────────────────────────────────────────────────┐
│                    aconfig Flag 生命周期                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  编译时：                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ media_better_together.aconfig                       │    │
│  │ flag {                                              │    │
│  │   name: "enable_full_scan_with_media_content_control"│   │
│  │   namespace: "media_better_together"                │    │
│  │ }                                                   │    │
│  └─────────────────────────────────────────────────────┘    │
│           ↓ 编译生成                                         │
│  /system/etc/aconfig_flags.pb  ← list 能读到这个             │
│                                                              │
│  运行时：                                                    │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ system_server 启动                                   │    │
│  │   ↓                                                 │    │
│  │ FlagService 初始化                                   │    │
│  │   ↓                                                 │    │
│  │ 读取 aconfig_flags.pb                                │    │
│  │   ↓                                                 │    │
│  │ 将 flag 值写入 DeviceConfig property 存储            │    │
│  │   ↓                                                 │    │
│  │ SettingsProvider 持久化                              │    │
│  └─────────────────────────────────────────────────────┘    │
│           ↓                                                 │
│  DeviceConfig.getProperty() 能读到  ← get 依赖这个          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 问题诊断

### 设备实际情况

| 检查项 | 结果 | 说明 |
|---|---|---|
| aconfig proto 文件 | ✅ 存在 | `list` 能读到 |
| Flag 定义 | ✅ 正确 | namespace 和 name 都正确 |
| Flag 值 | ✅ true | `list` 显示值为 true |
| 运行时加载 | ❌ 失败 | `get` 返回 null |

### 根本原因

**Flag 值没有被正确加载到 DeviceConfig property 存储中。**

可能的原因：
1. FlagService 初始化失败
2. aconfig proto 文件解析失败
3. flag 值写入 DeviceConfig 时出错
4. SettingsProvider 存储异常

---

## 验证步骤

### 1. 检查 FlagService 状态

```bash
# 检查 FlagService 是否运行
adb shell dumpsys activity services | grep -i flag
adb shell ps -A | grep -i flag

# 检查 DeviceConfig 服务
adb shell dumpsys device_config | head -50
```

### 2. 检查 aconfig 文件

```bash
# 检查文件是否存在
adb shell ls -la /system/etc/aconfig_flags.pb
adb shell ls -la /vendor/etc/aconfig_flags.pb

# 检查文件内容
adb shell strings /system/etc/aconfig_flags.pb | grep -i "screen_off\|media_content"
```

### 3. 检查系统日志

```bash
# 查看 flag 相关日志
adb logcat | grep -i "aconfig\|flag" | head -100

# 查看 system_server 启动日志
adb logcat -s SystemServer | grep -i flag
```

### 4. 手动触发 flag 加载

```bash
# 手动设置 flag 值（需要 root）
adb shell cmd device_config put media_solutions enable_screen_off_scanning true
adb shell cmd device_config put media_better_together enable_full_scan_with_media_content_control true

# 验证
adb shell cmd device_config get media_solutions enable_screen_off_scanning
# 应该返回 true 而不是 null
```

---

## 解决方案

### 临时方案：手动设置 flag 值

```bash
# 手动将 flag 值写入 DeviceConfig
adb shell cmd device_config put media_solutions enable_screen_off_scanning true
adb shell cmd device_config put media_better_together enable_full_scan_with_media_content_control true

# 验证
adb shell cmd device_config get media_solutions enable_screen_off_scanning
adb shell cmd device_config get media_better_together enable_full_scan_with_media_content_control
```

**注意：** 这个方案在设备重启后会失效，需要重新设置。

### 永久方案：修复 FlagService

需要检查：
1. FlagService 是否在 system_server 中正确注册
2. aconfig proto 文件是否被正确解析
3. flag 值是否被正确写入 DeviceConfig property 存储
4. SettingsProvider 是否有权限问题

### 排查清单

- [ ] 检查 FlagService 是否在 system_server 中启动
- [ ] 检查 aconfig proto 文件格式是否正确
- [ ] 检查 system_server 日志中是否有 flag 相关错误
- [ ] 检查 SettingsProvider 是否有写入权限
- [ ] 检查是否有 SELinux 策略阻止 flag 加载

---

## 与 CTS 失败的关系

### 之前的分析

之前认为 CTS 失败是因为 `cmd flag` 服务不存在，应该用 `cmd device_config`。

### 修正后的分析

现在发现：
1. `cmd device_config` 服务存在
2. `list` 命令能看到 flag
3. 但 `get` 命令返回 null

**CTS 工具链可能使用 `get` 命令读取 flag 值：**
- `get` 返回 null → CTS 认为 flag 不存在或值为 false
- 测试被标记为失败

### 结论

**CTS 失败的根本原因是 flag 值没有被正确加载到运行时存储中，而不是服务名称不匹配。**

---

## 快速验证脚本

```bash
#!/bin/bash
# flag_runtime_check.sh - 检查 flag 运行时状态

echo "===== 检查 flag 运行时状态 ====="
echo ""

echo "【1】检查 list 命令..."
LIST_RESULT=$(adb shell cmd device_config list 2>&1 | grep "enable_screen_off_scanning")
echo "list 结果: $LIST_RESULT"

echo ""
echo "【2】检查 get 命令..."
GET_RESULT=$(adb shell cmd device_config get media_solutions enable_screen_off_scanning 2>&1)
echo "get 结果: $GET_RESULT"

echo ""
echo "【3】诊断..."
if [ -n "$LIST_RESULT" ] && [ "$GET_RESULT" = "null" ]; then
    echo "⚠️  Flag 定义存在，但运行时未加载"
    echo ""
    echo "建议执行："
    echo "  adb shell cmd device_config put media_solutions enable_screen_off_scanning true"
elif [ "$GET_RESULT" = "true" ] || [ "$GET_RESULT" = "false" ]; then
    echo "✅ Flag 运行时状态正常"
else
    echo "❌ 无法确定 flag 状态"
fi
```

---

## 参考

- [AOSP DeviceConfigService 源码](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/packages/SettingsProvider/src/com/android/providers/settings/DeviceConfigService.java)
- [aconfig 文档](https://source.android.com/docs/core/architecture/modular-system/aconfig)

---

*文档创建时间：2026-06-16*  
*基于实际设备调试结果*
