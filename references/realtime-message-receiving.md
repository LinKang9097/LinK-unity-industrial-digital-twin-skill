# 实时消息接收

适用于 Unity 端通过 WebSocket、MQTT 或 Redis 接收后端推送。这个模块的职责是把外部消息可靠地带进 Unity，并转换成项目内部事件或状态更新。

生产环境中 MQTT 和 WebSocket 不应同时开启：MQTT 用于接收现场实时数据，WebSocket 用于接收服务器推送的历史回放数据。二者传入 Unity 后的数据格式保持一致，后续解析、状态更新和场景驱动走同一条内部链路。Redis 通常不启用，但需要在配置和代码结构中预留。

## 职责边界

负责：

- 建立连接、断线重连、心跳、订阅和取消订阅。
- 从 `StreamingAssets/DataDockingConfig.json` 读取通信配置。
- 接收原始消息并反序列化。
- 校验消息类型、时间戳、实体 ID 和载荷。
- 将网络线程收到的数据切换到 Unity 主线程。
- 分发为统一事件，例如设备状态、告警、库存变化、任务进度、位置更新。

不负责：

- 不直接移动设备模型。
- 不直接打开 UI 或刷新 Text。
- 不直接创建或删除库存货物。
- 不把后端业务规则重新实现到 Unity。
- 不管理 PLC 通信、读写区、握手、清除指令或任务下发流程，这些由后端处理。

## 协议建议

MQTT 和 WebSocket 接收到的数据格式应保持一致，区别只在数据来源：MQTT 是现场实时数据，WebSocket 是服务器历史回放数据。接入层不要让后续模块感知当前来源。

接收到的业务消息分两类：

- AGV 消息：使用 AGV 单独格式。
- 其余设备消息：输送线、提升机、堆垛机、RGV、桁架、机械手、拆码盘机、分拣线等共用一种格式。

接入层应先判断消息属于 AGV 还是普通设备，再转换成统一内部事件或状态。不要让设备表现层直接解析 AGV 原始消息或普通设备原始消息。

AGV 原始消息中如果包含 `amrCode`、`mapCode`、`cooX`、`cooY`、`direction`、`alarm`、`battery`、`speed`、`path`、`target`、`online`、`pause`、`taskType`、`timestamp` 等字段，接入层应转换为内部 AGV 状态，例如 `AgvId`、`MapId`、源坐标、源朝向、告警、在线状态、电量和速度。地图原点换算、车型 prefab 选择和动态生成由 AGV 管理器处理，不放在 MQTT 或 WebSocket 回调里。

普通设备原始消息如果包含 `DeviceNo`、`Alarm`、`Load`、`State`、`Source`、`Target`、`IsForward`、`IsReverse`、`PosXaxis`、`PosYaxis`、`PosZaxis`、`PosRaxis`、`TaskNo`、`MaterialType`、`ContainerType`、`ContainerNumber` 等字段，接入层只负责反序列化和转换为内部状态。字段来自哪一段 PLC 数据区、读写区地址如何定义、WCS 如何下发和确认任务，都不属于 Unity 接入层职责。

如果项目允许后端统一信封，可以使用类似结构：

```json
{
  "type": "device.state.changed",
  "messageId": "msg-20260902-0001",
  "timestamp": "2026-09-02T10:15:30.000Z",
  "entityId": "CV-001",
  "payload": {
    "state": "running",
    "speed": 0.8
  }
}
```

建议支持两类消息：

- 全量快照：初始化或重连后同步当前状态。
- 增量更新：实时推送局部变化。

Unity 端应能容忍重复消息、乱序消息、缺失字段、未知消息类型和设备未注册。

## 外部配置

三种通信方式都应通过外部配置控制，默认读取 Unity 项目的 `Assets/StreamingAssets/DataDockingConfig.json`。不要在场景脚本里写死 IP、端口、topic、WebSocket URL 或 Redis key。

配置可以包含：

```json
{
  "autoStart": true,
  "mqtt": {
    "enable": false,
    "brokers": [
      {
        "enable": true,
        "name": "本地",
        "ip": "127.0.0.1",
        "port": "1883",
        "subscriptions": [
          {
            "topic": "huanggang/slp/device/data",
            "qos": 0
          }
        ]
      }
    ]
  },
  "websocket": {
    "enable": true,
    "url": "ws://localhost:1880/ws/slp/data/replay"
  },
  "redis": {
    "enable": false,
    "ip": "127.0.0.1",
    "port": "6379"
  }
}
```

配置读取规则：

- `autoStart` 为 true 时，模块初始化后按配置自动启动通信。
- `mqtt.enable`、`websocket.enable`、`redis.enable` 控制各通信方式是否启用。
- 生产环境中 `mqtt.enable` 和 `websocket.enable` 不应同时为 true；如果配置同时开启，应记录错误并按项目策略只启用一个。
- Redis 默认作为预留能力，通常保持 `redis.enable = false`。
- 配置应在启动时读取并缓存；需要运行时切换配置时，提供显式重新加载入口。
- Windows 编辑器和 PC 包通常可直接读取 `StreamingAssets` 文件；Android、WebGL 等平台可能需要通过 `UnityWebRequest` 读取。

