# 库存管理

适用于 Unity 场景中的库存货物生成、库位状态、货物信息展示、取放货增减和库存可视化。库存模块表现后端给出的库存事实，不在 Unity 端决定真实库存。

## 职责边界

负责：

- 根据快照生成货物模型或占位表现。
- 区分平库和立库的库存表现方式。
- 根据货物类型、物料、批次、容器或接口字段选择不同货物模型。
- 根据接口返回的货架原点、长宽高、排列、层、深等结构信息，一次性计算并生成库存货物。
- 生成量太大时，支持分批、分帧或异步队列生成，避免 Unity 主线程明显卡顿。
- 根据库存变化事件增加、移除、移动或更新货物。
- 根据初始化快照直接生成启动时现场已有货物。
- 响应起始站台的货物生成请求和终点站台的货物删除请求。
- 维护货架、货物、库位、托盘、容器之间的表现绑定。
- 处理货架筛选，例如点击 1 货架时只显示 1 货架及其货物。
- 管理鼠标移入货物时的高亮效果和移出后的恢复。
- 点击货物或库位时提供展示数据。
- 管理在途货物的货物对象归属和状态，例如货物在输送线、AGV、堆垛机货叉或提升机中。
- 处理取货、放货、移库后的视觉变化。

不负责：

- 不直接控制 AGV、堆垛机、输送线运动。
- 不直接修改后端库存。
- 不把设备运动路径、速度、动画状态写进库存模块。
- 不自行判断业务库存是否成功，只表现后端状态或确认事件。
- 不让普通设备、输送线中间段或移动设备绕过起终点站台规则随意生成或删除货物。

## 核心对象

- 库存类型：平库、立库。
- 货架或库区：`rackId`、原点、长宽高、排列数、层数、深度数、方向轴、间距。
- 库位：`slotId`、占用状态、区域、排列、层、深。
- 货物：`cargoId`、物料、批次、数量、状态、模型类型。
- 托盘或容器：`carrierId`、绑定货物、当前位置。
- 在途货物：绑定货物 ID、承载设备 ID、视觉挂载点、业务状态。
- 货物流转标识：任务号、托盘条码、临时任务标记。
- 库存事件：入库、出库、移库、盘点、锁定、释放。

## 推荐输入模型

```csharp
// 中文注释：库存表现以库位和货物绑定为核心，不关心搬运设备如何运动。
public sealed class InventoryItemState
{
    public string WarehouseType;
    public string RackId;
    public string SlotId;
    public string CargoId;
    public string MaterialCode;
    public string ModelType;
    public string TaskId;
    public string PalletCode;
    public bool IsTemporaryTask;
    public int Quantity;
    public string Status;
}
```

## 平库与立库

库位通常分为平库和立库，两者的货物绑定方式不同。

平库：

- 货物通常绑定到区域、地面点位、缓存位、暂存位或输送线边的平面库位。
- 坐标可以由区域原点、行列、点位索引或接口返回的直接坐标计算。
- 不一定存在货架对象，但仍应维护 `slotId -> cargoId` 的绑定。

立库：

- 货物放在货架上，需要维护 `rackId -> slotId -> cargoId` 的绑定关系。
- 货架负责空间结构，库位负责具体排列、层、深，货物负责库存对象表现。
- 生成货物时先找到货架，再根据库位的排列、层、深计算货物位置。
- 货架隐藏、筛选或楼层切换时，货架下的库位和货物应一起响应可见性规则。

推荐绑定关系：

```text
立库：RackView(rackId) -> SlotView(slotId) -> CargoView(cargoId)
平库：AreaView(areaId) 或 SlotView(slotId) -> CargoView(cargoId)
```

不要只把货物挂在一个全局根节点下，否则做货架筛选、楼层切换和库位定位时会变得困难。

## 货物模型选择

货物模型可能有多种，不要在生成逻辑中写死单一 prefab。推荐通过配置或映射表选择：

```csharp
// 中文注释：货物模型配置负责把业务字段映射到具体 Prefab。
public sealed class CargoModelConfig
{
    public string ModelType;
    public GameObject Prefab;
}
```

选择优先级可以按项目约定：

- 接口返回明确 `modelType` 时，优先使用该模型。
- 没有模型类型时，可根据物料编码、货物类型、容器类型或批次规则映射。
- 找不到匹配模型时，使用默认货物模型或占位模型，并记录日志。

## 按排列、层、深生成货物

库存初始化时，接口可能返回货架原点、长宽高、排列、层、深等结构信息。库存模块应以货架或库区原点为基准，根据货架尺寸、排列方向、层高、深度间距计算库位坐标和货物坐标，再一次性生成当前库存表现。

推荐流程：

```text
接口返回库存/库位结构
-> 解析货架原点、长宽高、排列、层、深和库位 ID
-> 通过 rackId 找到或创建对应货架对象
-> 根据原点、方向轴、货架尺寸和间距计算每个库位的世界坐标或局部坐标
-> 根据库存数据选择货物模型
-> 生成货物并绑定 rackId、slotId、cargoId、floorId
-> 建立 rackId/slotId/cargoId 到场景对象的索引
```

