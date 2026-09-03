# 设备动作驱动

适用于把统一设备状态转换为 Unity 场景中的设备动作、动画和位置变化。设备动作驱动不关心数据来自 MQTT、WebSocket、Redis 还是 HTTP，只消费内部状态或事件。

## 设备类型

常见设备可以拆成独立驱动：

- 输送线：滚筒、皮带、链条、托盘移动、堵塞状态。
- 提升机：升降、楼层定位、载货状态、门开关。
- AGV：路径移动、朝向、载货、充电、等待、异常。
- 堆垛机：横移、升降、货叉伸缩、取放货。
- RGV：轨道移动、站点停靠、载货。
- 桁架/垳架：X/Y/Z/R 轴移动、抓取、放置。
- 机械手：关节动画、夹爪、抓取点、放置点。
- 拆码盘机：托盘堆叠、拆盘、码盘、升降动作。
- 分拣线：分流口、挡板、摆轮、包裹流向。

## 设备分类与基础类

设备建议先分为两条继承链：

- 非 AGV 设备：输送线、提升机、堆垛机、RGV、桁架、机械手、拆码盘机、分拣线等已提前摆放在场景中的设备。
- AGV 设备：运行时根据 AGV 数据动态生成，且可能存在 CMR、FMR、LMR 等多种车型或业务类型。

非 AGV 设备共用一个普通设备基类。基类只放所有普通设备都稳定拥有的能力，不把堆垛机、输送线、机械手等差异逻辑塞进基类。

```csharp
// 中文注释：普通设备基类只定义通用身份、告警、载货点和数据入口契约。
public abstract class BaseDevice : MonoBehaviour, IDeviceCargoMountProvider
{
    public string DeviceId;
    public string DeviceType;
    public bool IsAlarm;

    // 中文注释：设备可配置多个载货点，例如入口、出口、货叉或缓存位。
    public List<DeviceCargoMountPoint> CargoMountPoints = new List<DeviceCargoMountPoint>();

    public abstract void ShowDeviceIdUi(bool visible);
    public abstract void ApplyDeviceData(DeviceRuntimeState state, bool isInitialize);

    public Transform GetCargoMountPoint(string mountPointId)
    {
        return FindCargoMountPoint(mountPointId);
    }
}
```

普通设备子类负责实现：

- 是否贴设备 ID 的 UI，例如设备铭牌、悬浮标签或调试标签。
- 设备数据接收后的逻辑处理，例如输送线托盘段、堆垛机三轴、提升机楼层、机械手动画。
- 设备自身的初始化定位和平滑动作。

AGV 单独使用 AGV 基类。AGV 协议中常见字段可参考外部样例，例如 `amrCode`、`mapCode`、`amrCategory`、`amrType`、`cooX`、`cooY`、`direction`、`alarm`、`battery`、`speed`、`path`、`target`、`online`、`pause`、`taskType`、`timestamp` 等。不要在基类里绑定所有厂家字段，基类保留 Unity 表现层稳定需要的字段，原始 DTO 由解析层转换。

```csharp
// 中文注释：AGV 基类承载所有车型共有的状态，具体车型处理自己的外观和扩展字段。
public abstract class BaseAgv : MonoBehaviour, IDeviceCargoMountProvider
{
    public string AgvId;
    public string MapId;
    public string AgvType;
    public string AgvCategory;
    public bool IsAlarm;
    public bool IsOnline;
    public float Battery;
    public float Speed;
    public Vector3 SourcePosition;
    public float SourceDirection;

    // 中文注释：AGV 载货点通常位于车体子物体下，货物挂载后跟随车辆移动。
    public List<DeviceCargoMountPoint> CargoMountPoints = new List<DeviceCargoMountPoint>();

    public abstract void ApplyAgvData(AgvRuntimeState state, bool isInitialize);
    public abstract void ShowDeviceIdUi(bool visible);

    public Transform GetCargoMountPoint(string mountPointId)
    {
        return FindCargoMountPoint(mountPointId);
    }
}
```

