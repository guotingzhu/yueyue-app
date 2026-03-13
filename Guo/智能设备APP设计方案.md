# 智能设备 APP 架构设计方案

> 适用场景：智能 IoT 设备管理 APP（类似华为智慧生活、荣耀智慧空间）
> 文档版本：v1.0
> 日期：2026-03-13

---

## 一、架构分层原则

### 1.1 采用纵向切片（Vertical Slice）而非横向分层

```
横向分层（不推荐）            纵向切片（推荐）
  presentation layer    →     feature-order/
  domain layer          →     feature-profile/
  data layer            →     feature-scene/
```

**理由：** 纵向切片高内聚，feature 可独立开发、测试，未来可提取为独立 Git 仓库发布 aar。

### 1.2 整体层次结构

```
app                         组装层，DI 注册，路由组合
feature-xxx                 业务功能模块（纵向切片）
lib-xxx                     基础设施层（无业务含义）
```

---

## 二、模块结构设计

### 2.1 每个 feature 的标准拆分

```
feature-order/
  feature-order-api/        对外暴露的契约（接口、实体、路由）
    build.gradle.kts
    OrderService.kt         接口
    OrderEntity.kt          Domain 实体（对外）
    OrderRoute.kt           路由定义

  feature-order-impl/       业务实现（UI、ViewModel、UseCase、Repository）
    build.gradle.kts
    OrderViewModel.kt
    OrderRepositoryImpl.kt

  data-order/               数据层，归属于本 feature
    build.gradle.kts
    OrderTable.kt           (internal) Room Entity，跨模块不可见
    OrderDao.kt             (internal)
```

**关键原则：**
- 每个子目录有独立的 `build.gradle.kts`，是独立编译单元
- `data-xxx` 内部类用 `internal` 修饰，防止跨模块误用
- 其他 feature 只依赖 `-api` 模块，不感知 `-impl` 和 `data`
- `internal` 防止跨模块访问，但无法阻止非法依赖声明，需配合 **Gradle 验证任务 + Code Review** 约束

### 2.2 Feature 间通信原则

> **谁是数据 owner，契约就定义在谁的 `-api` 里**

```kotlin
// feature-auth-api
data class UserLoggedOutEvent : AppEvent   // auth 是 owner，定义在这里

// feature-order-impl 监听
eventBus.subscribe<UserLoggedOutEvent> { clearOrders() }

// feature-profile-impl 监听
eventBus.subscribe<UserLoggedOutEvent> { clearProfile() }
```

多个 feature 监听同一事件是正常的，**使用方多不代表定义权分散**。

---

## 三、完整模块划分

### 3.1 业务域划分

| 域 | 职责 | 是否需要本地DB |
|----|------|--------------|
| 设备域 | 设备管理、状态、配置、多协议适配 | 是（DeviceDatabase） |
| 场景域 | 场景、自动化规则、触发条件、执行动作 | 是（SceneDatabase） |
| 消息域 | 通知、告警、操作日志 | 是（MessageDatabase） |
| 插件域 | Native插件、RN插件、版本管理 | 是（PluginDatabase） |
| 产品展示域 | 商城、产品发现、产品详情 | 否（纯网络+内存缓存） |
| 固件域 | OTA升级 | 否（纯网络） |
| 首页域 | 聚合展示 | 否 |
| 个人中心域 | 用户信息展示 | 否 |

### 3.2 完整目录结构

```
project-root/
├── app/
│   └── build.gradle.kts

├── library/
│   ├── lib-storage/            AppDatabase（全局基础数据）
│   ├── lib-network-http/       REST API 封装（Retrofit/OkHttp）
│   ├── lib-network-mqtt/       MQTT 封装（隔离三方SDK）
│   ├── lib-network-coap/       CoAP 协议封装
│   ├── lib-network-common/     公共网络模型（NetworkResult、ApiResponse）
│   ├── lib-event/              事件总线基础设施
│   ├── lib-plugin-rn/          RN 容器基础设施
│   └── lib-utils/

├── feature-device/
│   ├── feature-device-api/
│   ├── feature-device-impl/
│   └── data-device/            DeviceDatabase

├── feature-scene/
│   ├── feature-scene-api/
│   ├── feature-scene-impl/
│   └── data-scene/             SceneDatabase

├── feature-message/
│   ├── feature-message-api/
│   ├── feature-message-impl/
│   └── data-message/           MessageDatabase

├── feature-plugin/
│   ├── feature-plugin-api/
│   ├── feature-plugin-impl/
│   └── data-plugin/            PluginDatabase

├── feature-product/            纯网络+内存缓存，依赖 ProductRepository 写缓存
│   ├── feature-product-api/
│   └── feature-product-impl/

├── feature-firmware/           纯网络
│   ├── feature-firmware-api/
│   └── feature-firmware-impl/

├── feature-home/               聚合展示，无独立DB
│   ├── feature-home-api/
│   └── feature-home-impl/

└── feature-profile/            读 UserRepository，无独立DB
    ├── feature-profile-api/
    └── feature-profile-impl/
```

