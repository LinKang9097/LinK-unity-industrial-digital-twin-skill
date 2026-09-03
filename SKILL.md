---
name: unity-industrial-digital-twin
description: 在 Unity 中实现工业数字孪生前端时使用，适用于实时消息接收、HTTP 面板数据、设备动作、告警、UI、库存和相机控制；不负责后端采集、存储或业务计算。
---

# Unity 工业数字孪生

用于指导 Codex 开发或改造 Unity 工业数字孪生前端。默认边界是：后端负责数据采集、清洗、保存、鉴权和业务计算；Unity 只负责接收后端推送或查询得到的数据，并驱动三维场景、设备状态、告警、库存对象、UI 和相机交互。

优先保持模块分层，不要让 MQTT、WebSocket、Redis、HTTP、UI、设备动画和库存逻辑混在同一个脚本里。所有外部数据进入 Unity 后，先转换为项目内统一的状态、事件或 DTO，再由表现层消费。

推荐数据流：

```text
后端数据源 -> 接入层 -> 消息/数据模型 -> 本地状态 -> 场景对象/UI/相机/库存表现
```

## 渐进式披露

只读取当前任务需要的参考文件：

- 处理 WebSocket、MQTT、Redis、`StreamingAssets/DataDockingConfig.json`、实时/历史数据接收、AGV/普通设备消息解析、主线程派发时，读取 [references/realtime-message-receiving.md](references/realtime-message-receiving.md)。
- 处理 HTTP 查询、面板详情、列表、统计数据、`StreamingAssets/UrlConfig.json`、中文接口名检索时，读取 [references/panel-data-http.md](references/panel-data-http.md)。
- 处理输送线、提升机、AGV、堆垛机、RGV、桁架/垳架、机械手、拆码盘机、分拣线、设备协议读取字段映射、起终点站台、初始化货物生成、AGV/非 AGV 设备基类、普通设备管理器、AGV 管理器、AGV 动态生成、多地图原点、初始化定位、XYZ 坐标驱动、载货点、任务号/托盘条码流转、未扫码临时任务等设备动作时，读取 [references/equipment-action-driving.md](references/equipment-action-driving.md)。
- 处理设备标红、闪烁、报警内容、报警列表、报警恢复时，读取 [references/equipment-alarms.md](references/equipment-alarms.md)。
- 处理常驻面板、弹窗、界面调用、数据显示、界面关闭时，读取 [references/ui-control.md](references/ui-control.md)。
- 处理平库/立库、货架绑定、货架筛选、库存货物生成、多货物模型、排列层深批量生成、货物高亮、在途货物、货物信息展示、取放货增减、库位状态时，读取 [references/inventory-management.md](references/inventory-management.md)。
- 处理多楼层、楼层切换、楼层可见性、楼层相机视角、设备/货物随楼层显示隐藏时，读取 [references/floor-management.md](references/floor-management.md)。
- 处理自由相机、平滑移动旋转、俯视正交视角、场景漫游、设备漫游、定位、跟随、视角复位时，读取 [references/camera-control.md](references/camera-control.md)。
- 当功能边界不清楚或多个模块可能重叠时，读取 [references/integration-boundaries.md](references/integration-boundaries.md)。

## 通用架构原则

- 接入层只负责连接、重连、反序列化、主线程派发和消息分发，不直接操作场景对象或 UI。
- HTTP 服务只负责请求型数据，例如详情、列表、历史、统计和弹窗加载数据，不承担高频实时同步。
- 设备动作模块只负责设备如何移动、旋转、升降、输送、抓取或切换动画，不决定库存真相。
- 库存模块只负责货物、库位、托盘、容器等库存表现和状态，不把搬运动作写进库存脚本。
- 告警模块是异常视觉覆盖层，应能覆盖设备正常表现，并在告警恢复后还原。
- UI 模块只负责界面生命周期、用户输入和数据绑定，不解析底层协议。
- 楼层模块负责当前楼层上下文和楼层可见性协调，不直接实现相机运动、设备动作或库存增减；非当前楼层设备隐藏时仍应继续接收状态并正常运行。
- 相机模块只负责视角行为，可以被 UI、场景点击或业务事件触发。

## Unity 实现要求

- 网络回调不能直接修改 Unity 对象；需要切回 Unity 主线程。
- 场景对象通过稳定的 `entityId`、`deviceId`、`slotId` 或 `cargoId` 绑定数据。
- 为无后端开发保留模拟数据入口，但模拟数据应走同一套状态和事件通道。
- 高频状态要做节流、合并、插值或补间，避免每条消息都刷新 UI、材质或 Transform。
- 写代码时不要过度保守，不要为了所有理论异常添加大量兜底、分支和空判断；用户会在后端、配置、协议和现场数据约定中处理好的部分，Unity 代码按明确约定实现即可。只有会直接导致 Unity 报错、现场无法排查或数据进入错误状态的情况，才添加必要校验和清晰日志。
- 代码注释使用中文；公共类型、字段和方法名可使用项目既有命名规范。
- 所有方法都需要添加 XML 文档注释，统一使用 `/// <summary>` 形式；有参数时必须用 `/// <param name="参数名">参数含义</param>` 标明参数含义。
- 除 WCS 协议字段实体类、后续请求接口返回参数实体类外，其他字段都需要添加中文行尾注释，格式类似 `public string robotId; // AGV唯一标识`。
- 设备动作，尤其是取放货动作，应优先在当前设备自己的脚本或子类中处理；不要为了多个同类设备新建一个集中管理所有取放货动作和流转的大脚本。
- 如果项目已有通信框架、UI 框架、依赖注入、事件总线或状态管理方案，优先沿用。

示例：

```csharp
public string robotId; // AGV唯一标识

/// <summary>
/// 根据设备状态刷新当前设备表现。
/// </summary>
/// <param name="state">设备运行状态。</param>
public void ApplyState(DeviceRuntimeState state)
{
}
```

## 常见目录建议

```text
Assets/Scripts/Networking/
Assets/Scripts/Messaging/
Assets/Scripts/Http/
Assets/Scripts/Twin/
Assets/Scripts/Devices/
Assets/Scripts/Alarms/
Assets/Scripts/Inventory/
Assets/Scripts/Floors/
Assets/Scripts/UI/
Assets/Scripts/Camera/
Assets/Scripts/Simulation/
Assets/StreamingAssets/Config/
```

这些目录只是默认建议。已有项目的组织方式优先级更高。