AGV 运行状态应从原始消息转换出来：

```csharp
// 中文注释：AGV 内部状态屏蔽外部协议差异，设备表现层只消费稳定字段。
public sealed class AgvRuntimeState
{
    public string AgvId;
    public string MapId;
    public string AgvType;
    public string AgvCategory;
    public bool IsAlarm;
    public bool IsOnline;
    public float Battery;
    public float Speed;
    public Vector3 SourcePosition;
    public float SourceDirection;
    public string PathJson;
    public string TargetJson;
    public long Timestamp;
}
```

## 设备管理器

普通设备和 AGV 使用两个总管理脚本，不要混在同一个大管理器里。

普通设备管理器负责场景中已存在设备的注册、查找和批量操作：

```csharp
// 中文注释：普通设备启动时从指定父物体读取，后续通过设备 ID 查找。
public sealed class NormalDeviceManager : MonoBehaviour
{
    public Transform DeviceRoot;
    private readonly Dictionary<string, BaseDevice> devices = new Dictionary<string, BaseDevice>();

    public void RegisterSceneDevices()
    {
        foreach (var device in DeviceRoot.GetComponentsInChildren<BaseDevice>(true))
        {
            Register(device);
        }
    }

    public bool TryGetDevice(string deviceId, out BaseDevice device)
    {
        return devices.TryGetValue(deviceId, out device);
    }
}
```

普通设备读取规则：

- 非 AGV 设备在建模或场景搭建阶段已放到某个父物体下。
- 程序运行时从这个父物体下递归读取所有 `BaseDevice` 子类。
- `DeviceId` 必须稳定且唯一；重复或为空应打印错误，避免后续消息找错设备。
- 管理器只做注册、查找、广播通用显示设置，不把具体设备动作写进管理器。

AGV 管理器负责运行时创建、查找、更新和销毁 AGV：

```csharp
// 中文注释：AGV 根据实时数据动态生成，并按车型选择不同 prefab。
public sealed class AgvManager : MonoBehaviour
{
    public Transform AgvRoot;
    public List<AgvPrefabBinding> Prefabs;
    private readonly Dictionary<string, BaseAgv> agvs = new Dictionary<string, BaseAgv>();

    public BaseAgv GetOrCreateAgv(AgvRuntimeState state)
    {
        if (agvs.TryGetValue(state.AgvId, out var agv))
        {
            return agv;
        }

        var prefab = FindPrefab(state.AgvType, state.AgvCategory);
        var instance = Instantiate(prefab, AgvRoot);
        var created = instance.GetComponent<BaseAgv>();
        Register(created, state);
        return created;
    }
}
```

AGV 生成规则：

- 首次收到某个 `AgvId` 的数据时，根据 `AgvType`、`AgvCategory` 或项目约定字段选择 prefab 动态生成。
- 已存在 AGV 时只更新状态和位置，不重复生成。
- 如果数据里携带地图 ID，例如 `mapCode`，应转换为内部 `MapId` 并写入 AGV 状态。
- AGV 离线后是否隐藏、置灰或保留最后位置，由项目策略决定；不要直接删除导致历史状态丢失，除非后端明确下发移除事件。

## 职责边界

负责：

- 根据设备状态播放动画或补间。
- 收到初始化数据时，直接把可移动设备定位到对应位置和角度。
- 根据精确 XYZ 坐标、速度、路径点驱动 Transform。
- 后续实时数据控制设备平滑移动和旋转。
- 对堆垛机、AGV、桁架、RGV 等可移动设备，按各轴坐标或物体坐标控制移动。
- 在当前设备脚本或当前设备子类中处理本设备的取放货动作表现。
- 为所有设备提供载货点，用于控制货物在当前设备上的显示位置。
- 根据任务号或托盘条码协调货物在设备间流转，AGV 除外。
- 标注起始站台、终点站台或起终点复合站台，用于限制运行时货物生成和删除入口。
- 识别未扫码前的临时任务号，并按固定路线特殊处理。
- 把离散状态转换为连续表现，例如插值移动。
- 暴露动作完成事件给协调层。

