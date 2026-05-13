# hello_xr 进程启动过程详解

## 概述

hello_xr 是 OpenXR SDK 中的主要示例程序，演示了从初始化到渲染的完整 OpenXR 生命周期。本文档详细追踪从 `main()` 到渲染循环的每一个 API 调用。

**入口点:** `src/tests/hello_xr/main.cpp:289` — `main(int argc, char* argv[])`

---

## 整体调用流程图

```
main()
 ├─ 阶段0: 平台/图形插件构造
 │   ├─ CreatePlatformPlugin()     → CoInitializeEx (Win32)
 │   ├─ CreateGraphicsPlugin()     → 构造图形插件对象
 │   └─ CreateOpenXrProgram()      → 构造 OpenXrProgram（无 API 调用）
 │
 ├─ 阶段1: CreateInstance()
 │   ├─ LogLayersAndExtensions()   → xrEnumerateApiLayerProperties
 │   │                                xrEnumerateInstanceExtensionProperties
 │   ├─ CreateInstanceInternal()   → xrCreateInstance ★
 │   └─ LogInstanceInfo()          → xrGetInstanceProperties
 │
 ├─ 阶段2: InitializeSystem()
 │   ├─ xrGetSystem ★
 │   └─ xrEnumerateEnvironmentBlendModes
 │
 ├─ 阶段3: InitializeDevice()
 │   ├─ LogViewConfigurations()    → xrEnumerateViewConfigurations
 │   │                                xrGetViewConfigurationProperties
 │   │                                xrEnumerateViewConfigurationViews
 │   │                                xrEnumerateEnvironmentBlendModes
 │   └─ graphicsPlugin->InitializeDevice()
 │       └─ xrGetD3D11GraphicsRequirementsKHR (以 D3D11 为例)
 │
 ├─ 阶段4: InitializeSession()
 │   ├─ xrCreateSession ★
 │   ├─ LogReferenceSpaces()       → xrEnumerateReferenceSpaces
 │   ├─ InitializeActions()        → xrCreateActionSet ★
 │   │                                xrStringToPath (x16)
 │   │                                xrCreateAction ★ (x4)
 │   │                                xrSuggestInteractionProfileBindings ★ (x5)
 │   │                                xrCreateActionSpace ★ (x2)
 │   │                                xrAttachSessionActionSets ★
 │   ├─ CreateVisualizedSpaces()   → xrCreateReferenceSpace ★ (x7)
 │   └─ 创建 appSpace              → xrCreateReferenceSpace ★
 │
 ├─ 阶段5: CreateSwapchains()
 │   ├─ xrGetSystemProperties
 │   ├─ xrEnumerateViewConfigurationViews
 │   ├─ xrEnumerateSwapchainFormats
 │   └─ 每个 view:
 │       ├─ xrCreateSwapchain ★ (颜色)
 │       ├─ xrEnumerateSwapchainImages (x2)
 │       ├─ xrCreateSwapchain ★ (深度，若支持)
 │       └─ xrEnumerateSwapchainImages (x2)
 │
 └─ 阶段6: 渲染主循环
     while (!quit) {
       ├─ PollEvents()
       │   └─ xrPollEvent (循环直至 XR_EVENT_UNAVAILABLE)
       │       └─ 状态 = READY → xrBeginSession ★
       │       └─ 状态 = STOPPING → xrEndSession
       │
       ├─ PollActions()
       │   ├─ xrSyncActions
       │   ├─ xrGetActionStateFloat (左右手)
       │   ├─ xrGetActionStatePose (左右手)
       │   ├─ xrApplyHapticFeedback (挤压 > 90% 时)
       │   ├─ xrGetActionStateBoolean (退出操作)
       │   └─ xrRequestExitSession (退出触发时)
       │
       └─ RenderFrame()
           ├─ xrWaitFrame
           ├─ xrBeginFrame
           ├─ RenderLayer():
           │   ├─ xrLocateViews
           │   ├─ xrLocateSpace (每个可视化空间 + 每只手)
           │   └─ 每个 view:
           │       ├─ xrAcquireSwapchainImage
           │       ├─ xrWaitSwapchainImage
           │       ├─ graphicsPlugin->RenderView()  (GPU 渲染)
           │       └─ xrReleaseSwapchainImage
           └─ xrEndFrame
     }
```

> ★ 标记表示对象创建/销毁类 API

---

## 阶段0: 平台和图形 API 初始化（OpenXR 前）

**文件:** `src/tests/hello_xr/main.cpp`

