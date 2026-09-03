# 面板数据 HTTP 对接

适用于 Unity UI 面板通过 HTTP 查询详情、列表、历史、统计或配置类数据。HTTP 模块服务于请求型数据，不承担实时同步主链路。

## 职责边界

负责：

- 封装 GET、POST 等请求。
- 从 `StreamingAssets/UrlConfig.json` 读取接口配置。
- 通过中文接口名称在配置文件中检索接口地址。
- 处理鉴权头、超时、重试和错误码。
- 将接口结果转换为 UI 可用 DTO。
- 为面板提供加载中、成功、失败、空数据状态。
- 统一打印接口调用日志，成功使用 `Debug.Log`，失败使用 `Debug.LogError`。

不负责：

- 不用 HTTP 高频轮询设备位置，除非项目明确没有实时通道。
- 不直接控制设备动作。
- 不把接口返回的原始 JSON 直接散落到 UI 脚本里解析。
- 不在面板脚本里硬编码 IP、端口或 API 路径。

## 适合 HTTP 的数据

- 设备详情。
- 库存列表和货物详情。
- 历史报警。
- 任务详情和任务列表。
- 统计指标和报表。
- 系统配置、区域配置、设备元数据。

## 推荐模式

UI 面板不要直接拼 URL，也不要直接读取配置文件。优先封装服务类：

```csharp
// 中文注释：面板只调用服务方法，不关心具体 URL 和鉴权细节。
public interface IDevicePanelService
{
    Task<DeviceDetailDto> GetDeviceDetailAsync(string deviceId);
}
```

面板打开时加载数据，关闭时取消未完成请求。列表类面板要处理分页、刷新和空状态。

## 接口配置来源

HTTP 接口的 IP、端口和 API 路径来自外部配置文件，默认读取 Unity 项目的 `Assets/StreamingAssets/UrlConfig.json`。不要在代码里写死接口地址。

配置文件推荐结构：

```json
{
  "globalSocket": "http://192.168.2.83:5103",
  "lstUrls": [
    {
      "sceneName": "SLP",
      "interfaceName": "获取设备清单",
      "socket": "",
      "api": "/api/device-types/with-devices"
    }
  ]
}
```

查找规则：

- 优先用当前场景名 `sceneName` 和中文接口名 `interfaceName` 精确匹配。
- 如果项目只有一个场景或调用方已明确场景，可以只按中文接口名匹配，但要避免重名。
- 单个接口配置中的 `socket` 有值时，使用该 `socket` 作为接口 IP 和端口。
- 单个接口配置中的 `socket` 为空时，回退使用 `globalSocket`。
- 最终请求地址由 `socket + api` 拼接得到，拼接时处理多余或缺失的 `/`。
- 找不到接口名、`api` 为空、`socket` 和 `globalSocket` 都为空时，应给出明确错误，不要静默请求空地址。

配置中的 `interfaceName` 面向业务和面板调用，可以使用中文，例如：

```text
立库库存初始化
获取设备清单
今日设备开动率
获取单个库位库存详情
```

## 配置读取与查询

建议单独封装配置仓库或解析器，HTTP Service 通过中文名称取 URL：

```csharp
// 中文注释：接口配置仓库负责读取 UrlConfig，并按场景名和中文接口名查找接口地址。
public interface IUrlConfigProvider
{
    string GetUrl(string sceneName, string interfaceName);
}
```

```csharp
// 中文注释：HTTP 服务只通过中文接口名拿最终 URL，不在业务代码里硬编码 IP、端口和 API。
var url = urlConfigProvider.GetUrl("SLP", "获取设备清单");
```

在 Unity 中读取 `StreamingAssets` 时要注意平台差异：

- Windows 编辑器和 PC 包通常可以直接读取文件路径。
- Android、WebGL 等平台可能需要通过 `UnityWebRequest` 读取 `StreamingAssets` 内容。
- 配置应在应用启动或 HTTP 模块初始化时加载，并缓存解析结果。
- 如果允许运行中替换配置，应提供显式重新加载入口，不要每次请求都读取文件。

## 调用时机归属

谁需要数据，谁决定调用时机；HTTP Service 只执行请求。

- 面板打开、刷新、分页、筛选时，由 UI 或 ViewModel 决定调用哪个中文接口名。
- 楼层切换、设备选择、告警点击等事件可以触发面板刷新，但仍由对应 UI/ViewModel 决定是否调用 HTTP。
- 实时消息可以提示“数据已变化”，但是否通过 HTTP 补充详情，应由正在显示的面板或 ViewModel 决定。

## 接口日志

HTTP 请求日志由 HTTP Service 或统一请求封装层打印，不要让每个面板自己重复写日志。日志格式统一为：

```text
接口名称 + 参数 + 返回的数据
```

成功请求使用 `Debug.Log`：

```csharp
// 中文注释：接口请求成功时，统一打印接口名称、请求参数和返回数据。
Debug.Log($"接口名称：{interfaceName}，参数：{requestParamsJson}，返回的数据：{responseText}");
```

失败请求使用 `Debug.LogError`：

```csharp
// 中文注释：接口请求失败时，统一打印接口名称、请求参数和错误返回，方便现场排查。
Debug.LogError($"接口名称：{interfaceName}，参数：{requestParamsJson}，返回的数据：{errorText}");
```

注意：

- `interfaceName` 使用配置文件中的中文接口名称。
- 参数应打印调用接口时实际传入的查询参数或请求体。
- 返回的数据成功时打印响应内容，失败时打印错误响应、状态码或异常信息。
- 如果返回数据很大，可以按项目规则截断，但不能丢失接口名称和参数。
- 日志打印应集中在统一请求封装中，避免同一次请求被多处重复打印。

## 与实时消息的关系

实时消息用于告诉 Unity “发生了变化”；HTTP 用于补充“详情是什么”。

常见组合：

- 收到设备告警事件后，告警列表即时增加一条简要记录。
- 用户点击告警详情时，通过 HTTP 查询完整报警内容。
- 收到库存变化事件后，刷新本地状态；用户打开库位面板时 HTTP 查询详细库存。

## 验证重点

- 接口失败时 UI 有明确状态。
- 配置文件缺失或格式错误时有明确错误。
- 通过中文接口名能正确查到 URL。
- 接口项 `socket` 为空时能回退到 `globalSocket`。
- 接口项 `socket` 有值时能覆盖 `globalSocket`。
- 请求成功时使用 `Debug.Log` 打印接口名称、参数和返回数据。
- 请求失败时使用 `Debug.LogError` 打印接口名称、参数和返回数据或错误信息。
- 面板关闭后不会继续刷新已销毁对象。
- 数据为空时不报错。
- 重复打开面板不会重复绑定事件或发起失控请求。
