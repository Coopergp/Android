# Android 9 → 10 → 11 系统级演进路线详解

> 覆盖范围：AOSP架构变更、安全模型演进、进程模型、权限体系、Binder通信、虚拟机、编译体系、存储隔离、网络栈、媒体框架等核心子系统

---

## 目录

1. [Android 9 (Pie) 系统核心演进](#1-android-9-pie-系统核心演进)
2. [Android 10 (Q) 系统核心演进](#2-android-10-q-系统核心演进)
3. [Android 11 (R) 系统核心演进](#3-android-11-r-系统核心演进)
4. [三代系统横向对比总结](#4-三代系统横向对比总结)
5. [系统开发者关键实践要点](#5-系统开发者关键实践要点)

---

## 1. Android 9 (Pie) 系统核心演进

### 1.1 安全体系

#### 1.1.1 默认启用 HTTPS（Network Security Config 强制化）
- 应用默认不再允许明文 HTTP 流量（targetSdkVersion≥28）
- 系统强制走 `NetworkSecurityConfig`，开发者需在 `res/xml/network_security_config.xml` 中显式声明允许 cleartext 的域名
- 系统层面：`NetworkSecurityConfigProvider` 在 `SSLCSocketFactory` 初始化链路中插入校验逻辑

```xml
<!-- 旧行为：所有HTTP均可通过 -->
<!-- Android 9 需显式声明 -->
<network-security-config>
    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">example.com</domain>
    </domain-config>
</network-security-config>
```

#### 1.1.2 BiometricPrompt 统一生物认证 API
- 废弃 `FingerprintManager`，引入 `BiometricPrompt`
- 系统层：`BiometricService` 作为新服务在 `SystemServer` 中注册
- 底层抽象出 `IBiometricServiceReceiver` + `BiometricManager`，支持指纹、虹膜、人脸多模态统一调用
- 认证结果通过 `AuthenticationCallback` 回调，不再需要 app 直接访问硬件 HAL

#### 1.1.3 SELinux 进一步收紧
- 引入 `treble_sysprop_neverallow`：禁止 system 分区进程直接写 vendor 的 sysprop
- 新增 `ioctl` 过滤：使用 ioctl allowlisting（`ioctl` neverallow 规则），限制特定进程可执行的 ioctl 命令号
- `proc` 文件系统访问进一步限制，`/proc/net/` 节点对非特权进程关闭

#### 1.1.4 Conscrypt 成为默认 TLS Provider
- `Conscrypt`（基于 BoringSSL）完全替代 Android 上的 `OpenSSL Provider`
- 支持 TLS 1.3（默认启用）
- 系统级提供 `SSLContext.getInstance("TLS")` 均走 Conscrypt

#### 1.1.5 硬件安全模块（StrongBox）
- 引入 `StrongBox Keymaster HAL`：要求密钥在独立安全芯片（SE）中生成和存储
- API：`KeyGenParameterSpec.Builder.setIsStrongBoxBacked(true)`
- `KeyStore` 服务扩展 `StrongBoxUnavailableException` 处理路径

---

### 1.2 Binder & IPC

#### 1.2.1 Binder 驱动改进
- Android 9 对应内核版本普遍为 4.9/4.14，Binder 驱动加入 `BINDER_TYPE_FD_ARRAY` 支持批量文件描述符传递
- 引入 `oneway` 调用的拥塞控制：`BC_TRANSACTION_SG`（scatter-gather）优化大数据块拷贝，减少内存拷贝次数

#### 1.2.2 Vendor APEX 前身：HIDL 接口冻结
- 所有 Vendor HAL 接口必须通过 `HIDL`（HAL Interface Definition Language）定义
- 系统服务与 HAL 通过 `hwbinder`（`/dev/hwbinder`）通信，与 `libbinder`（`/dev/binder`）隔离
- `vndbinder`（`/dev/vndbinder`）用于 vendor 进程间通信

**Binder 域隔离架构：**

```
App/System Process          Vendor Process
     │                           │
  /dev/binder               /dev/vndbinder
     │                           │
  libbinder                  libbinder
     │                           │
     └─────── hwbinder ──────────┘
           /dev/hwbinder
           (HAL 调用)
```

---

### 1.3 ART 虚拟机

#### 1.3.1 .art 文件格式与 CompilerFilter
- `CompilerFilter` 新增 `quicken` 档位，介于 `verify` 和 `speed` 之间
- `dex2oat` 在安装时默认使用 `speed-profile` 编译（基于 Cloud Profile 或 UI 流畅度采集的 profile）

#### 1.3.2 Hidden API 黑名单机制（非 SDK 接口限制）
- 正式引入 `HiddenApiPolicy`，分为 `black`、`dark-grey`、`light-grey`、`white` 四档
- 系统通过 `art/runtime/hidden_api.cc` 中的访问控制检查，在反射、JNI 等调用路径中插入拦截
- 对应枚举：`hiddenapi::Action::kAllow / kAllowButWarn / kDeny`

```
调用路径：
Java Reflection → Class::GetDeclaredField
                → hiddenapi::ShouldDenyAccessToMember(member, caller, access_method)
                → 根据 ApiList 决定 allow/warn/deny
```

#### 1.3.3 Profile Guided Optimization (PGO) 成熟化
- `dexlayout` 工具在 dexopt 阶段对热点代码进行重排，提升 I/O 局部性
- `ArtManager` API 暴露给系统，支持 background dex compilation

---

### 1.4 进程模型与内存管理

#### 1.4.1 App Standby Buckets（应用待机分组）
- 引入 `UsageStatsManager.getAppStandbyBucket()` API
- 系统将 App 按活跃度分为 5 组：`ACTIVE / WORKING_SET / FREQUENT / RARE / RESTRICTED`
- 影响：Job 调度频率、网络访问、报警精度
- 系统实现：`AppStandbyController` 在 `UsageStatsService` 内，监听屏幕交互、通知、前台服务等事件更新分组

#### 1.4.2 前台服务必须显示通知（强制化）
- targetSdkVersion≥28 的应用：启动前台服务后必须在 5 秒内调用 `startForeground()`，否则系统抛出 `ForegroundServiceDidNotStartInTimeException`
- 系统侧：`ActiveServices.setServiceForegroundLocked()` 中加入超时 Handler

#### 1.4.3 后台 Activity 启动限制（前奏）
- Android 9 开始限制：targetSdkVersion≥28 不允许后台服务直接 `startActivity()`（需 FLAG_ACTIVITY_NEW_TASK + 用户交互）
- 系统层：`ActivityStarter.canRequestFullScreenIntent()` 校验

---

### 1.5 Wi-Fi & 网络

#### 1.5.1 Wi-Fi RTT（IEEE 802.11mc）
- 新增 `WifiRttManager`，支持室内定位测距
- 系统服务：`RttService` → `WifiRttController` → HAL 层 `IWifiRttController`

#### 1.5.2 DNS over TLS（Private DNS）
- 系统默认支持 DoT（DNS-over-TLS）
- 设置路径：`Settings.Global.PRIVATE_DNS_MODE`（`off / opportunistic / hostname`）
- 系统实现：`DnsResolver` 守护进程（`netd` 子模块）处理 TLS 握手

---

### 1.6 显示 & 刘海屏支持

#### 1.6.1 DisplayCutout API
- 正式引入 `DisplayCutout` 类，暴露刘海区域 `SafeInsets`
- `WindowInsets.getDisplayCutout()` 让应用感知刘海形状
- `WindowManager.LayoutParams.layoutInDisplayCutoutMode` 控制窗口与刘海区域的关系：
  - `LAYOUT_IN_DISPLAY_CUTOUT_MODE_DEFAULT`
  - `LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES`
  - `LAYOUT_IN_DISPLAY_CUTOUT_MODE_NEVER`

#### 1.6.2 多相机 API（Multi-Camera）
- `CameraCharacteristics.LOGICAL_MULTI_CAMERA_PHYSICAL_IDS`
- 系统层：`CameraService` 引入逻辑摄像头抽象，底层可映射到多个物理摄像头

---

## 2. Android 10 (Q) 系统核心演进

### 2.1 项目 Mainline（APEX 模块化）

#### 2.1.1 架构背景
Android 10 引入 **Project Mainline**，将部分系统组件从 AOSP 分离为可独立更新的 APEX 模块（Android Package Extensions），通过 Play Store 推送更新，无需完整 OTA。

#### 2.1.2 APEX 格式
- APEX 文件本质是 ZIP 包，内含：
  - `apex_payload.img`：ext4/erofs 格式的只读镜像
  - `AndroidManifest.xml`：描述模块元信息
  - `apex_pubkey`：签名公钥
- 挂载机制：`apexd` 守护进程在启动时用 `dm-verity` 将 `apex_payload.img` 挂载到 `/apex/<module-name>/`

```
启动流程：
init → apexd（/system/bin/apexd）
     → 扫描 /system/apex/ 和 /data/apex/active/
     → 校验签名（APK Signature Scheme v3）
     → loop device + dm-verity 挂载
     → 更新 /apex/com.android.xxx → 最新版本软链接
```

#### 2.1.3 首批 Mainline 模块（Android 10）

| 模块名 | 包含内容 |
|--------|----------|
| `com.android.conscrypt` | TLS/SSL 库 |
| `com.android.media` | Media Framework 核心 |
| `com.android.media.swcodec` | 软件编解码器（MediaCodec） |
| `com.android.resolv` | DNS Resolver（原 `netd` 子集） |
| `com.android.runtime` | ART 虚拟机 + core libraries |
| `com.android.tzdata` | 时区数据库 |
| `com.android.i18n` | ICU 国际化库 |

#### 2.1.4 对系统开发者的影响
- 以上模块的代码路径在 `/apex/com.android.xxx/` 下，而非 `/system/lib/`
- 系统服务加载库时需通过 `ClassLoaderFactory` 的 APEX-aware 路径
- 开发者修改 ART、Conscrypt 时，需重新打包 APEX，而非仅替换 `.so`

---

### 2.2 存储隔离（Scoped Storage）

#### 2.2.1 设计动机
- 解决应用滥用外部存储，扫描用户私人文件
- 废弃 `READ_EXTERNAL_STORAGE` / `WRITE_EXTERNAL_STORAGE` 的"万能"语义

#### 2.2.2 隔离沙箱机制
- 每个应用获得专属目录：`/sdcard/Android/data/<package>/`，无需权限可读写
- 访问其他应用文件或媒体库：必须通过 `MediaStore` API 或 `Storage Access Framework (SAF)`
- `MediaStore` 新增 `IS_PENDING` 列，支持原子写入

#### 2.2.3 系统实现：FUSE + MediaProvider
- `MediaProvider` 作为 Mainline 模块（`com.android.providers.media`）单独维护
- 底层使用 FUSE（用户态文件系统）对 `/sdcard` 做访问控制代理
- 内核态 FUSE 驱动将文件 I/O 请求路由到 `MediaProvider` 进程，由其决策是否放行

```
App → open("/sdcard/DCIM/photo.jpg")
    → VFS → FUSE → /dev/fuse → MediaProvider 守护进程
    → 检查调用方 UID、MediaStore 归属、权限
    → 允许 / 拒绝 / 重定向
```

#### 2.2.4 受影响的旧 API

| 旧方式 | Android 10 替代方案 |
|--------|---------------------|
| `File("/sdcard/...")` | `MediaStore.Images.Media.EXTERNAL_CONTENT_URI` |
| `Environment.getExternalStorageDirectory()` | `Context.getExternalFilesDir()` |
| `WRITE_EXTERNAL_STORAGE` | 应用私有目录无需权限；媒体文件用 MediaStore |

---

### 2.3 权限体系重大变更

#### 2.3.1 后台位置权限独立化
- 新增 `ACCESS_BACKGROUND_LOCATION` 权限（危险权限，需运行时申请）
- 应用必须先持有前台位置权限（`ACCESS_FINE_LOCATION` 或 `ACCESS_COARSE_LOCATION`），才能申请后台位置权限
- targetSdkVersion≥29：不能在单次 `requestPermissions()` 中同时请求前台+后台位置权限
- 系统实现：`LocationManagerService` → `LocationPermissionChecker`，区分调用时 `AppOpsManager.OP_FINE_LOCATION` 的 foreground/background 状态

#### 2.3.2 后台 Activity 启动彻底收紧
- targetSdkVersion≥29：后台应用几乎不能直接启动 Activity（除非满足豁免白名单条件）
- 豁免条件：持有 `SYSTEM_ALERT_WINDOW`、收到 `PendingIntent`（由前台应用/系统发出）、Notification 关联的 `fullScreenIntent`
- 系统层：`ActivityTaskManagerService.startActivity()` → `BackgroundActivityStartController.checkBackgroundActivityStart()`

#### 2.3.3 设备标识符权限收紧
- IMEI、序列号等硬件标识符现在需要 `READ_PRIVILEGED_PHONE_STATE`（仅系统 App 可用）
- 普通 App 通过 `Build.getSerial()` 会收到 `SecurityException`
- 系统内部使用：`Phone.getImei()` 内部校验 `READ_PRIVILEGED_PHONE_STATE`

#### 2.3.4 剪贴板访问限制
- 后台应用不再能读取剪贴板内容（`ClipboardManager.getPrimaryClip()` 返回 null）
- 系统实现：`ClipboardService.getPrimaryClip()` 检查调用者 uid 是否处于前台

---

### 2.4 进程模型演进

#### 2.4.1 后台进程 ANR 检测增强
- 新增 `Context.ACTIVITY_SERVICE` → `ActivityManager.getHistoricalProcessExitReasons()`（Android 11 正式，此处为基础奠定）
- `ProcessErrorStateInfo` 携带更丰富的 ANR 触发堆栈

#### 2.4.2 AppOps 精细化
- `AppOpsManager` 新增 `startWatchingNoted()` API，支持监听某 Op 是否被记录（相对于以前只能监听"是否被使用"）
- 系统服务通过 `AppOpsService.noteOpNoThrow()` 打点，配合审计日志

---

### 2.5 网络栈模块化（Network Stack APEX）

#### 2.5.1 NetworkStack 独立为 APEX 模块
- `com.android.networkstack` 作为 Mainline 模块，包含：
  - DHCP 客户端
  - Captive Portal 检测
  - 网络评估（NetworkMonitor）
  - IP 地址冲突检测（ARP probe）

#### 2.5.2 ConnectivityService 重构
- `ConnectivityService` 不再直接处理具体协议逻辑，转为通过 Binder 调用 NetworkStack 进程
- `NetworkMonitor` 运行在独立进程（`com.android.networkstack.process`）

#### 2.5.3 TUN/TAP VPN 改进
- `VpnService.Builder.setHttpProxy()` 正式支持
- `ConnectivityManager.getDefaultProxy()` 链路改善

---

### 2.6 Wi-Fi 架构重构

#### 2.6.1 WifiService 重大重构
- `WifiService` 从 `SystemServer` 中独立出来，移到 `com.android.wifi` APEX
- `WifiManager` → `WifiServiceImpl` → `WifiStateMachine`（改名为 `ClientModeImpl`）
- 状态机（StateMachine）全面 Kotlin 化（部分组件）

#### 2.6.2 Wi-Fi Network Suggestion
- 新增 `WifiNetworkSuggestion` API，允许应用"建议"网络配置（无需用户手动连接）
- 与旧 `WifiConfiguration` 共存，优先级规则：用户手动配置 > 系统建议 > 应用建议

#### 2.6.3 Wi-Fi 直连（Wi-Fi P2P）改进
- `WifiP2pManager.discoverPeers()` 调用需要 `ACCESS_FINE_LOCATION`（Android 10 强制）

---

### 2.7 Thermal API

#### 2.7.1 系统热管理 API 开放
- `PowerManager.getThermalHeadroom()` → 返回 0.0~1.0 的归一化热余量
- `PowerManager.OnThermalStatusChangedListener`：监听热状态变化
- 热状态枚举：`THERMAL_STATUS_NONE / LIGHT / MODERATE / SEVERE / CRITICAL / EMERGENCY / SHUTDOWN`
- 系统实现：`ThermalManagerService` → `ThermalHal` → `IThermal.hal`

---

### 2.8 折叠屏 & 多窗口

#### 2.8.1 折叠屏支持
- `FoldingFeature`（Jetpack Window Manager 前身基础）
- `ActivityInfo.CONFIG_SCREEN_SIZE` 在折叠状态变化时触发配置变更
- 系统层：`WindowManagerService` 感知 `DisplayAreaGroup` 变更，触发 Activity 重建或配置更新

#### 2.8.2 屏幕尺寸配置去敏感
- `ActivityInfo.configChanges` 不再推荐使用 `screenSize`，改用 `smallestScreenSize`

---

### 2.9 ART & 编译链路

#### 2.9.1 ART 作为 Mainline 模块
- `com.android.runtime` APEX 包含 ART 运行时、`core-oj.jar`、`core-libart.jar`
- 可通过 Play Store OTA 更新 ART，无需系统 OTA

#### 2.9.2 dex2oat 后台策略
- 新增 `BackgroundDexOptService.ScheduleIdleOptimization()`
- 利用 JobScheduler 在充电+空闲时段异步完成 dexopt
- `CompilerFilter` 根据 CloudProfile（Google Play 提供）自动选择热点路径编译

#### 2.9.3 Compact Dex（CDex）
- `--compact-dex-level` 选项正式在生产中启用
- CDex 对比标准 DEX 减少约 10~30% 体积，优化常量池去重

---

### 2.10 SELinux 演进

#### 2.10.1 System_ext & Product 分区引入
- 除了 `system`、`vendor`、`odm` 分区，新增 `system_ext`（OEM 扩展系统组件）和 `product`（运营商/产品定制）分区
- SELinux 为这两个分区引入对应类型：`system_ext_file`、`product_file`
- 系统服务加载路径需区分这些分区

#### 2.10.2 Treble 合规性增强
- `vintf`（Vendor Interface Manifest）校验在启动时强制执行
- `compatibility_matrix.x.xml` 必须与设备声明的 HAL 版本匹配，否则启动拒绝

---

## 3. Android 11 (R) 系统核心演进

### 3.1 权限体系深化

#### 3.1.1 一次性权限（One-Time Permission）
- 用户可授予"仅此一次"的运行时权限，适用于：`CAMERA`、`RECORD_AUDIO`、`ACCESS_FINE_LOCATION`、`ACCESS_COARSE_LOCATION`、`ACCESS_BACKGROUND_LOCATION`
- 系统实现：权限授予时标记 `FLAG_PERMISSION_ONE_TIME`，应用回到后台超过阈值时自动撤销
- 撤销逻辑：`PermissionController` 中的 `OneTimePermissionRevoker`，由 `JobScheduler` 定期驱动

```
用户点击"仅此一次" →
    PackageManager 标记 FLAG_PERMISSION_ONE_TIME →
    App 进入后台 →
    OneTimePermissionRevoker 检测 (延迟约1分钟) →
    自动撤销权限 + AppOp reset
```

#### 3.1.2 权限自动重置
- 长时间不使用的应用权限自动重置（几个月无交互）
- 系统实现：`PermissionController` APEX 模块中的 `HibernationController`（与休眠联动）
- 对应设置：`Settings > Privacy > Permission manager > Auto-reset unused apps`

#### 3.1.3 Bluetooth 权限分离
- 新增 `BLUETOOTH_SCAN`、`BLUETOOTH_CONNECT`、`BLUETOOTH_ADVERTISE` 三个新危险权限（Android 12 正式落地，11 为前期设计）
- Android 11 中 `BLUETOOTH` 和 `BLUETOOTH_ADMIN` 仍沿用，但开始规划分离路线

#### 3.1.4 前台服务类型强制声明
- targetSdkVersion≥30：`<service android:foregroundServiceType="...">` 必须声明服务类型
- 可用类型：`camera`、`connectedDevice`、`dataSync`、`location`、`mediaPlayback`、`mediaProjection`、`microphone`、`phoneCall`
- 系统侧：`ActiveServices.setServiceForegroundLocked()` 校验 `foregroundServiceType` 是否匹配

---

### 3.2 Scoped Storage 强制化与 MANAGE_EXTERNAL_STORAGE

#### 3.2.1 全面强制 Scoped Storage
- targetSdkVersion≥30：`requestLegacyExternalStorage` 清单标记失效，必须迁移到 Scoped Storage 模型
- FUSE 层始终对这类应用生效

#### 3.2.2 MANAGE_EXTERNAL_STORAGE 特殊权限
- 特殊应用（如文件管理器、防病毒）可通过 `ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION` 申请全盘访问
- 需在 `<uses-permission>` 声明 `MANAGE_EXTERNAL_STORAGE`，并引导用户到系统设置页手动开启
- Google Play 需要提供申请理由（审核）
- 系统实现：`MediaProvider` 中对 UID 持有的 `AppOpsManager.OP_MANAGE_EXTERNAL_STORAGE` 进行检查

---

### 3.3 进程模型：Bubble 与应用休眠

#### 3.3.1 应用休眠（App Hibernation）
- 扩展自 Android 9 的 App Standby，增加 `HIBERNATION` 状态（最深度休眠）
- 进入条件：数月无交互
- 休眠效果：
  - 强制停止应用（Kill 进程）
  - 清除运行时缓存（缓存 DEX 文件、优化代码删除）
  - 自动重置权限
- 系统实现：`UsageStatsService` → `AppStandbyController.setAppStandbyBucket(BUCKET_RESTRICTED)` → `ActivityManager.forceStopPackage()`

#### 3.3.2 Bubbles API 正式化
- `BubbleMetadata` + `Notification.Builder.setBubbleMetadata()` 正式 API（Android 10 为实验性）
- 气泡本质是一个悬浮的 Activity 窗口（`ActivityTaskManager` 管理）
- `WindowManager` 引入 `WINDOWING_MODE_PINNED` 的变体用于 Bubble 窗口

---

### 3.4 无线通信与网络

#### 3.4.1 5G API 开放
- `TelephonyManager.isDataCapable()` + `TelephonyManager.listen(LISTEN_DISPLAY_INFO_CHANGED)`
- `ServiceState.getNrState()` 返回 NR（5G）连接状态
- 新增 `TelephonyDisplayInfo`，区分 `OVERRIDE_NETWORK_TYPE_NR_NSA`、`NR_NSA_MMWAVE` 等精细网络类型

#### 3.4.2 带宽估算增强
- `ConnectivityManager.registerNetworkCallback()` 回调中 `LinkProperties.getLinkDownstreamBandwidthKbps()` 精度提升
- 系统实现：`NetworkStatsService` 统计更细粒度的吞吐量数据

#### 3.4.3 Wi-Fi 改进：建议 API 增强
- `WifiNetworkSuggestion` 增加 `setCredentialSharedWithUser(true)`，允许建议网络被用户手动管理
- `WifiManager.ACTION_WIFI_NETWORK_SUGGESTION_POST_CONNECTION`：建议连接成功后广播通知

---

### 3.5 媒体框架

#### 3.5.1 MediaCodec 增强
- 新增 `MediaCodec.setOnFirstTunnelFrameReadyListener()`
- Tunnel 模式（隧道解码）正式支持音视频同步播放（AudioTrack + MediaCodec 直连）
- 系统层：`MediaCodecList.getCodecInfos()` 包含 `CodecCapabilities.isFeatureSupported("tunnel-playback")`

#### 3.5.2 Image & Video Access 改进
- `MediaStore.createWriteRequest()` / `MediaStore.createDeleteRequest()` / `MediaStore.createTrashRequest()`：批量媒体操作 API，需用户确认
- 系统弹出系统对话框（不是应用自定义），防止静默删除用户媒体

```java
// 批量删除需要用户授权
PendingIntent pendingIntent = MediaStore.createDeleteRequest(
        getContentResolver(), uriList);
startIntentSenderForResult(pendingIntent.getIntentSender(), REQUEST_CODE, null, 0, 0, 0);
```

#### 3.5.3 媒体控制器重构
- `MediaSession2` / `MediaController2` 全面替代旧 `MediaSession` 体系
- 锁屏播放控制：`MediaStyle` Notification + `MediaSessionManager` 渲染系统级播放控制卡片

---

### 3.6 多屏幕与折叠屏

#### 3.6.1 Jetpack Window Manager 1.0
- `WindowManager.getCurrentWindowMetrics()` / `getMaximumWindowMetrics()` 正式 API
- 区分窗口可用区域 vs 最大可用区域，支持折叠屏展开/折叠状态

#### 3.6.2 Concurrent Camera Access（并发相机）
- `CameraManager.getConcurrentCameraIds()` 查询可同时使用的相机组合
- `CameraManager.isConcurrentSessionConfigurationSupported()` 校验并发流配置
- 系统层：`CameraService` 引入并发会话调度逻辑

---

### 3.7 安全体系

#### 3.7.1 Heap Pointer Authentication（PAC）
- 在支持 ARMv8.3 PA（Pointer Authentication）的设备上，ART 运行时对堆指针签名
- 防止 ROP（Return-Oriented Programming）攻击
- 系统编译链路：`-mbranch-protection=standard` 编译选项

#### 3.7.2 BiometricPrompt 增强
- 新增 `BiometricManager.canAuthenticate(int authenticators)` 区分：
  - `BIOMETRIC_STRONG`（Class 3：安全指纹/人脸）
  - `BIOMETRIC_WEAK`（Class 2）
  - `DEVICE_CREDENTIAL`（PIN/密码/图案）
- 系统层：`BiometricService` 维护每种认证器的强度分级

#### 3.7.3 Keystore 2.0 前瞻
- Android 11 奠定 `Keystore 2.0` 基础（Android 12 完整落地）
- 引入 `Domain.SELINUX` 命名空间，密钥访问由 SELinux 策略控制

---

### 3.8 ART & 运行时

#### 3.8.1 Garbage Collector 改进（Generational CC）
- Android 10 引入的 `Concurrent Copying (CC)` GC 在 Android 11 加入 `Generational` 模式
- 将堆划分为 young region 和 old region，young GC 仅扫描年轻代，降低 GC 停顿时间
- 启用方式：`ART_USE_READ_BARRIER=true`（默认启用）

#### 3.8.2 Non-SDK Interface 限制强化
- 黑名单范围扩大：将更多内部 API 从 `dark-grey` 升级为 `black`
- Android 11 新增 `UnsupportedOperationException` 以替代之前的软警告方式

---

### 3.9 窗口管理 & 输入

#### 3.9.1 Insets Controller API
- 正式引入 `WindowInsetsController`，替代 `View.setSystemUiVisibility()`（已废弃）
- 控制系统栏显示/隐藏：`windowInsetsController.hide(WindowInsets.Type.statusBars())`
- 系统层：`InsetsController` 与 `InsetsSourceProvider` 协作，通过 `SurfaceControl` 动画驱动

#### 3.9.2 Conversations API（通知对话）
- `MessagingStyle` + `ShortcutInfo` 构建对话型通知
- 系统通知中心将对话通知单独分区展示，优先级高于普通通知
- `BubbleMetadata` 可直接从对话通知触发气泡

---

### 3.10 性能与调试工具

#### 3.10.1 新增 HealthStats API
- `HealthStats`（通过 `BatteryStatsManager`）提供更细粒度的耗电统计
- `UidHealthStats`：按 UID 统计 CPU 时间、唤醒锁持有时长、网络流量

#### 3.10.2 NDK 新增 ImageDecoder JNI
- `AImageDecoder` NDK API 支持在 native 层直接解码 JPEG/PNG/WebP/GIF
- 底层复用 `libhwui` 中的解码逻辑

#### 3.10.3 exitReason API
- `ActivityManager.getHistoricalProcessExitReasons()` 正式可用
- 返回 `ApplicationExitInfo` 列表，字段包含：
  - `getReason()`：`REASON_ANR / REASON_LOW_MEMORY / REASON_CRASH / REASON_CRASH_NATIVE / REASON_SIGNALED`
  - `getTraceInputStream()`：ANR/Tombstone 的 trace 流
  - `getTimestamp()`、`getPid()`、`getProcessStateSummary()`

```java
ActivityManager am = getSystemService(ActivityManager.class);
List<ApplicationExitInfo> exitReasons = am.getHistoricalProcessExitReasons(
        null, // 当前包名
        0,    // pid，0表示所有
        10    // 最近10条
);
for (ApplicationExitInfo info : exitReasons) {
    Log.d(TAG, "Exit reason: " + info.getReason() + ", desc: " + info.getDescription());
}
```

---

## 4. 三代系统横向对比总结

### 4.1 安全体系演进对比

| 特性 | Android 9 | Android 10 | Android 11 |
|------|-----------|------------|------------|
| 生物认证 | BiometricPrompt 引入 | 稳定 | 多强度分级（STRONG/WEAK） |
| 位置权限 | 前/后台未分离 | ACCESS_BACKGROUND_LOCATION 独立 | 一次性权限支持 |
| 设备标识符 | 部分收紧 | IMEI 需特权权限 | 进一步收紧 MAC 随机化 |
| 存储 | 无沙箱 | Scoped Storage（可选退出） | Scoped Storage 强制 |
| 后台 Activity 启动 | 部分限制 | 严格限制 | 延续+细化豁免条件 |
| SELinux | ioctl 过滤 | system_ext/product 分区类型 | 进一步收紧 |
| TLS | TLS 1.3 默认 / Conscrypt | 稳定 | Keystore 2.0 前瞻 |
| 剪贴板 | 无限制 | 后台应用禁止读取 | 延续 |

### 4.2 进程模型演进对比

| 特性 | Android 9 | Android 10 | Android 11 |
|------|-----------|------------|------------|
| App Standby Buckets | 引入（5档） | 稳定 | 新增 HIBERNATION 档 |
| 前台服务 | 5秒超时强制 | 稳定 | 必须声明 serviceType |
| 后台 Activity 启动 | 部分限制 | 彻底收紧 | 稳定延续 |
| 权限自动重置 | 无 | 无 | 引入长期无使用自动重置 |
| 进程退出原因 | 无 API | 奠定基础 | getHistoricalProcessExitReasons |

### 4.3 系统架构模块化演进

| 特性 | Android 9 | Android 10 | Android 11 |
|------|-----------|------------|------------|
| 模块化 | HIDL 冻结 | APEX/Mainline 引入 | APEX 模块扩展，15+ 模块 |
| ART 更新方式 | 随 OTA | APEX OTA（Play Store） | 延续 |
| DNS Resolver | netd 内嵌 | 独立 APEX 模块 | 延续 |
| MediaProvider | 系统分区 | Mainline 模块 | 延续+FUSE 强制 |
| NetworkStack | systemserver 内 | 独立 APEX | 延续 |
| Wi-Fi | SystemServer 内 | 独立 APEX | 延续 |

### 4.4 ART 虚拟机演进

| 特性 | Android 9 | Android 10 | Android 11 |
|------|-----------|------------|------------|
| 非 SDK 接口限制 | 引入四档 | 扩大黑名单 | 更多升级为 black |
| GC | CC GC | CC GC 优化 | Generational CC |
| Compact DEX | 实验性 | 生产启用 | 延续 |
| ART 可独立更新 | 不支持 | APEX 支持 | 延续 |
| Profile 编译 | PGO 成熟 | Cloud Profile | 延续 |

---

## 5. 系统开发者关键实践要点

### 5.1 HAL 接口开发规范

#### 5.1.1 HIDL → AIDL 迁移路线（Android 11 开始）
- Android 11 宣布：新 HAL 接口优先使用 `Stable AIDL for HAL`（AIDL over libbinder_ndk），而非 HIDL
- HIDL 进入维护模式（不再开发新特性）
- 迁移工具：`hidl2aidl` 辅助转换现有 HIDL 接口

```bash
# AIDL HAL 文件结构示例
hardware/interfaces/vibrator/aidl/
├── android/hardware/vibrator/
│   ├── IVibrator.aidl
│   ├── Effect.aidl
│   └── EffectStrength.aidl
└── Android.bp
```

#### 5.1.2 HAL 实例注册
```cpp
// AIDL HAL 服务端注册（Android 11+）
int main() {
    ABinderProcess_setThreadPoolMaxThreadCount(0);
    std::shared_ptr<Vibrator> vibrator = ndk::SharedRefBase::make<Vibrator>();
    const std::string instance = std::string(Vibrator::descriptor) + "/default";
    binder_status_t status = AServiceManager_addService(vibrator->asBinder().get(), instance.c_str());
    ABinderProcess_joinThreadPool();
    return EXIT_FAILURE;
}
```

---

### 5.2 APEX 模块开发

#### 5.2.1 创建 APEX 模块的 Android.bp 配置
```
apex {
    name: "com.android.example",
    manifest: "apex_manifest.json",
    androidManifest: "AndroidManifest.xml",
    key: "com.android.example.key",
    certificate: ":com.android.example.certificate",
    native_shared_libs: ["libexample"],
    prebuilts: ["example_config.xml"],
    // 可选：指定挂载点
    package_name: "com.android.example",
}
```

#### 5.2.2 APEX 调试技巧
```bash
# 查看已挂载的 APEX
ls -la /apex/

# 激活特定 APEX（adb push 后）
adb push com.android.example.apex /data/apex/active/
adb reboot

# 查看 apexd 日志
adb logcat -s apexd

# 检查 APEX 签名
apexer --sign com.android.example.apex
```

---

### 5.3 SELinux 策略开发

#### 5.3.1 分区策略文件结构（Android 10+）
```
system/sepolicy/
├── public/        # system 分区公开接口（vendor 可引用）
├── private/       # system 分区私有策略
├── vendor/        # vendor 分区基础策略
├── reqd_mask/     # 必须的基础权限
system/sepolicy/system_ext/  # system_ext 分区
system/sepolicy/product/     # product 分区
device/<oem>/<device>/sepolicy/  # 设备特定策略
```

#### 5.3.2 常见调试命令
```bash
# 查看 AVC 拒绝日志
adb shell dmesg | grep avc
adb logcat | grep SELinux

# 生成 allow 规则建议
adb shell dmesg | grep avc | audit2allow

# 查看进程 SELinux 上下文
adb shell ps -Z | grep <process_name>

# 查看文件 SELinux 标签
adb shell ls -Z /vendor/lib/

# 验证 neverallow 规则
sepolicy-analyze -n /system/etc/selinux/plat_sepolicy.cil
```

#### 5.3.3 Treble 合规性验证
```bash
# 运行 VTS（Vendor Test Suite）合规测试
vts-tradefed run commandAndExit vts --module VtsHalVibratorV1_0Target

# 检查 VINTF manifest
cat /vendor/etc/vintf/manifest.xml
cat /system/etc/vintf/compatibility_matrix.xml

# 验证 HAL 接口合规性
lshal --version
```

---

### 5.4 存储隔离适配

#### 5.4.1 系统组件适配 Scoped Storage
```java
// 正确：使用 MediaStore 查询
Cursor cursor = getContentResolver().query(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
    new String[]{MediaStore.Images.Media._ID, MediaStore.Images.Media.DISPLAY_NAME},
    null, null, null
);

// 正确：获取 Uri 后用 ContentResolver 读取
Uri imageUri = ContentUris.withAppendedId(
    MediaStore.Images.Media.EXTERNAL_CONTENT_URI, id);
try (InputStream is = getContentResolver().openInputStream(imageUri)) {
    // 处理图片流
}
```

#### 5.4.2 FUSE 层调试
```bash
# 查看 FUSE 挂载
mount | grep fuse

# MediaProvider 进程
adb shell ps | grep media

# 查看 FUSE 日志
adb shell logcat -s MediaProvider

# 关闭 FUSE（仅调试，需 root）
adb shell setprop persist.sys.fuse false && adb reboot
```

---

### 5.5 Binder 通信调试

#### 5.5.1 Binder 状态查看
```bash
# 查看 Binder 线程统计
cat /proc/binder/stats

# 查看特定进程 Binder 状态
cat /proc/binder/proc/<pid>

# 查看 hwbinder（HAL 通信）
cat /proc/hwbinder/stats

# dumpsys 查看服务 Binder 信息
adb shell dumpsys activity binder-proxies

# 检查 Binder 引用泄漏
adb shell dumpsys meminfo <package> | grep Binder
```

#### 5.5.2 Binder 死亡通知
```java
// 监听服务死亡（系统开发常用）
IBinder binder = ServiceManager.getService("myservice");
binder.linkToDeath(new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        Log.e(TAG, "Service died! Attempting reconnect...");
        // 重新连接逻辑
    }
}, 0);
```

---

### 5.6 性能调优关键工具链

#### 5.6.1 Perfetto（Android 9+ 推荐，10 成熟）
```bash
# 录制系统级 trace
adb shell perfetto \
    -c - --txt \
    -o /data/misc/perfetto-traces/trace.perfetto-trace \
<<EOF
buffers: {
    size_kb: 63488
    fill_policy: DISCARD
}
data_sources: { config { name: "linux.ftrace" ftrace_config { ftrace_events: "sched/sched_switch" ftrace_events: "power/suspend_resume" atrace_categories: "gfx" atrace_categories: "am" atrace_categories: "wm" } } }
duration_ms: 10000
EOF

# 拉取 trace 文件，在 ui.perfetto.dev 分析
adb pull /data/misc/perfetto-traces/trace.perfetto-trace
```

#### 5.6.2 Simpleperf（NDK 性能分析）
```bash
# CPU 热点采样
adb shell simpleperf record -p <pid> -o /data/local/tmp/perf.data sleep 10
adb pull /data/local/tmp/perf.data
simpleperf report -i perf.data

# 系统级调用链
adb shell simpleperf record -a --call-graph dwarf -o /data/local/tmp/perf.data sleep 5
```

#### 5.6.3 tombstone 分析（native crash）
```bash
# 查看最新 tombstone
adb shell ls /data/tombstones/
adb pull /data/tombstones/tombstone_00

# 用 ndk-stack 解析
adb logcat | ndk-stack -sym $PROJECT/obj/local/arm64-v8a/

# Android 11 新增：从 exitReason API 获取 trace
// ApplicationExitInfo.getTraceInputStream() 直接读取
```

---

### 5.7 Wi-Fi 系统开发

#### 5.7.1 WifiService 调试
```bash
# 查看 Wi-Fi 状态机日志
adb shell dumpsys wifi

# Wi-Fi 详细日志
adb shell wpa_cli status
adb shell wpa_cli list_networks

# 扫描结果
adb shell dumpsys wifi | grep -A 20 "Scan Results"

# 查看 APEX 模块版本
adb shell pm list packages --apex-only | grep wifi
```

#### 5.7.2 Network Stack 调试
```bash
# ConnectivityService 状态
adb shell dumpsys connectivity

# NetworkStack 日志
adb shell dumpsys network_stack

# 网络评估状态（Captive Portal 检测）
adb shell dumpsys network_stack | grep -A 5 "NetworkMonitor"
```

---

### 5.8 Android 11 APEX 扩展模块列表

以下模块在 Android 11 作为 Mainline 管理（相比 Android 10 新增）：

| 新增模块 | 功能 |
|----------|------|
| `com.android.cellbroadcast` | 紧急广播 |
| `com.android.extservices` | 文字分类、智能回复等 |
| `com.android.neuralnetworks` | NNAPI 神经网络加速 |
| `com.android.permission` | PermissionController |
| `com.android.wifi` | Wi-Fi 协议栈 |
| `com.android.ipsec` | IPsec/IKEv2 VPN |
| `com.android.tethering` | 热点共享 |

---

## 附录：关键系统组件变更速查表

### A. 废弃 API 与替代方案

| 废弃 API | 版本 | 替代方案 |
|----------|------|----------|
| `FingerprintManager` | 9 | `BiometricPrompt` |
| `WifiConfiguration` | 10（弃用方向） | `WifiNetworkSuggestion` |
| `View.setSystemUiVisibility()` | 11 | `WindowInsetsController` |
| `Environment.getExternalStorageDirectory()` | 10 | `Context.getExternalFilesDir()` |
| `Build.getSerial()` （普通 App） | 10 | 无（需特权） |
| `READ_EXTERNAL_STORAGE` （广义） | 10/11 | MediaStore / SAF |

### B. 关键系统服务变更路径

| 服务 | Android 9 位置 | Android 10 变化 | Android 11 变化 |
|------|----------------|-----------------|-----------------|
| ART | /system/lib/libart.so | APEX: com.android.runtime | 延续 |
| DNS Resolver | netd 进程内 | APEX: com.android.resolv | 延续 |
| Wi-Fi | SystemServer 内 | APEX: com.android.wifi（部分） | 完全独立 |
| MediaProvider | /system/app/ | Mainline 模块 | 延续+FUSE 强制 |
| Conscrypt | /system/lib/ | APEX: com.android.conscrypt | 延续 |
| PermissionController | /system/priv-app/ | 独立 | APEX: com.android.permission |

### C. 内核版本要求

| Android 版本 | 最低内核要求 | 推荐内核 |
|-------------|-------------|----------|
| Android 9 | 4.4 | 4.9 / 4.14 |
| Android 10 | 4.9 | 4.14 / 4.19 |
| Android 11 | 4.14 | 5.4 |

### D. 构建系统关键变化

| 变化 | 版本 | 说明 |
|------|------|------|
| Soong 构建系统成熟 | 9 | `Android.bp` 全面替代 `Android.mk` |
| APEX 构建支持 | 10 | `apex {}` 模块类型 |
| Rust 语言支持 | 11 | Android.bp 支持 `rust_*` 模块类型 |
| `hidl2aidl` 工具 | 11 | HIDL→AIDL 迁移辅助 |

---

## 参考资源

- [Android Source - AOSP](https://source.android.com/)
- [Android 9 Release Notes](https://source.android.com/docs/setup/start/android-9-0-changes)
- [Android 10 Release Notes](https://source.android.com/docs/setup/start/android-10-0-changes)
- [Android 11 Release Notes](https://source.android.com/docs/setup/start/android-11-release-notes)
- [Project Mainline - APEX](https://source.android.com/docs/core/ota/modular-system-components)
- [SELinux for Android](https://source.android.com/docs/security/features/selinux)
- [HIDL to AIDL Migration](https://source.android.com/docs/core/architecture/aidl/aidl-hals)
- [Scoped Storage](https://developer.android.com/about/versions/11/privacy/storage)
- [Perfetto Tracing](https://perfetto.dev/docs/)
- [Android Treble](https://source.android.com/docs/core/architecture/treble)