不负责：

- 不直接订阅 MQTT topic。
- 不直接查询 HTTP 接口。
- 不直接创建或删除库存货物。
- 不决定在途货物的真实归属；只提供设备动作和必要的货物挂载点。
- 不决定任务是否完成，只表现后端给出的状态。
- 不新增一个集中管理所有 AGV 或所有设备取放货动作的大脚本；跨设备协调只做轻量事件分发，具体动作仍回到设备自身。

## 设备取放货脚本边界

取放货动作应尽量由当前设备自己的脚本处理。这样新增车型、设备类型或取放货细节时，只改对应设备，不会把所有设备动作耦合到一个中心脚本里。

推荐：

```text
AGV 数据到达 -> AgvManager 找到对应 AGV -> 当前 AGV 子类处理本车取放货表现
堆垛机数据到达 -> NormalDeviceManager 找到堆垛机 -> 当前堆垛机脚本处理货叉取放货表现
```

不推荐：

```text
所有 AGV 数据 -> GlobalAgvCargoActionManager -> 统一判断所有车型的取放货和流转
```

规则：

- 当前设备的移动、旋转、货叉伸缩、夹爪开合、载货点切换等动作，优先放在当前设备脚本或当前设备子类中。
- 设备总管理器只负责注册、查找、创建和分发，不写具体取放货动作细节。
- 库存模块只负责货物对象、模型、归属和挂载关系，不写设备机械动作细节。
- 如果确实存在跨设备流转协调，可加轻量协调层发布事件，但协调层只表达“哪个货物从哪个设备到哪个设备”，不控制每个设备怎么完成取放货动作。
- 多种 AGV 取放货差异应通过 `BaseAgv` 子类、策略对象或车型配置处理，不通过一个中心脚本写满车型判断。

## 起终点站台与初始化货物生成

运行过程中，只有被明确标注为起始站台或终点站台的设备，才允许触发货物生成或删除。其他设备即使有载货点，也只负责承载、移动、交接或显示已有货物，不应自行生成或删除货物。

推荐用站台角色标记设备：

```csharp
// 中文注释：站台货物角色用于限制运行时货物生成和删除入口。
public enum StationCargoRole
{
    None,           // 普通站台，不生成也不删除货物
    Source,         // 起始站台，可触发生成货物
    Target,         // 终点站台，可触发删除货物
    SourceAndTarget // 既可作为起点，也可作为终点
}
```

规则：

- `Source` 或 `SourceAndTarget` 站台可以在运行时触发货物生成，例如入口扫码、入库申请、人工上料点。
- `Target` 或 `SourceAndTarget` 站台可以在运行时触发货物删除，例如出库口、人工下料点、离开数字孪生场景的末端。
- 输送线中间段、堆垛机货叉、提升机轿厢、RGV 台面、AGV 车体、桁架夹具等不应直接生成或删除货物，只能通过载货点接收、挂载或移交已有货物。
- 站台设备只表达“此处允许生成/删除”的意图或事件，实际货物对象的创建、销毁、索引维护和去重由库存模块处理。
- 站台角色应通过 Inspector、配置或设备元数据明确设置，不要仅根据设备名称猜测。

初始化阶段是例外。程序启动后，后端会把所有字段的上一条数据推送出来作为初始化快照。Unity 收到初始化数据时，应根据当前载货状态、托盘条码、任务号、库位或站台状态直接生成已有货物，用于还原现场当前状态。

初始化生成和运行时生成必须区分：

- 初始化生成：还原启动时现场已有货物，可发生在库位、输送线、站台、提升机、堆垛机、RGV、AGV 等已有载货点。
- 运行时生成：只允许从起始站台触发。
- 运行时删除：只允许从终点站台触发。
- 初始化快照重复到达时，应按 `cargoId`、`ContainerNumber`、`TaskNo` 或项目约定键去重，不要重复生成同一货物。
- 初始化完成后，应标记系统进入实时阶段，后续货物增删必须遵守起终点站台规则。