### 步骤 0.1: 解析命令行参数
**位置:** `main.cpp:292-295`
```cpp
UpdateOptionsFromCommandLine(*options, argc, argv)
```
解析 `--graphics|-g`、`--formfactor|-ff`、`--viewconfig|-vc`、`--blendmode|-bm`、`--space|-s` 等参数。

### 步骤 0.2: 创建平台插件
**位置:** `main.cpp:311` → `src/tests/hello_xr/platformplugin_factory.cpp:11`
```cpp
CreatePlatformPlugin(data)
```
Windows 实现 (`platformplugin_win32.cpp:22`):
- 调用 `CoInitializeEx(nullptr, COINIT_MULTITHREADED)` 初始化 COM
- 返回空的扩展列表（Win32 平台不需要额外的实例扩展）

### 步骤 0.3: 创建图形插件
**位置:** `main.cpp:314` → `src/tests/hello_xr/graphicsplugin_factory.cpp:37`
```cpp
CreateGraphicsPlugin(options->GraphicsPlugin)
```
根据 `--graphics` 参数查找并构造对应的图形插件（D3D11、D3D12、OpenGL、OpenGL ES、Vulkan、Vulkan2、Metal 之一）。

### 步骤 0.4: 创建 OpenXR 程序对象
**位置:** `main.cpp:317` → `src/tests/hello_xr/openxr_program.cpp:1134`
```cpp
CreateOpenXrProgram(platformPlugin, graphicsPlugin)
```
构造 `OpenXrProgram` 对象（`openxr_program.cpp:91-93`），仅存储两个 shared_ptr，所有 OpenXR 句柄初始化为 `XR_NULL_HANDLE`。

---

## 阶段1: CreateInstance() — 创建 XrInstance

**位置:** `src/tests/hello_xr/openxr_program.cpp:218`

### 1a. LogLayersAndExtensions() — 枚举可用的 API 层和扩展

**位置:** `openxr_program.cpp:124`

| # | API 调用 | 说明 |
|---|---------|------|
| 1 | `xrEnumerateInstanceExtensionProperties(nullptr, 0, &count, nullptr)` | 获取不含层的扩展数量 |
| 2 | `xrEnumerateInstanceExtensionProperties(nullptr, count, &count, exts.data())` | 获取不含层的扩展属性列表 |
| 3 | `xrEnumerateApiLayerProperties(0, &count, nullptr)` | 获取可用 API 层数量 |
| 4 | `xrEnumerateApiLayerProperties(count, &count, layers.data())` | 获取 API 层属性列表 |
| 5+ | `xrEnumerateInstanceExtensionProperties(layerName, ...)` ×2 | 对每个 API 层，枚举该层支持的扩展 |

### 1b. CreateInstanceInternal() — 实际创建实例

**位置:** `openxr_program.cpp:171`

| # | API 调用 | 说明 |
|---|---------|------|
| 6 | `xrEnumerateInstanceExtensionProperties(nullptr, 0, &count, nullptr)` | 再次查询实例扩展数量 |
| 7 | `xrEnumerateInstanceExtensionProperties(nullptr, count, &count, exts.data())` | 再次查询实例扩展列表 |
| — | — | 检查 `XR_KHR_composition_layer_depth` 是否支持 |
| **8** | **`xrCreateInstance(&createInfo, &m_instance)`** | **创建 XrInstance 对象** |

**XrInstanceCreateInfo 结构体内容:**
```cpp
XrInstanceCreateInfo createInfo{XR_TYPE_INSTANCE_CREATE_INFO};
createInfo.next = m_platformPlugin->GetInstanceCreateExtension();
createInfo.enabledExtensionCount = extensions.size();
createInfo.enabledExtensionNames = extensions.data();  // 平台扩展 + 图形扩展 + 深度扩展(如支持)
strcpy(createInfo.applicationInfo.applicationName, "HelloXR");
createInfo.applicationInfo.apiVersion = XR_API_VERSION_1_0;
```

### 1c. LogInstanceInfo() — 记录实例信息

**位置:** `openxr_program.cpp:161`

| # | API 调用 | 说明 |
|---|---------|------|
| 9 | `xrGetInstanceProperties(m_instance, &instanceProperties)` | 获取运行时名称和版本号 |

---

## 阶段2: InitializeSystem() — 获取 XrSystemId

**位置:** `openxr_program.cpp:310`

