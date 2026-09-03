# UI 界面控制

适用于常驻面板、弹窗、界面调用、数据显示、界面关闭和用户交互。UI 模块负责界面生命周期和显示绑定，不负责底层通信和设备动作细节。

## UI 类型

- 常驻面板：连接状态、总览指标、报警摘要、任务摘要。
- 弹窗：设备详情、货物详情、报警详情、确认框。
- 列表面板：库存列表、任务列表、报警历史、设备列表。
- 操作面板：视角切换、定位设备、筛选、搜索。

## 职责边界

负责：

- 打开、关闭、切换、刷新界面。
- 把 DTO、状态或 ViewModel 显示到 Text、TMP、列表、图表。
- 响应按钮、输入框、下拉框和列表点击。
- 处理加载中、空数据、错误、无权限等状态。

不负责：

- 不直接解析原始 JSON。
- 不直接订阅 MQTT topic。
- 不直接写设备 Transform 移动逻辑。
- 不直接生成或删除库存对象。

## 推荐模式

UI 层通过服务或状态仓库取数据：

```csharp
// 中文注释：UI 控制器只处理界面生命周期和数据绑定。
public sealed class DeviceDetailPanel : MonoBehaviour
{
    public async void Open(string deviceId)
    {
        // 中文注释：实际项目中应处理取消、异常和加载状态。
        var detail = await devicePanelService.GetDeviceDetailAsync(deviceId);
        Bind(detail);
    }
}
```

常驻面板可以订阅内部状态变化，但要在销毁或关闭时取消订阅。

## UI 调用其他模块

UI 可以发起高层意图，不写具体实现：

```csharp
// 中文注释：UI 只发出聚焦请求，相机如何移动由相机模块决定。
cameraController.Focus("AGV-001");
```

```csharp
// 中文注释：UI 只请求展示设备，设备查找和高亮由对应服务处理。
twinSelectionService.SelectEntity("CV-001");
```

## 验证重点

- 面板重复打开关闭不会重复注册事件。
- 数据未返回时界面不报空引用。
- 长文本不会溢出关键 UI 区域。
- 弹窗关闭后异步回调不会刷新已销毁对象。