推荐流程：

```text
初始化快照 -> 解析所有设备上一条状态 -> 库存模块按载货状态重建货物 -> 标记初始化完成
实时数据 -> 当前设备脚本处理动作 -> 只有起始/终点站台触发货物生成或删除事件 -> 库存模块执行增删
```

站台触发事件可以保持很薄：

```csharp
// 中文注释：站台只发送货物生成意图，实际创建由库存模块完成。
public sealed class StationCargoCreateRequested
{
    public string StationId;       // 站台编号
    public string TaskNo;          // 当前任务号
    public string ContainerNumber; // 托盘或容器编号
    public int MaterialType;       // 货物类型
    public int ContainerType;      // 容器类型
}
```

## 通用输入模型

设备驱动建议消费稳定模型：

```csharp
// 中文注释：设备驱动只消费已整理好的运行状态，不解析外部原始消息。
public sealed class DeviceRuntimeState
{
    public string EntityId;
    public string DeviceType;
    public string WorkState;
    public Vector3? TargetPosition;
    public Quaternion? TargetRotation;
    public Vector3? AxisPosition;
    public float Speed;
    public bool HasCargo;
    public string TaskId;
    public string PalletCode;
    public bool IsTemporaryTask;
}
```

## 协议读取字段映射

后端会处理 PLC、WCS、WMS、现场设备通信和数据保存，Unity 不需要管理 PLC 读写区、握手、清除指令或任务下发流程。设备协议文档只用于理解后端推送到 Unity 的字段含义。进入 Unity 后，应先把协议字段转换为内部状态，再交给设备子类、库存、告警和 UI 使用。

普通设备读取字段建议按用途分层：

```csharp
// 中文注释：普通设备读取 DTO 只承接后端推送字段，不把 PLC 通信逻辑带进 Unity。
public sealed class NormalDeviceReadDto
{
    public string DeviceNo;
    public int Alarm;
    public int Load;
    public int State;
    public int Source;
    public int Target;
    public bool IsForward;
    public bool IsReverse;
    public double? PosXaxis;
    public double? PosYaxis;
    public double? PosZaxis;
    public double? PosRaxis;
    public string TaskNo;
    public int MaterialType;
    public int ContainerType;
    public string ContainerNumber;
}
```

字段归类建议：

- `DeviceNo`：设备唯一编号，映射到内部 `DeviceId` 或 `EntityId`。
- `Alarm`：告警状态或告警码，转换为 `IsAlarm` 和告警事件；具体标红、闪烁、报警内容由告警模块处理。
- `Load`：载货状态，表示设备载货点是否有货；堆垛机协议中类似字段可能叫 `ForkLift`。
- `State`：设备运行状态码，先保留原始码，再由设备类型自己的状态映射器转换成内部 `WorkState`。
- `Source`、`Target`：源地址和目标地址，用于任务、路径或货物流转表现；不要在 Unity 端反推业务调度结果。
- `IsForward`、`IsReverse`：方向类字段，通常只给输送线、RGV 或需要方向表现的设备使用，不放进所有设备基类。
- `PosXaxis`、`PosYaxis`、`PosZaxis`、`PosRaxis`：轴坐标或旋转坐标，必须经过坐标原点、单位比例和轴向映射后再驱动模型。
- `TaskNo`：任务号，用于普通设备货物流转关联；临时任务号按未扫码临时任务规则处理。
- `MaterialType`：货物类型，用于货物模型、材质或信息展示，归库存模块消费。
- `ContainerType`：托盘、料框、容器类型，用于选择容器表现。
- `ContainerNumber`：托盘条码或容器编号，用于库存货物、在途货物和设备载货点绑定。