| # | API 调用 | 说明 |
|---|---------|------|
| **10** | **`xrGetSystem(m_instance, &systemInfo, &m_systemId)`** | **获取 SystemId** |
| 11 | `xrEnumerateEnvironmentBlendModes(m_instance, m_systemId, viewConfigType, 0, &count, nullptr)` | 查询混合模式数量 |
| 12 | `xrEnumerateEnvironmentBlendModes(m_instance, m_systemId, viewConfigType, count, &count, blendModes.data())` | 获取混合模式列表，选择第一个（或命令行指定值） |

**关键数据结构:**
```cpp
XrSystemGetInfo systemInfo{XR_TYPE_SYSTEM_GET_INFO};
systemInfo.formFactor = formFactor;  // XR_FORM_FACTOR_HEAD_MOUNTED_DISPLAY
```

验证 `viewConfigType` 必须是 `XR_VIEW_CONFIGURATION_TYPE_PRIMARY_MONO` 或 `PRIMARY_STEREO`。

---

## 阶段3: InitializeDevice() — 初始化图形设备

**位置:** `openxr_program.cpp:352`

### 3a. LogViewConfigurations() — 枚举视图配置

**位置:** `openxr_program.cpp:226`

| # | API 调用 | 说明 |
|---|---------|------|
| 13 | `xrEnumerateViewConfigurations(m_instance, m_systemId, 0, &count, nullptr)` | 查询视图配置类型数量 |
| 14 | `xrEnumerateViewConfigurations(m_instance, m_systemId, count, &count, types.data())` | 获取视图配置类型列表 |
| | | **对每个视图配置类型:** |
| 15 | `xrGetViewConfigurationProperties(m_instance, m_systemId, type, &props)` | 获取 FOV 可变性等属性 |
| 16 | `xrEnumerateViewConfigurationViews(m_instance, m_systemId, type, 0, &count, nullptr)` | 查询该配置的视图数量 |
| 17 | `xrEnumerateViewConfigurationViews(m_instance, m_systemId, type, count, &count, views.data())` | 获取推荐/最大分辨率、采样数 |
| 18 | `xrEnumerateEnvironmentBlendModes(m_instance, m_systemId, type, 0, &count, nullptr)` | 查询混合模式数量 |
| 19 | `xrEnumerateEnvironmentBlendModes(m_instance, m_systemId, type, count, &count, modes.data())` | 获取混合模式列表 |

### 3b. 图形设备初始化

**位置:** `openxr_program.cpp:357`
```cpp
m_graphicsPlugin->InitializeDevice(m_instance, m_systemId);
```

以 D3D11 为例 (`src/tests/hello_xr/graphicsplugin_d3d11.cpp`):
| API 调用 | 说明 |
|---------|------|
| `xrGetD3D11GraphicsRequirementsKHR(m_instance, m_systemId, &reqs)` | 获取 D3D11 适配器 LUID 和最低 Feature Level |
| — | 然后创建 D3D11 设备、上下文、交换链数据等 |

其他图形后端类似，调用对应的 `xrGet<Graphics>GraphicsRequirementsKHR`。

---

## 阶段4: InitializeSession() — 创建 XrSession

**位置:** `openxr_program.cpp:595`

| # | API 调用 | 说明 |
|---|---------|------|
| **20** | **`xrCreateSession(m_instance, &createInfo, &m_session)`** | **创建 XrSession** |

```cpp
XrSessionCreateInfo createInfo{XR_TYPE_SESSION_CREATE_INFO};
createInfo.next = m_graphicsPlugin->GetGraphicsBinding();  // 如 XrGraphicsBindingD3D11KHR
createInfo.systemId = m_systemId;
```

### 4a. LogReferenceSpaces() — 枚举参考空间类型

**位置:** `openxr_program.cpp:360`

| # | API 调用 | 说明 |
|---|---------|------|
| 21 | `xrEnumerateReferenceSpaces(m_session, 0, &count, nullptr)` | 查询参考空间数量 |
| 22 | `xrEnumerateReferenceSpaces(m_session, count, &count, spaces.data())` | 获取参考空间类型列表 |

### 4b. InitializeActions() — 初始化动作输入系统

**位置:** `openxr_program.cpp:386`

#### 创建 ActionSet
| # | API 调用 | 说明 |
|---|---------|------|
| **23** | **`xrCreateActionSet(m_instance, &info, &m_input.actionSet)`** | 创建 "gameplay" 动作集 |

#### 解析语义路径
| # | API 调用 | 路径 |
|---|---------|------|
| 24 | `xrStringToPath(...)` | `/user/hand/left` |
| 25 | `xrStringToPath(...)` | `/user/hand/right` |