## 货架原点与尺寸计算

货架、库区或库位组通常应有一个稳定原点。生成货物时不要手工摆放每个货物，而是通过结构参数计算位置。

推荐模型：

```csharp
// 中文注释：货架布局配置描述从原点推导库位坐标所需的空间参数。
public sealed class RackLayoutConfig
{
    public string RackId;
    public string WarehouseType;
    public Vector3 Origin;
    public Vector3 ColumnDirection;
    public Vector3 LayerDirection;
    public Vector3 DepthDirection;
    public int ColumnCount;
    public int LayerCount;
    public int DepthCount;
    public float RackLength;
    public float RackWidth;
    public float RackHeight;
    public Vector3 CargoOffset;
}
```

坐标计算原则：

- 原点可以是货架左下前角、中心点或项目约定的基准点，但必须在配置中保持一致。
- 排列、层、深应转换为从 0 开始的索引后再计算，避免接口从 1 开始导致偏移错误。
- 如果接口提供货架长宽高，可以由尺寸除以排列数、层数、深度数得到间距。
- 如果接口直接提供列间距、层高、深度间距，优先使用接口或配置中的明确间距。
- `ColumnDirection`、`LayerDirection`、`DepthDirection` 应使用归一化方向，避免货架旋转后坐标计算错误。
- 货物模型中心点不一定等于库位中心，可以通过 `CargoOffset` 做统一偏移。

示例：

```csharp
// 中文注释：根据货架原点、排列、层、深计算货物生成位置。
private Vector3 CalculateCargoPosition(RackLayoutConfig layout, int column, int layer, int depth)
{
    var columnIndex = column - 1;
    var layerIndex = layer - 1;
    var depthIndex = depth - 1;

    var columnSpacing = layout.ColumnCount <= 1 ? 0f : layout.RackLength / layout.ColumnCount;
    var layerSpacing = layout.LayerCount <= 1 ? 0f : layout.RackHeight / layout.LayerCount;
    var depthSpacing = layout.DepthCount <= 1 ? 0f : layout.RackWidth / layout.DepthCount;

    return layout.Origin
        + layout.ColumnDirection.normalized * columnSpacing * columnIndex
        + layout.LayerDirection.normalized * layerSpacing * layerIndex
        + layout.DepthDirection.normalized * depthSpacing * depthIndex
        + layout.CargoOffset;
}
```

实际项目中要按货架原点定义调整是否需要加半个间距：如果原点代表库位中心，则不加；如果原点代表货架边角，通常需要把货物移动到库位中心。

## 货架绑定与筛选

立库货物必须和货架建立明确绑定。点击或选择某个货架时，应只显示该货架及其货物；其他货架和货物按筛选规则隐藏。这个筛选属于库存可见性过滤，不改变库存真实状态。

推荐接口：

```csharp
// 中文注释：库存可见性服务负责按货架筛选显示，不改变库存真实数据。
public interface IInventoryVisibilityService
{
    void ShowAllRacks();
    void ShowOnlyRack(string rackId);
}
```

筛选规则：

- 点击 `1` 货架时，只显示 `rackId = "1"` 的货架、库位和货物。
- 其他货架、库位和货物隐藏，但库存状态继续保留。
- 当前有楼层筛选时，货架筛选应叠加楼层筛选，不能把非当前楼层货架强行显示出来。
- 当前有告警、选中或搜索结果时，要定义显示优先级；通常“选中的货架及其货物可见”优先于普通库存显示。
- 退出货架筛选时恢复到当前楼层或当前全局筛选规则，而不是无条件显示全部对象。

UI 只表达筛选意图：

```csharp
// 中文注释：UI 只请求筛选货架，具体显示隐藏由库存模块处理。
inventoryVisibilityService.ShowOnlyRack("1");
```

如果生成数量较大，不要在一帧内实例化全部高面数模型。可以：

- 每帧生成固定数量。
- 使用协程分批生成。
- 使用对象池复用货物对象。
- 使用简化模型、合批或实例化渲染。
- 先生成占位表现，再逐步替换为完整模型。

示例：

```csharp
// 中文注释：大量货物分批生成，避免单帧实例化过多对象造成卡顿。
private IEnumerator CreateCargoInBatches(IReadOnlyList<InventoryItemState> items, int batchSize)
{
    for (var index = 0; index < items.Count; index++)
    {
        CreateCargo(items[index]);

        if ((index + 1) % batchSize == 0)
        {
            yield return null;
        }
    }
}
```

## 与设备动作的关系

搬运动作和库存变化要解耦：

```text
设备动作表现 -> 到达取货点 -> 动作完成事件
后端确认库存变化 -> 库存事件 -> Unity 更新货物表现
```