不要把所有字段都塞进 `BaseDevice`。基类只保留稳定公共属性；协议字段进入 DTO，转换后的共性结果进入 `DeviceRuntimeState`，设备专有结果进入各自状态模型。

推荐转换关系：

```text
后端推送字段 -> NormalDeviceReadDto -> DeviceRuntimeState + 设备专有状态 -> 设备子类表现
```

其中：

- 输送线通常关注 `Load`、`State`、`Source`、`Target`、`IsForward`、`IsReverse`、`TaskNo`、`ContainerNumber`。
- 提升机通常关注 `Load`、`State`、`Source`、`Target`、垂直位置、当前层和 `ContainerNumber`。
- RGV 通常关注 `State`、`Source`、`Target`、当前位置、水平坐标、水平速度和载货状态。
- 堆垛机通常关注 `State`、`Alarm`、货叉载货、区域/列/层、X/Y/Z 轴坐标、水平/垂直/货叉速度和托盘编号。
- 桁架/垳架通常关注 `DeviceNo`、`Alarm`、`Load`、`State`、`Source`、`Target`、`PosXaxis`、`PosYaxis`、`PosZaxis`、`PosRaxis`、`TaskNo`、`MaterialType`、`ContainerType`、`ContainerNumber`。

`State` 的数字含义应按设备类型分别映射。不同设备即使都有 `State` 字段，也不要假设同一个数值在所有设备上含义完全一致。

```csharp
// 中文注释：状态码由设备类型自己的映射器解释，避免把协议枚举写死在基类里。
public interface IDeviceStateCodeMapper
{
    string ToWorkState(string deviceType, int stateCode);
}
```

桁架/垳架可以使用专有状态承载轴和货物信息：

```csharp
// 中文注释：桁架/垳架专有状态保留四轴坐标和当前搬运对象。
public sealed class GantryRuntimeState
{
    public string DeviceId;
    public bool IsAlarm;
    public bool HasCargo;
    public string WorkState;
    public int Source;
    public int Target;
    public Vector4 AxisPosition;
    public string TaskNo;
    public int MaterialType;
    public int ContainerType;
    public string ContainerNumber;
}
```

## 驱动策略

- 收到初始化数据时，直接设置设备位置、角度和各轴初始值，不做平滑补间，避免启动后从默认位置飞到真实位置。
- 初始化完成后的实时数据，应使用平滑移动和旋转表现设备变化。
- 坐标变化频繁的设备使用插值或补间，避免位置跳变。
- 多轴设备应拆分轴控制，例如横移、升降、伸缩分别处理。
- 复杂动作使用动作队列或小型状态机，不要写成长串 `if/else`。
- Animator 通常只用于机械手这类复杂姿态设备；其余设备优先通过位置、旋转、轴向移动和简单状态表现实现。
- 后端给最终状态，Unity 负责表现过程；不要让 Unity 自行推导业务结果。

## 初始化定位与后续平滑

可移动设备在收到初始化数据时，应直接定位到数据对应的位置和角度。初始化数据用于校准场景，不应用补间从 prefab 默认位置移动到真实位置。

初始化阶段：

- 直接设置设备根节点位置和角度。
- 直接设置堆垛机、桁架等多轴设备的各轴位置。
- 直接设置 AGV、RGV 等整体移动设备的位置、角度和载货状态。
- 记录设备已经完成初始化，后续数据进入平滑驱动流程。

后续实时阶段：

- 根据实时 XYZ 坐标平滑移动。
- 根据实时角度、朝向或路径方向平滑旋转。
- 多轴设备分别平滑移动各轴。
- 数据跳变超过阈值时，可按项目策略直接校准，避免长时间追赶错误位置。

示例：

```csharp
// 中文注释：初始化数据直接校准位置，后续实时数据再平滑移动。
public void ApplyDevicePose(DeviceRuntimeState state, bool isInitialize)
{
    if (isInitialize)
    {
        SetPoseImmediately(state);
        return;
    }

    MoveAndRotateSmoothly(state);
}
```