## WebSocket

WebSocket 用于从服务器接收历史回放数据。它和 MQTT 的业务消息格式一致，区别是数据来源和使用场景。

实现时注意：

- 连接配置不要写死在场景脚本中。
- 心跳和重连策略独立封装。
- 收到消息后进入消息队列，由主线程消费。
- 断线时向状态层发出连接状态变化事件，供 UI 显示。
- 可以支持开始时间、结束时间、初始消息、ping 消息等回放配置，但这些配置只属于 WebSocket 接入层。

## MQTT

MQTT 用于接收现场实时数据，适合设备状态、告警、任务事件和按主题订阅的场景。MQTT 的 QoS 默认使用 0，除非现场协议或项目配置明确要求其他值。

MQTT 应支持多个 broker 配置，每个 broker 可以连接不同 IP 和端口，并订阅不同 topic。不要让每个设备对象各自建立 MQTT 连接，应由 MQTT 接入层统一管理连接和订阅。

Topic 命名可以按项目约定，但要保持稳定和可过滤，例如：

```text
twin/site/{siteId}/device/{deviceId}/state
twin/site/{siteId}/device/{deviceId}/alarm
twin/site/{siteId}/inventory/{areaId}/changed
twin/site/{siteId}/task/{taskId}/progress
```

实现时注意：

- 订阅逻辑与设备表现解耦。
- QoS 默认 0；保留消息、离线消息要和后端约定清楚。
- 支持 `brokers` 多连接配置，并跳过 `enable = false` 的 broker。
- 每个 broker 独立处理连接、重连、初始化消息和订阅列表。
- 不要让每个设备对象各自建立 MQTT 连接。
- Topic 只用于路由，业务含义仍以消息体为准。

## Redis

Redis 通常不建议让 Unity 直接连接生产 Redis，除非项目架构明确允许。更常见做法是由后端订阅 Redis 后，再通过 WebSocket 或 MQTT 推送给 Unity。

如果确实需要 Unity 接 Redis：

- 只用于受控内网或开发环境。
- 连接信息必须配置化。
- 不在 Unity 端写入关键业务数据。
- Redis Pub/Sub 消息仍要转换成统一内部事件。
- 可以预留订阅 topic、hash 轮询、轮询间隔和只在值变化时通知等配置，但默认不启用。

## 线程与主线程派发

MQTT 和 WebSocket 的接收回调通常运行在子线程，而 Unity API 必须在主线程调用。主线程切换应该在实时消息接收模块内统一处理，模块对外只派发主线程上的内部事件或状态更新。

推荐流程：

```text
子线程收到 MQTT/WebSocket 消息
-> 只做轻量入队或原始文本缓存
-> Unity 主线程 Update/LateUpdate 消费队列
-> 解析或分发为内部事件
-> 状态层、设备驱动、UI、库存等模块响应
```

如果反序列化开销较大，可以在子线程完成纯数据解析，但任何 Unity API、GameObject、Transform、Material、UI 访问都必须放到主线程。

```csharp
// 中文注释：网络回调只入队，不直接调用 Unity API。
private readonly ConcurrentQueue<string> messageQueue = new ConcurrentQueue<string>();

// 中文注释：MQTT 或 WebSocket 子线程收到消息后，把原始消息放入队列。
private void OnMessageReceivedInWorkerThread(string message)
{
    messageQueue.Enqueue(message);
}

// 中文注释：Unity 主线程中消费消息，再分发给状态层和表现层。
private void Update()
{
    while (messageQueue.TryDequeue(out var message))
    {
        DispatchOnMainThread(message);
    }
}
```

## 内部事件

不要让表现层解析原始 JSON。接入层应输出项目内部事件，例如：

```csharp
// 中文注释：实时接入层输出统一事件，表现层不需要知道消息来自 MQTT 还是 WebSocket。
public sealed class DeviceStateChangedEvent
{
    public string EntityId;
    public string State;
    public float Speed;
    public DateTime Timestamp;
}
```

AGV 和普通设备可以有不同原始 DTO，但输出到设备驱动前应转成统一的内部状态或事件。设备驱动只关心实体 ID、设备类型、位置、状态、任务、载货等运行信息，不关心消息来自 MQTT、WebSocket 还是 Redis。

## 验证重点

- 后端断开后能重连。
- 重连后能重新订阅并请求快照。
- 网络线程不会直接操作 Unity 对象。
- MQTT 和 WebSocket 同时开启时能报错或按项目策略降级为单通道。
- MQTT 支持多个 broker，并能分别订阅不同 topic。
- MQTT 订阅 QoS 默认 0。
- Redis 配置存在但默认不启用。
- AGV 消息和普通设备消息能分别解析，并转换为统一内部事件。
- 错误消息不会导致主循环异常。
- 高频消息不会导致帧率明显下降。