---

## 四、数据库设计

### 4.1 数据库划分策略

> **共享基础数据放 AppDatabase，业务私有数据放 feature 自己的 Database**

| 数据库 | 归属 | 包含的表 |
|--------|------|---------|
| AppDatabase | lib-storage | User、Family、Room、Member、Product、ProductProfile |
| DeviceDatabase | data-device | Device、DeviceState、DeviceConfig、Group |
| SceneDatabase | data-scene | Scene、Automation、Trigger、Action |
| MessageDatabase | data-message | Notification、Alarm、OperationLog |
| PluginDatabase | data-plugin | Plugin、PluginState |

### 4.2 AppDatabase 详细设计

产品（Product）数据定性为**全局基础数据**，原因：
- 设备添加、展示、连接过程都依赖产品配置 Profile
- 被设备域、场景域、产品展示域等多个模块频繁读取
- 性质类似字典/元数据，不属于任何单一 feature 的私有数据

```kotlin
@Database(entities = [
    UserTable::class,
    FamilyTable::class,
    RoomTable::class,
    MemberTable::class,
    ProductTable::class,
    ProductProfileTable::class
])
class AppDatabase : RoomDatabase()
```

**ProductProfileTable 用 key-value 存储而非大 JSON：**
```
ProductTable
  productId
  name
  imageUrl
  category          耳机 / 手表 / WiFi设备 / CoAP设备
  updatedAt

ProductProfileTable
  productId         FK → ProductTable
  key               配置项名称
  value             配置项值
  type              STRING / INT / BOOL / JSON
```

理由：
- 不同产品 Profile 字段差异大，key-value 更灵活
- 支持按 key 单独查询（如连接流程只取连接相关配置）
- 方便增量更新单个配置项

### 4.3 AppDatabase 访问方式

**所有 feature 通过 Repository 接口访问，不直接操作 Dao：**

```
lib-storage/
  AppDatabase
  ProductRepository    ← 产品数据唯一读写入口
  UserRepository       ← 用户数据唯一读写入口
  FamilyRepository     ← 家庭数据唯一读写入口
```

```kotlin
// feature 依赖 Repository 接口，不直接依赖 Dao
class DeviceRepositoryImpl(
    private val productRepo: ProductRepository   // 读产品配置
)
```

### 4.4 DeviceStateTable 特殊处理

设备状态是高频读写数据（设备可能每秒上报多次）：

```
DeviceStateTable  →  内存缓存（StateFlow/Map）为主 + 定时持久化
DeviceTable       →  Room 持久化（低频写）
```

---

## 五、网络层设计

### 5.1 lib-network 内部拆分

```
library/
  lib-network-http/       REST API（Retrofit/OkHttp封装）
  lib-network-mqtt/       MQTT封装（隔离三方SDK防腐层）
  lib-network-coap/       CoAP协议封装（局域网直连）
  lib-network-common/     公共模型（NetworkResult、ApiResponse等）
```

### 5.2 MQTT 防腐层封装

MQTT 为公司其他部门提供的三方 SDK，必须通过防腐层隔离：

```kotlin
// lib-network-mqtt：防腐层接口
interface MqttService {
    fun subscribe(topic: String): Flow<MqttMessage>
    fun publish(topic: String, payload: ByteArray)
    fun connectionState(): Flow<ConnectionState>
}

class MqttServiceImpl(
    private val sdk: 三方MqttClient   // 只有这里依赖三方SDK
) : MqttService

// feature 只依赖 MqttService 接口，完全不感知三方SDK
class DeviceRepositoryImpl(
    private val mqtt: MqttService
)
```