#### 创建 Action
| # | API 调用 | 类型 | 名称 |
|---|---------|------|------|
| **26** | **`xrCreateAction(actionSet, &info, &m_input.grabAction)`** | `FLOAT_INPUT` | "grab_object" |
| **27** | **`xrCreateAction(actionSet, &info, &m_input.poseAction)`** | `POSE_INPUT` | "hand_pose" |
| **28** | **`xrCreateAction(actionSet, &info, &m_input.vibrateAction)`** | `VIBRATION_OUTPUT` | "vibrate_hand" |
| **29** | **`xrCreateAction(actionSet, &info, &m_input.quitAction)`** | `BOOLEAN_INPUT` | "quit_session" (无子路径) |

#### 解析控制器输入路径 (14 次 xrStringToPath)
| # | 路径示例 |
|---|---------|
| 30-43 | `/user/hand/left/input/select/click`, `/user/hand/left/input/squeeze/value`, `/user/hand/left/input/squeeze/force`, `/user/hand/left/input/grip/pose`, `/user/hand/left/output/haptic`, `/user/hand/left/input/menu/click`, `/user/hand/left/input/b/click`, `/user/hand/left/input/trigger/value` (以及右手对称路径) |

#### 为 5 个交互配置文件建议绑定

对每个配置文件，先 `xrStringToPath` 获取配置文件的路径，再 `xrSuggestInteractionProfileBindings`：

| # | 配置文件 | xrStringToPath | xrSuggestInteractionProfileBindings |
|---|---------|---------------|-------------------------------------|
| 44-45 | `/interaction_profiles/khr/simple_controller` | ✓ | ✓ |
| 46-47 | `/interaction_profiles/oculus/touch_controller` | ✓ | ✓ |
| 48-49 | `/interaction_profiles/htc/vive_controller` | ✓ | ✓ |
| 50-51 | `/interaction_profiles/valve/index_controller` | ✓ | ✓ |
| 52-53 | `/interaction_profiles/microsoft/motion_controller` | ✓ | ✓ |

#### 创建 ActionSpace 并附加 ActionSet

| # | API 调用 | 说明 |
|---|---------|------|
| **54** | **`xrCreateActionSpace(m_session, &info, &handSpace[LEFT])`** | 左手 Grip Pose 的动作空间 |
| **55** | **`xrCreateActionSpace(m_session, &info, &handSpace[RIGHT])`** | 右手 Grip Pose 的动作空间 |
| **56** | **`xrAttachSessionActionSets(m_session, &attachInfo)`** | 将 actionSet 附加到会话 |

### 4c. CreateVisualizedSpaces() — 创建可视化参考空间

**位置:** `openxr_program.cpp:576`

为 7 种参考空间变体各调用一次 `xrCreateReferenceSpace`：

| # | API 调用 | 空间 |
|---|---------|------|
| **57-63** | **`xrCreateReferenceSpace(m_session, &info, &space)`** | "ViewFront", "Local", "Stage", "StageLeft", "StageRight", "StageLeftRotated", "StageRightRotated" |

> 部分空间可能创建失败（某些运行时不支持），失败时仅记录 warning。

### 4d. 创建应用程序参考空间

**位置:** `openxr_program.cpp:613`

| # | API 调用 | 说明 |
|---|---------|------|
| **64** | **`xrCreateReferenceSpace(m_session, &info, &m_appSpace)`** | 创建应用程序的主参考空间（默认 "Stage"） |

---

## 阶段5: CreateSwapchains() — 创建交换链

**位置:** `openxr_program.cpp:618`

### 查询系统属性和视图配置

| # | API 调用 | 说明 |
|---|---------|------|
| 65 | `xrGetSystemProperties(m_instance, m_systemId, &systemProperties)` | 获取 maxSwapchainImageWidth/Height、maxLayerCount 等 |
| 66 | `xrEnumerateViewConfigurationViews(m_instance, m_systemId, viewConfigType, 0, &viewCount, nullptr)` | 获取视图数量（立体 = 2，单目 = 1） |
| 67 | `xrEnumerateViewConfigurationViews(m_instance, m_systemId, viewConfigType, viewCount, &viewCount, configViews.data())` | 缓存到 `m_configViews` |
| 68 | `xrEnumerateSwapchainFormats(m_session, 0, &count, nullptr)` | 查询支持的格式数量 |
| 69 | `xrEnumerateSwapchainFormats(m_session, count, &count, formats.data())` | 获取格式列表并选择颜色/深度格式 |

### 为每个视图创建交换链