## 精确坐标驱动

堆垛机、AGV、桁架、RGV 等可移动设备通常会从数据中拿到精确 XYZ 坐标。设备动作模块应以数据坐标为准，驱动对应轴或设备主体移动。

实现建议：

- AGV、RGV 等整体移动设备，优先用目标 XYZ 坐标驱动设备根节点位置，并按朝向、速度或路径数据平滑过渡。
- 堆垛机应拆分横移、升降、货叉伸缩等轴；每个轴由对应坐标或位置字段驱动。
- 桁架应按 X/Y/Z 三轴分别驱动，不要把多轴运动写成不可拆的整体补间。
- 数据坐标不能直接赋给 Unity 的 `Transform.position`，必须通过设备或场景原点换算为 Unity 坐标。
- 坐标单位、Unity 缩放比例、数据原点、Unity 原点、原点偏移和轴方向应集中配置，不要散落在设备脚本中。
- 数据跳变时要按项目规则插值、限速或直接校准，避免视觉表现和真实数据长期偏离。

## 坐标原点换算

堆垛机、AGV、桁架、RGV 等设备的 XYZ 坐标通常来自现场坐标系或上位机坐标系。Unity 场景也有自己的原点、比例和轴方向，因此必须先做坐标映射，再驱动模型。

推荐把每类设备或每个区域的坐标映射配置化：

```csharp
// 中文注释：坐标映射配置描述现场数据坐标如何转换为 Unity 坐标。
public sealed class DeviceCoordinateMapping
{
    public string DeviceId;
    public Vector3 SourceOrigin;
    public Vector3 UnityOrigin;
    public Vector3 SourceXAxisInUnity;
    public Vector3 SourceYAxisInUnity;
    public Vector3 SourceZAxisInUnity;
    public float UnitScale;
}
```

换算原则：

- `SourceOrigin` 表示数据坐标系中的原点，可能是设备零点、库区原点、轨道起点或现场约定点。
- `UnityOrigin` 表示该数据原点在 Unity 场景中的位置。
- 数据坐标先减去 `SourceOrigin`，得到相对位移。
- 相对位移按 `UnitScale` 转换单位，例如毫米转米、厘米转米或现场单位转 Unity 单位。
- X/Y/Z 三个数据轴要分别映射到 Unity 中的方向，不能默认现场 Z 轴一定等于 Unity Z 轴。
- 堆垛机、桁架这类多轴设备可以每个轴单独映射，也可以共用一个设备坐标映射。
- 坐标映射应支持项目配置覆盖，方便现场原点调整后无需修改设备动作代码。

示例：

```csharp
// 中文注释：先按数据原点计算相对坐标，再映射到 Unity 原点和 Unity 方向轴。
public Vector3 ToUnityPosition(DeviceCoordinateMapping mapping, Vector3 sourcePosition)
{
    var relative = (sourcePosition - mapping.SourceOrigin) * mapping.UnitScale;

    return mapping.UnityOrigin
        + mapping.SourceXAxisInUnity.normalized * relative.x
        + mapping.SourceYAxisInUnity.normalized * relative.y
        + mapping.SourceZAxisInUnity.normalized * relative.z;
}
```

不要在每个设备脚本里写类似 `new Vector3(x, y, z)` 的直接映射。应该统一通过坐标映射服务处理，确保 AGV、堆垛机、桁架、RGV 的原点、比例和轴方向一致。

示例：

```csharp
// 中文注释：设备坐标转换集中处理，避免每个设备脚本重复写比例、原点偏移和轴向映射。
public interface IDeviceCoordinateMapper
{
    Vector3 ToUnityPosition(string deviceId, Vector3 sourcePosition);
}
```

## AGV 多地图坐标

AGV 项目可能同时存在多个地图，例如外部数据中的 `mapCode` 可出现 `AA`、`BB`、`CC`、`DD` 等。每个地图都可能有自己的数据原点、Unity 原点、单位比例和轴方向，不能用一个全局原点套所有 AGV。