如果项目为了视觉连贯，需要在动作过程中临时把货物挂到 AGV 或货叉上，应区分“视觉临时挂载”和“真实库存状态”。真实库存以后端消息为准。

## 初始化与站台增删

程序启动后，后端会推送所有字段的上一条数据作为初始化快照。库存模块应根据初始化数据直接生成当前现场已有货物，并建立货物、库位、设备载货点、任务号和托盘条码的索引。

初始化规则：

- 初始化快照用于还原当前现场状态，不受起终点站台限制。
- 初始化时可根据设备 `Load`、`ContainerNumber`、`TaskNo`、库位占用状态或后端库存快照生成货物。
- 初始化生成必须去重，优先使用稳定的 `cargoId`、`ContainerNumber`、`TaskNo` 或项目约定组合键。
- 初始化完成后，系统进入实时阶段，后续货物生成和删除必须遵守站台角色规则。

运行时规则：

- 起始站台触发货物生成，库存模块负责创建货物对象、选择模型、建立索引并挂载到站台载货点。
- 终点站台触发货物删除，库存模块负责解绑、销毁或回收货物对象，并清理索引。
- 非起始/终点站台的设备只允许移动、挂载、解绑或交接已有货物，不允许直接创建或删除货物。
- 如果运行时收到非起终点设备的新增或删除意图，应记录错误或按项目策略忽略，避免场景货物凭空出现或消失。

## 在途货物

在途货物应由库存模块管理货物对象和货物状态，但设备动作模块负责承载设备如何移动。换句话说：

```text
库存模块：这件货物是否存在、是什么模型、当前挂在哪个设备或库位上。
设备动作模块：AGV、输送线、堆垛机、提升机如何移动和播放动画。
```

推荐协作方式：

- 后端消息或库存事件确认货物进入在途状态。
- 库存模块把货物对象从库位解绑，并挂到设备提供的货物挂载点。
- 设备动作模块继续驱动设备移动；货物作为子对象或跟随对象同步移动。
- 到达目标位置并收到后端确认后，库存模块把货物绑定到新库位、输送线位置或目标容器。

设备脚本可以暴露挂载点，但不直接创建、删除或决定货物真实归属：

```csharp
// 中文注释：设备只提供货物挂载点，货物对象仍由库存模块管理。
public interface ICargoCarrier
{
    string CarrierId { get; }
    Transform CargoMountPoint { get; }
}
```

如果只是视觉演示，允许临时视觉挂载；但必须标注为临时表现，最终状态以后端库存事件或快照为准。

普通设备上的在途货物通常通过任务号或托盘条码关联；AGV 可按独立协议、车载状态或坐标数据处理。未扫码前的临时任务号应带有临时标记，不要和正常任务或正常托盘条码混用。

所有设备都应提供载货点。移动设备的载货点通常在设备子物体下，货物挂上后会跟随设备移动；固定设备的载货点应统一放在固定父物体下。库存模块只使用设备暴露的载货点挂载货物，不自行推导设备内部空间位置。

## 货物高亮

鼠标移入货物时应显示高亮效果，移出后恢复原始表现。高亮属于库存对象的交互表现，适合放在库存模块或库存交互子模块中。

实现注意：

- 高亮不要覆盖告警、选中、禁用等更高优先级表现。
- 修改材质或颜色前记录原始状态，移出后恢复。
- 大量货物场景中，优先使用轮廓、高亮层或共享材质方案，避免频繁实例化材质。
- 鼠标悬停只触发视觉反馈；点击货物展示详情仍由 UI/面板模块负责。

## 货物生成

- 货物生成应区分平库和立库；立库根据货架原点、长宽高、排列、层、深计算位置，并绑定 rackId、slotId、cargoId。
- 大批量库存不要为每个货物生成高面数模型。
- 远景可用合批、实例化、占位块或简化模型。
- 货物点击详情可以通过 HTTP 查询完整信息。
- 库位状态和货物对象应能从快照重建。

## 验证重点

- 初始快照能正确生成库存。
- 初始化快照能根据上一条设备数据生成已有在途货物，并避免重复生成。
- 初始化完成后，运行时货物生成只来自起始站台，货物删除只来自终点站台。
- 平库和立库能按不同绑定规则生成货物。
- 立库货物能绑定到正确货架和库位。
- 点击单个货架时，只显示该货架及其货物。
- 退出货架筛选时能恢复当前楼层或当前全局筛选状态。
- 多种货物模型能根据接口字段或配置正确选择。
- 能根据货架原点、长宽高、排列、层、深计算库位和货物位置。
- 原点为边角或中心点时，货物位置偏移符合项目约定。
- 生成量大时能分批生成，避免明显卡顿。
- 鼠标移入货物能高亮，移出后能恢复。
- 在途货物由库存模块管理对象归属，并能跟随承载设备表现。
- 重复库存事件不会重复生成同一货物。
- 出库或移库后旧库位能正确清空。
- 后端重发快照时能校正 Unity 本地表现。