对每个视图（`viewCount` 次迭代）：

| # | API 调用 | 说明 |
|---|---------|------|
| **70a** | **`xrCreateSwapchain(m_session, &createInfo, &swapchain.handle)`** | 创建颜色交换链 |
| 70b | `xrEnumerateSwapchainImages(swapchain, 0, &imageCount, nullptr)` | 获取颜色交换链图像数量 |
| 70c | `xrEnumerateSwapchainImages(swapchain, imageCount, &imageCount, images.data())` | 获取图像并交给图形插件管理 |
| **70d** | **`xrCreateSwapchain(m_session, &info, &depthSwapchain.handle)`** | 创建深度交换链（若深度格式可用） |
| 70e | `xrEnumerateSwapchainImages(depthSwapchain, 0, &count, nullptr)` | 获取深度交换链图像数量 |
| 70f | `xrEnumerateSwapchainImages(depthSwapchain, count, &count, images.data())` | 获取深度图像 |

**XrSwapchainCreateInfo 颜色交换链参数:**
```cpp
swapchainCreateInfo.arraySize = 1;
swapchainCreateInfo.format = m_colorSwapchainFormat;
swapchainCreateInfo.width = vp.recommendedImageRectWidth;
swapchainCreateInfo.height = vp.recommendedImageRectHeight;
swapchainCreateInfo.mipCount = 1;
swapchainCreateInfo.faceCount = 1;
swapchainCreateInfo.sampleCount = graphicsPlugin->GetSupportedSwapchainSampleCount(vp);
swapchainCreateInfo.usageFlags = XR_SWAPCHAIN_USAGE_SAMPLED_BIT | XR_SWAPCHAIN_USAGE_COLOR_ATTACHMENT_BIT;
```

---

## 阶段6: 渲染主循环

**位置:** `main.cpp:329-343`

```cpp
while (!quitKeyPressed) {
    bool exitRenderLoop = false;
    program->PollEvents(&exitRenderLoop, &requestRestart);
    if (exitRenderLoop) break;

    if (program->IsSessionRunning()) {
        program->PollActions();
        program->RenderFrame();
    } else {
        std::this_thread::sleep_for(std::chrono::milliseconds(250));
    }
}
```

### 6a. PollEvents() — 事件轮询

**位置:** `openxr_program.cpp:781`

内部调用 `TryReadNextEvent()` (`openxr_program.cpp:761`):

| # | API 调用 | 说明 |
|---|---------|------|
| L1 | **`xrPollEvent(m_instance, &m_eventDataBuffer)`** | 非阻塞轮询，循环消费直至 `XR_EVENT_UNAVAILABLE` |

处理的事件类型：

| 事件类型 | 处理方式 |
|---------|---------|
| `XR_TYPE_EVENT_DATA_INSTANCE_LOSS_PENDING` | `exitRenderLoop=true`, `requestRestart=true` |
| `XR_TYPE_EVENT_DATA_SESSION_STATE_CHANGED` | 见下方状态机 |
| `XR_TYPE_EVENT_DATA_INTERACTION_PROFILE_CHANGED` | 记录当前绑定的输入源名称 |
| `XR_TYPE_EVENT_DATA_REFERENCE_SPACE_CHANGE_PENDING` | 忽略 |
| 其他 | 记录 "Ignoring event type %d" |

#### 会话状态机

| 新状态 | API 调用 | 动作 |
|--------|---------|------|
| `XR_SESSION_STATE_READY` | **`xrBeginSession(m_session, &info)`** | `m_sessionRunning = true` |
| `XR_SESSION_STATE_STOPPING` | `xrEndSession(m_session)` | `m_sessionRunning = false` |
| `XR_SESSION_STATE_EXITING` | — | `exitRenderLoop = true`，不重启 |
| `XR_SESSION_STATE_LOSS_PENDING` | — | `exitRenderLoop = true`，重启以获取新实例 |

### 6b. PollActions() — 动作状态同步

**位置:** `openxr_program.cpp:900`

| # | API 调用 | 说明 |
|---|---------|------|
| L2 | **`xrSyncActions(m_session, &syncInfo)`** | 同步所有动作的当前状态 |
| L3 | `xrGetActionStateFloat(m_session, &getInfo, &grabValue)` (左手) | 读取抓取动作值 [0,1] |
| L4 | `xrGetActionStatePose(m_session, &getInfo, &poseState)` (左手) | 检查手部姿势是否活跃 |
| L5 | `xrGetActionStateFloat(m_session, &getInfo, &grabValue)` (右手) | 右手抓取值 |
| L6 | `xrGetActionStatePose(m_session, &getInfo, &poseState)` (右手) | 右手姿势状态 |
| L7 | `xrGetActionStateBoolean(m_session, &getInfo, &quitValue)` | 检查退出操作 |