**好处：** 三方 SDK 升级或替换时，只改 `lib-network-mqtt` 内部实现，所有 feature 不受影响。

### 5.3 CoAP 与 MQTT 的区别

```
MQTT  → 设备 → 云端服务器 → App（广域网，云端中转）
CoAP  → 设备 → App 直连（局域网，低延迟）
```

---

## 六、设备多协议适配

设备类型包含：儿童手表、蓝牙耳机、WiFi设备、CoAP设备，使用**策略模式**隔离协议差异：

```
feature-device-impl/
  protocol/
    DeviceProtocol.kt       interface，统一协议抽象
    BleProtocol.kt          蓝牙（手表、耳机）
    WifiProtocol.kt         WiFi 设备
    CoapProtocol.kt         CoAP 设备
```

上层业务通过 `DeviceService` 统一操作，**完全不感知底层协议差异**。

---

## 七、插件系统设计

### 7.1 插件分类

| 类型 | 加载方式 | 特点 |
|------|---------|------|
| Native 插件 | 动态加载 .so 或独立 module | 性能高，与系统耦合 |
| RN 插件 | 加载 JS Bundle | 动态下发，热更新 |

### 7.2 模块结构

```
feature-plugin/
  feature-plugin-api/
    PluginService.kt          统一插件接口
    PluginEntity.kt           含 type: NATIVE / RN

  feature-plugin-impl/
    native/
      NativePluginLoader.kt
    rn/
      RNPluginLoader.kt
      RNBundleManager.kt      JS Bundle 下载、版本管理

library/
  lib-plugin-rn/              RN 容器基础设施（下沉到 library，复用性高）
```

---

## 八、事件系统设计

```
lib-event/                    只放机制，零业务
  AppEvent                    标记接口
  AppEventBus                 SharedFlow 实现
```

**业务事件定义原则：**
- 全局基础设施事件 → `lib-event`
- 业务事件 → 发出方的 `-api` 模块

```
feature-auth-api/
  UserLoggedOutEvent          auth 是 owner，定义在这里

feature-order-impl            依赖 feature-auth-api 监听事件
feature-profile-impl          依赖 feature-auth-api 监听事件
```

---

## 九、跨域数据查询原则

不同数据库之间无法 JOIN，跨域查询在 Repository 层组合：

```kotlin
// 查询某房间设备触发的场景
val devices = deviceRepository.getByRoom(roomId)
val scenes = sceneRepository.getByDeviceIds(devices.map { it.id })
```

---

## 十、settings.gradle.kts 模块注册示例

```kotlin
include(
    ":app",

    // feature-device
    ":feature-device:feature-device-api",
    ":feature-device:feature-device-impl",
    ":feature-device:data-device",

    // feature-scene
    ":feature-scene:feature-scene-api",
    ":feature-scene:feature-scene-impl",
    ":feature-scene:data-scene",

    // feature-message
    ":feature-message:feature-message-api",
    ":feature-message:feature-message-impl",
    ":feature-message:data-message",

    // feature-plugin
    ":feature-plugin:feature-plugin-api",
    ":feature-plugin:feature-plugin-impl",
    ":feature-plugin:data-plugin",

    // feature-product
    ":feature-product:feature-product-api",
    ":feature-product:feature-product-impl",

    // feature-firmware
    ":feature-firmware:feature-firmware-api",
    ":feature-firmware:feature-firmware-impl",

    // feature-home
    ":feature-home:feature-home-api",
    ":feature-home:feature-home-impl",

    // feature-profile
    ":feature-profile:feature-profile-api",
    ":feature-profile:feature-profile-impl",

    // library
    ":library:lib-storage",
    ":library:lib-network-http",
    ":library:lib-network-mqtt",
    ":library:lib-network-coap",
    ":library:lib-network-common",
    ":library:lib-event",
    ":library:lib-plugin-rn",
    ":library:lib-utils"
)
```

---

## 十一、待决策项

- [ ] CoAP 是三方 SDK 还是自研？如是局域网直连，需考虑设备发现（mDNS/SSDP）归属
- [ ] `AppDatabase` 是否需要按表拆分为多个 DAO 文件分人维护
- [ ] RN 插件的 JS Bundle 存储路径和安全校验方案
- [ ] 设备状态持久化的定时策略（间隔时间、触发条件）
- [ ] Gradle 依赖约束验证任务的具体实现方案