推荐为 AGV 单独维护地图坐标配置：

```csharp
// 中文注释：每个 AGV 地图都维护独立的数据原点和 Unity 原点。
public sealed class AgvMapCoordinateMapping
{
    public string MapId;
    public Vector3 SourceOrigin;
    public Vector3 UnityOrigin;
    public Vector3 SourceXAxisInUnity;
    public Vector3 SourceYAxisInUnity;
    public Vector3 SourceZAxisInUnity;
    public float UnitScale;
    public float DirectionOffset;
}
```

AGV 坐标换算规则：

- 先根据 AGV 状态中的 `MapId` 找到对应地图配置。
- 再把 `cooX`、`cooY` 等源坐标转换为 Unity 坐标。
- AGV 的朝向应结合 `direction` 和当前地图的 `DirectionOffset` 转换。
- 如果 AGV 数据缺少地图 ID，应按项目默认地图处理或记录错误，不要默默套用上一次地图。
- 如果 AGV 从一个地图切换到另一个地图，应更新 `MapId`、重新使用新地图原点换算位置，并按项目策略决定是否直接校准位置，避免跨地图平滑飞行。

地图切换与楼层切换不是同一件事：

- AGV 地图切换用于解释 AGV 坐标属于哪个地图原点。
- 楼层切换用于控制当前看哪一层、对象显示隐藏和相机视角。
- 一个楼层可以包含多个 AGV 地图，一个 AGV 地图也可能只对应楼层的一部分区域。
- 地图不可见时，AGV 仍应继续接收状态并更新内部位置；显示策略交给楼层或地图可见性控制器处理。

## Animator 使用范围

Animator 通常只用于机械手。机械手需要根据当前货物的垛型以及下一抓动作驱动动画，例如不同垛型、抓取姿态、放置姿态、夹爪开合和关节动作。

机械手动画建议输入：

- 当前货物垛型。
- 下一抓动作。
- 抓取点和放置点。
- 夹爪状态。
- 动作阶段，例如准备、抓取、移动、放置、复位。

其余设备，例如输送线、提升机、堆垛机、AGV、RGV、桁架、拆码盘机、分拣线，默认通过 Transform 移动、旋转、轴向位移、材质状态或简单部件显示实现。除非项目已有成熟 Animator 方案，否则不要为了普通位移设备引入复杂动画状态机。

## 载货点

所有设备都应有载货点，用于控制货物在当前设备上的显示位置。载货点是设备动作模块和库存模块之间的稳定交接点。

规则：

- 会移动的设备，载货点应放在当前设备的某个子物体下，跟随设备移动和旋转，例如 AGV 车体、RGV 台面、堆垛机货叉、桁架夹具。
- 不会移动的设备，载货点统一放在一个固定父物体下，便于库存模块挂载和查找。
- 设备模块负责提供载货点 Transform，不直接创建、删除或决定货物真实归属。
- 库存模块负责把货物对象挂到载货点、从载货点解绑或切换到下一个库位/设备。
- 一个设备有多个载货位置时，应通过载货点 ID 区分，例如入口位、出口位、左货叉、右货叉、缓存位。

推荐接口：

```csharp
// 中文注释：所有设备通过统一接口暴露载货点，库存模块通过载货点挂载货物。
public interface IDeviceCargoMountProvider
{
    string DeviceId { get; }
    Transform GetCargoMountPoint(string mountPointId);
}
```

## 货物流转标识

货物的流转基本由任务号或托盘条码控制，AGV 除外。设备动作模块可以根据任务号或托盘条码判断货物应处于哪个设备段、挂载点或过渡状态，但货物对象的真实归属仍由库存模块维护。

推荐规则：