当抓取值 > 0.9 时触发触觉反馈：
| # | API 调用 | 说明 |
|---|---------|------|
| L* | **`xrApplyHapticFeedback(m_session, &hapticInfo, &vibration)`** | 振幅 0.5，持续 `XR_MIN_HAPTIC_DURATION` |

当退出操作触发时：
| # | API 调用 | 说明 |
|---|---------|------|
| L* | **`xrRequestExitSession(m_session)`** | 请求会话退出 → 触发 `SESSION_STATE_STOPPING` 事件 |

### 6c. RenderFrame() — 帧渲染

**位置:** `openxr_program.cpp:949`

#### 帧同步
| # | API 调用 | 说明 |
|---|---------|------|
| **L** | **`xrWaitFrame(m_session, &frameWaitInfo, &frameState)`** | 阻塞至下一帧可用，返回 `predictedDisplayTime` 和 `shouldRender` |
| **L** | **`xrBeginFrame(m_session, &frameBeginInfo)`** | 标记帧开始 |

#### RenderLayer() — 渲染合成层

**位置:** `openxr_program.cpp:977`

| # | API 调用 | 说明 |
|---|---------|------|
| **L** | **`xrLocateViews(m_session, &viewLocateInfo, &viewState, viewCount, &count, views.data())`** | 获取双眼的 View 姿态和 FOV |
| | | 若位置/方向跟踪无效 → 返回 false 跳过本帧 |

**空间定位** — 对每个可视化参考空间 (≤7):
| # | API 调用 | 说明 |
|---|---------|------|
| **L** | **`xrLocateSpace(visualizedSpace, m_appSpace, time, &location)`** | 获取参考空间相对 appSpace 的姿态（用于渲染调试立方体） |

**手部空间定位** — 左右手各一次:
| # | API 调用 | 说明 |
|---|---------|------|
| **L** | **`xrLocateSpace(handSpace, m_appSpace, time, &location)`** | 获取手部空间姿态（用于渲染手部立方体） |

**交换链操作** — 对每个视图 (1 或 2):
| # | API 调用 | 说明 |
|---|---------|------|
| **L** | **`xrAcquireSwapchainImage(swapchain, &acquireInfo, &imageIndex)`** | 获取下一个可用的交换链图像索引 |
| **L** | **`xrWaitSwapchainImage(swapchain, &waitInfo)`** | 等待图像就绪（`timeout = XR_INFINITE_DURATION`） |
| — | `graphicsPlugin->RenderView(...)` | 执行实际的 GPU 渲染（非 OpenXR API） |
| **L** | **`xrReleaseSwapchainImage(swapchain, &releaseInfo)`** | 释放交换链图像 |

#### 帧结束
**位置:** `openxr_program.cpp:969`

| # | API 调用 | 说明 |
|---|---------|------|
| **L** | **`xrEndFrame(m_session, &frameEndInfo)`** | 将合成层提交给运行时 |

```cpp
XrFrameEndInfo frameEndInfo{XR_TYPE_FRAME_END_INFO};
frameEndInfo.displayTime = frameState.predictedDisplayTime;
frameEndInfo.environmentBlendMode = m_blendMode;
frameEndInfo.layerCount = layers.size();
frameEndInfo.layers = layers.data();  // XrCompositionLayerProjection
```

---

## 关机流程

**位置:** `openxr_program.cpp:95-122` (析构函数)

| 顺序 | API 调用 | 对象 |
|------|---------|------|
| 1 | `xrDestroySpace(...)` ×2 | 左右手 ActionSpace |
| 2 | `xrDestroyActionSet(...)` | "gameplay" ActionSet |
| 3 | `xrDestroySwapchain(...)` ×viewCount | 颜色交换链 |
| 4 | `xrDestroySpace(...)` ×N | 可视化参考空间 (≤7) |
| 5 | `xrDestroySpace(...)` | appSpace |
| 6 | `xrDestroySession(...)` | XrSession |
| 7 | `xrDestroyInstance(...)` | XrInstance |

---

## 核心 OpenXR 对象生命周期一览

| OpenXR 对象 | 创建 API | 代码位置 | 销毁 API | 数量 |
|-------------|---------|---------|---------|------|
| `XrInstance` | `xrCreateInstance` | `openxr_program.cpp:215` | `xrDestroyInstance` | 1 |
| `XrSession` | `xrCreateSession` | `openxr_program.cpp:605` | `xrDestroySession` | 1 |
| `XrSpace` (appSpace) | `xrCreateReferenceSpace` | `openxr_program.cpp:614` | `xrDestroySpace` | 1 |
| `XrSpace` (visualized) | `xrCreateReferenceSpace` | `openxr_program.cpp:585` | `xrDestroySpace` | ≤7 |
| `XrSpace` (hand) | `xrCreateActionSpace` | `openxr_program.cpp:566-568` | `xrDestroySpace` | 2 |
| `XrActionSet` | `xrCreateActionSet` | `openxr_program.cpp:393` | `xrDestroyActionSet` | 1 |
| `XrAction` | `xrCreateAction` | `openxr_program.cpp:409-435` | (随 ActionSet 隐式销毁) | 4 |
| `XrSwapchain` (color) | `xrCreateSwapchain` | `openxr_program.cpp:706` | `xrDestroySwapchain` | viewCount |
| `XrSwapchain` (depth) | `xrCreateSwapchain` | `openxr_program.cpp:727` | `xrDestroySwapchain` | viewCount (可选) |

---

## 每帧执行的 API (渲染循环)

| API | 频率 | 说明 |
|-----|------|------|
| `xrPollEvent` | 每帧至少 1 次 | 非阻塞轮询，消费所有待处理事件 |
| `xrSyncActions` | 每帧 1 次 | 同步当前动作状态 |
| `xrGetActionStateFloat` | 每帧 2 次 | 左右手抓取操作 |
| `xrGetActionStatePose` | 每帧 2 次 | 左右手姿势状态 |
| `xrGetActionStateBoolean` | 每帧 1 次 | 退出操作 |
| `xrApplyHapticFeedback` | 每帧 0-2 次 | 仅在抓取 > 0.9 时 |
| `xrWaitFrame` | 每帧 1 次 | 阻塞等待帧时机 |
| `xrBeginFrame` | 每帧 1 次 | |
| `xrLocateViews` | 每帧 1 次 | |
| `xrLocateSpace` | 每帧 ≤9 次 | 可视化空间(≤7) + 双手(2) |
| `xrAcquireSwapchainImage` | 每帧 viewCount 次 | |
| `xrWaitSwapchainImage` | 每帧 viewCount 次 | |
| `xrReleaseSwapchainImage` | 每帧 viewCount 次 | |
| `xrEndFrame` | 每帧 1 次 | |

---

## 关键文件索引

| 文件 | 内容 |
|------|------|
| `src/tests/hello_xr/main.cpp` | `main()` 入口、命令行解析、主循环 |
| `src/tests/hello_xr/openxr_program.h` | `IOpenXrProgram` 抽象接口 |
| `src/tests/hello_xr/openxr_program.cpp` | `OpenXrProgram` 实现 — 所有初始化/渲染逻辑 |
| `src/tests/hello_xr/platformplugin.h` | `IPlatformPlugin` 接口 |
| `src/tests/hello_xr/platformplugin_win32.cpp` | Windows 平台插件 (CoInitializeEx) |
| `src/tests/hello_xr/graphicsplugin.h` | `IGraphicsPlugin` 接口 |
| `src/tests/hello_xr/graphicsplugin_d3d11.cpp` | D3D11 图形插件实现 |
| `src/tests/hello_xr/options.h` | `Options` 结构体（命令行参数） |

---

## hello_xr 启动过程中完整 OpenXR API 调用索引