- 输送线、堆垛机、提升机、RGV、桁架、机械手、拆码盘机、分拣线等货物流转，优先使用任务号 `taskId` 或托盘条码 `palletCode` 关联。
- AGV 由于通常有独立坐标、任务模型或车载状态，不强制套用普通设备的任务号/托盘条码流转逻辑。
- 同一任务号或托盘条码跨设备流转时，应通过统一协调层或库存模块更新货物挂载关系。
- 缺失任务号或托盘条码时，不要把货物混入正常任务链路。

## 未扫码临时任务

有些货物在未扫码前没有正确任务号或托盘条码，上位机可能会给一个约定的临时任务号，例如 `1000`，也可能是项目提前协调的其他值。这类任务必须单独处理，不与正常任务混用逻辑。

处理原则：

- 临时任务号列表应配置化，例如 `1000`、`TEMP` 或项目约定值，不要硬编码在多个脚本中。
- 临时任务通常按固定路线移动，直到扫码后产生正确任务号或托盘条码。
- 扫码完成后，应从临时任务链路切换到正常任务链路，并建立正确货物关联。
- 临时任务不要进入正常任务统计、正常任务查找或正常任务完成判断，除非项目明确要求。
- 如果临时任务长时间未转为正常任务，应允许告警、日志或 UI 提示。

示例：

```csharp
// 中文注释：临时任务号通过配置判断，避免和正常任务逻辑混在一起。
public interface ITemporaryTaskRule
{
    bool IsTemporaryTask(string taskId, string palletCode);
}
```

## 设备动作与库存

取放货时不要让设备脚本直接生成或删除库存对象。推荐由动作完成事件通知库存模块：

```text
堆垛机货叉到达库位 -> 动作完成事件 -> 库存模块更新货物表现
```

如果货物在输送线、AGV、堆垛机货叉、提升机等设备上处于在途状态，设备模块负责运动和挂载点，库存模块负责货物对象、模型和归属状态。

普通设备上的货物流转通常通过任务号或托盘条码关联；AGV 货物可按 AGV 独立状态、车载货物字段或项目协议处理。临时任务号只用于未扫码前的固定路线表现，不能污染正常库存或任务关联。

## 验证重点

- 设备状态切换不会卡在旧动画。
- 初始化数据能直接定位设备位置、角度和各轴状态，不出现从默认位置飞行的过程。
- 初始化后的实时数据能控制设备平滑移动和旋转。
- 堆垛机、AGV、桁架、RGV 等设备能按精确 XYZ 坐标或各轴坐标移动。
- 设备数据坐标能通过数据原点、Unity 原点、单位比例和轴向映射换算成 Unity 坐标。
- Animator 默认只用于机械手，并能根据货物垛型和下一抓动作驱动。
- 所有设备都能提供载货点；移动设备载货点跟随设备子物体，固定设备载货点位于统一父物体下。
- 普通设备能在启动时从场景父物体下读取并注册，重复或缺失 `DeviceId` 能报错。
- 起始站台、终点站台和普通站台的角色标记清楚。
- 初始化快照能根据所有设备上一条数据直接还原已有货物，且不会重复生成。
- 初始化完成后，只有起始站台能触发货物生成，只有终点站台能触发货物删除。
- 输送线中间段、堆垛机、提升机、RGV、AGV、桁架等非起终点对象不会在运行时误生成或误删除货物。
- AGV 能根据数据动态生成，并按 `AgvId` 查找和复用已有实例。
- AGV 多车型能通过 `AgvType`、`AgvCategory` 或项目约定字段选择不同 prefab。
- AGV 坐标能按 `MapId` 使用对应地图原点、比例、轴方向和朝向偏移。
- AGV 地图切换不会使用旧地图原点，也不会无意触发楼层业务状态变化。
- 普通设备货物流转能按任务号或托盘条码关联，AGV 例外逻辑清楚。
- 临时任务号能按固定路线处理，并在扫码后切换到正常任务逻辑。
- 临时任务不会混入正常任务统计或正常任务流转逻辑。
- 高频位置更新时运动平滑。
- 后端发来异常状态时设备能停下或进入异常表现。
- 多设备同时运行时不会互相覆盖状态。