| API 函数 | 首次出现位置 | 类型 | 频率 |
|----------|-------------|------|------|
| `xrEnumerateApiLayerProperties` | `openxr_program.cpp:147` | 查询 | 启动时 1 次 |
| `xrEnumerateInstanceExtensionProperties` | `openxr_program.cpp:128,186` | 查询 | 启动时多次 |
| `xrCreateInstance` | `openxr_program.cpp:215` | 创建 | 启动时 1 次 |
| `xrGetInstanceProperties` | `openxr_program.cpp:165` | 查询 | 启动时 1 次 |
| `xrGetSystem` | `openxr_program.cpp:317` | 创建 | 启动时 1 次 |
| `xrEnumerateViewConfigurations` | `openxr_program.cpp:231` | 查询 | Logger 调用 |
| `xrGetViewConfigurationProperties` | `openxr_program.cpp:243` | 查询 | Logger 调用 |
| `xrEnumerateViewConfigurationViews` | `openxr_program.cpp:249,640` | 查询 | Logger + Swapchain 创建 |
| `xrEnumerateEnvironmentBlendModes` | `openxr_program.cpp:278,335` | 查询 | Logger + 系统初始化 |
| `xrGetSystemProperties` | `openxr_program.cpp:625` | 查询 | Swapchain 创建时 1 次 |
| `xrGetD3D11GraphicsRequirementsKHR` | `graphicsplugin_d3d11.cpp` | 查询 | 设备初始化时 1 次 |
| `xrCreateSession` | `openxr_program.cpp:605` | 创建 | 启动时 1 次 |
| `xrEnumerateReferenceSpaces` | `openxr_program.cpp:364` | 查询 | Logger 调用 |
| `xrCreateActionSet` | `openxr_program.cpp:393` | 创建 | 启动时 1 次 |
| `xrStringToPath` | `openxr_program.cpp:397-463` | 查询 | 启动时 ~16 次 |
| `xrCreateAction` | `openxr_program.cpp:409-435` | 创建 | 启动时 4 次 |
| `xrSuggestInteractionProfileBindings` | `openxr_program.cpp:483-560` | 设置 | 启动时 5 次 |
| `xrCreateActionSpace` | `openxr_program.cpp:566` | 创建 | 启动时 2 次 |
| `xrAttachSessionActionSets` | `openxr_program.cpp:573` | 设置 | 启动时 1 次 |
| `xrCreateReferenceSpace` | `openxr_program.cpp:585,613` | 创建 | 启动时 ≤8 次 |
| `xrEnumerateSwapchainFormats` | `openxr_program.cpp:652` | 查询 | Swapchain 创建时 |
| `xrCreateSwapchain` | `openxr_program.cpp:706,727` | 创建 | 每视图 1-2 次 |
| `xrEnumerateSwapchainImages` | `openxr_program.cpp:711,732` | 查询 | 每交换链 2 次 |
| `xrPollEvent` | `openxr_program.cpp:766` | 运行时 | 每帧 ≥1 次 |
| `xrBeginSession` | `openxr_program.cpp:832` | 运行时 | 首次 READY 事件时 |
| `xrEndSession` | `openxr_program.cpp:839` | 运行时 | STOPPING 事件时 |
| `xrSyncActions` | `openxr_program.cpp:908` | 运行时 | 每帧 1 次 |
| `xrGetActionStateFloat` | `openxr_program.cpp:917` | 运行时 | 每帧 2 次 |
| `xrGetActionStatePose` | `openxr_program.cpp:936` | 运行时 | 每帧 2 次 |
| `xrGetActionStateBoolean` | `openxr_program.cpp:943` | 运行时 | 每帧 1 次 |
| `xrApplyHapticFeedback` | `openxr_program.cpp:930` | 运行时 | 条件触发 |
| `xrRequestExitSession` | `openxr_program.cpp:945` | 运行时 | 条件触发 |
| `xrWaitFrame` | `openxr_program.cpp:954` | 运行时 | 每帧 1 次 |
| `xrBeginFrame` | `openxr_program.cpp:957` | 运行时 | 每帧 1 次 |
| `xrLocateViews` | `openxr_program.cpp:990` | 运行时 | 每帧 1 次 |
| `xrLocateSpace` | `openxr_program.cpp:1011,1027` | 运行时 | 每帧 ≤9 次 |
| `xrAcquireSwapchainImage` | `openxr_program.cpp:1054` | 运行时 | 每帧每视图 1 次 |
| `xrWaitSwapchainImage` | `openxr_program.cpp:1058` | 运行时 | 每帧每视图 1 次 |
| `xrReleaseSwapchainImage` | `openxr_program.cpp:1086` | 运行时 | 每帧每视图 1 次 |
| `xrEndFrame` | `openxr_program.cpp:974` | 运行时 | 每帧 1 次 |
| `xrDestroySpace` | `openxr_program.cpp:98,107,112` | 销毁 | 关机时 |
| `xrDestroyActionSet` | `openxr_program.cpp:100` | 销毁 | 关机时 |
| `xrDestroySwapchain` | `openxr_program.cpp:104` | 销毁 | 关机时 |
| `xrDestroySession` | `openxr_program.cpp:116` | 销毁 | 关机时 |
| `xrDestroyInstance` | `openxr_program.cpp:120` | 销毁 | 关机时 |

---

*文档生成日期: 2026-05-08*
*基于 OpenXR SDK 1.1.59.1*
