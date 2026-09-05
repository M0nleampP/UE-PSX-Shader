# UE5.8 — PSX 风格后处理材质完整指南

这个文件包含：
1) 可直接复制到 UE5.8 的 Material Editor 中的 Custom 节点完整文本（Inputs/Output 类型说明）；
2) 在蓝图中运行时创建 Dynamic Material Instance 并把 ScreenSize 传入（含可执行的节点流程和替代方案）；
3) 一个可选但更精确的 Render Target（320×240，Point filter）工作流及其在蓝图中的实现建议。

文件放置于仓库：UE-PSX-Shader

---

一、UE5.8 可直接粘贴的 Custom 节点（每个 Custom 节点请在材质编辑器中新建 Custom node，把下面 Code 区整段复制进去，并在节点右侧的 Inputs 中按说明添输入）

说明：Custom 节点的设置示例
- Node Name: NearestLowResUV
- Output Type: float2
- Inputs (按行)：UV|float2, LowRes|float2
- Code:（整个代码块粘进 Code 区）

---------

Custom Node: NearestLowResUV
- Output Type: float2
- Inputs: UV (float2), LowRes (float2)
Code:
float2 pixelUV = UV * LowRes;
pixelUV = floor(pixelUV) + 0.5;
return pixelUV / LowRes;

用途：把 UV 对齐到 LowRes 像素中心以模拟 nearest-downsample。

---------

Custom Node: ScreenWobble
- Output Type: float2
- Inputs: UV (float2), ScreenSize (float2), WobbleStrength (float)
Code:
float2 pix = UV * ScreenSize;
float n  = frac(sin(dot(pix, float2(12.9898,78.233))) * 43758.5453);
float n2 = frac(sin(dot(pix, float2(93.9898,67.345))) * 12741.2743);
float ox = (n - 0.5) * WobbleStrength;
float oy = (n2 - 0.5) * WobbleStrength;
return UV + float2(ox / ScreenSize.x, oy / ScreenSize.y);

用途：基于屏幕像素位置的 cheap-noise，为 UV 添加像素级微偏移，模拟 PSX 顶点精度抖动。

---------

Custom Node: QuantizeColor
- Output Type: float3
- Inputs: Color (float3), Levels (float)
Code:
return round(Color * Levels) / Levels;

用途：颜色量化（Levels = 31 表示 5-bit）。

---------

Custom Node: BayerDither
- Output Type: float
- Inputs: PixelPos (float2)
Code:
int xi = (int)fmod(PixelPos.x, 4.0);
int yi = (int)fmod(PixelPos.y, 4.0);
int idx = yi * 4 + xi;
static const int bayer[16] = {0,8,2,10, 12,4,14,6, 3,11,1,9, 15,7,13,5};
float v = (float)bayer[idx] / 16.0; // 0..0.9375
return v - 0.5; // roughly [-0.5, +0.4375]

用途：4×4 Bayer 抖动矩阵，用作量化抖动。

---------

Custom Node: SampleSceneAtUV (UE5.8 推荐)
- Output Type: float4
- Inputs: UV (float2)
Code:
// 在 UE5.8 Post-Process Custom 节点中可用：
return SceneTextureLookup(UV, 14);

说明：14 通常对应 PostProcessInput0（在大多数 UE5.8 构建中）。若在你的项目中报错，改用材质编辑器内置的 SceneTexture 节点 (PostProcessInput0) 并将其 UV 输入连接为 UV_Low（或使用下面的 RT 方案）。

---------

二、完整材质主流程（按步骤在材质编辑器里连线）

1) 创建材质 M_PP_PSX：
   - Material Domain = Post Process
   - Shading Model = Unlit
   - Blendable Location = Before Tonemapper

2) 创建材质参数（Scalar / Vector / StaticSwitch）：
   - LowRes (Vector2) 默认 (320,240)
   - ScreenSize (Vector2) 默认 (1920,1080)
   - ColorLevels (Scalar) 默认 31
   - WobbleStrength (Scalar) 默认 0.8
   - DitherStrength (Scalar) 默认 1.0
   - UseWobble (StaticSwitch) 默认 true

3) 获取 UV：使用 ScreenPosition 节点，取其 .xy 输出作为 UV。

4) Wobble (可切换)：
   - 使用 StaticSwitch(UseWobble) 分支：
     - True: UV_W = ScreenWobble(UV, ScreenSize, WobbleStrength)
     - False: UV_W = UV

5) Low-res UV：UV_Low = NearestLowResUV(UV_W, LowRes)

6) 采样场景颜色：ColRGBA = SampleSceneAtUV(UV_Low)  // 或 SceneTexture(PostProcessInput0) 节点
   - Col = ColRGBA.rgb

7) 颜色量化：Col_Q = QuantizeColor(Col, ColorLevels)

8) 计算像素坐标并抖动：
   - PixelPos = floor(UV_W * ScreenSize)
   - dVal = BayerDither(PixelPos)
   - d = dVal * (1.0 / ColorLevels) * DitherStrength

9) 输出颜色：OutColor = saturate(Col_Q + float3(d,d,d))
   - 连接 OutColor 到 Emissive Color（Material Output），Alpha = 1

调试建议：
- 先把 WobbleStrength = 0，看 LowRes + Quantize + Dither 的效果；再开启 Wobble 调整强度。
- 如果看到模糊，优先检查是否使用了 RT + Point filter 或 SceneTextureLookup。

---

三、在蓝图中创建 Dynamic Material Instance 并设置 ScreenSize（两种方法：编辑器 + 运行时）

方法 A（编辑器快速法）
- 在 PostProcessVolume 的 Details 面板里，把一个 Material Instance (基于 M_PP_PSX) 直接加入 Blendables。手动把 ScreenSize 的默认值设置为目标分辨率用于测试。

方法 B（推荐，运行时动态设置）
步骤（Blueprint nodes，Event BeginPlay）：
1. Get GameViewport -> Get Viewport Size (X,Y)
2. Create Dynamic Material Instance (Parent = M_PP_PSX) -> Return Value = MID
   - 如果你已经在 PPV 中放了一个 Material Instance Asset，你可以用它作为 Parent
3. Make Vector (X, Y, 0) -> Set Vector Parameter Value on MID (Parameter Name = "ScreenSize")
4. 将 MID 应用到 PostProcessVolume：
   - 方案 B1（常见）：在编辑器里先把一个 Material Instance 放到 PPV 的 Blendables，然后在 BP 中修改该材质实例。流程：
     a. 在 PPV 的 Blendables 放入一个 Material Instance Asset（比如 MI_PP_PSX_Default）
     b. 在 BeginPlay：Create Dynamic Material Instance with Parent = MI_PP_PSX_Default，设置参数后，把这个 MID 赋回 PPV。具体实现有两种：直接通过 C++/插件使用 PostProcessVolume->AddOrUpdateBlendable；或在 BP 中通过 Set Members in PostProcessSettings（复杂），如果你的项目允许 C++，用 C++ 调用 AddOrUpdateBlendable 最稳妥。
   - 方案 B2（BP 纯实现）替代：将材质实例赋给一个 PostProcessComponent（放在一个持久 Actor 上）。创建一个 Actor，在它上面添加 Post Process Component，并在 BeginPlay 中：
     a. Create Dynamic Material Instance (Parent = M_PP_PSX)
     b. Set Vector Parameter Value (ScreenSize)
     c. PostProcessComponent->Settings.WeightedBlendables.Array.Add (Blendable: MID with Weight 1)
     d. Enable component（或者将 Actor 放在场景里并设置为 Unbound/Global）

注意：Unreal 蓝图默认没有直接的“Add Blendable”节点，某些项目会使用自定义插件或 C++ 扩展来暴露 AddOrUpdateBlendable。如果你需要纯蓝图方案，我可以把一个基于 PostProcessComponent 的 Actor 示例的逐步 BP 实现写成文件并提交。

示例伪节点流程（最简）:
- Event BeginPlay -> Get GameViewport -> Get Viewport Size -> Create Dynamic Material Instance (Parent = M_PP_PSX) -> Set Vector Parameter Value (ScreenSize) -> (把 MID 加到 PPV 或 PostProcessComponent 的 Blendables)

---

四、Render Target 方案（精确 nearest 采样）

何时使用：当你需要绝对保证点过滤 (Point) 的低分辨率像素块效果，并避免任何线性混合时。

流程概述：
1. 创建 Render Target：
   - 目标分辨率：320x240（或 256x224）
   - Texture Filter: Point
   - Clear Color: Black（按需）

2. 创建一个简单材质 M_CopyToRT：
   - Domain = Post Process 或 Unlit Material，用于把 SceneTexture(PostProcessInput0) 直接绘制到 RT 上（很简单：采样 SceneTexture, 输出到 Emissive）。

3. 创建一个小的后处理或自定义渲染通道，把当前帧复制到 RT：
   - 方法：使用 Blueprints 的 Draw Material to Render Target (在 Engine 里可用) 或自定义后处理来把 SceneTexture 渲染到 RT。
   - 示例（Blueprint）：在每帧或每次镜头更新时调用 Draw Material to Render Target(Target = RT_320x240, Material = M_CopyToRT)，这样 RT 将包含 Point-filtered 的低分辨率图像。

4. 最终后处理材质 M_PP_PSX_RT：
   - 在主流程中，把 SampleSceneAtUV(UV_Low) 的采样替换为 Texture Sample of RT（设置为 Texture Sample 和 Point filter）。
   - 其它量化/抖动逻辑不变。

注意性能：每帧 Draw Material to Render Target 会有开销（额外一次 RT 写入）。你可以降低频率（例如每 N 帧更新一次）来节省开销，代价是 wobble / 动态物体的延迟更新。

蓝图实现要点（Draw Material to RT 示例）
- 在持久化 Actor（例如一个 Manager Actor）里：
  - 在 BeginPlay 创建或引用 Render Target 资产
  - Create Dynamic Material Instance for M_CopyToRT（如果需要参数）
  - 每帧（Event Tick）或按固定频率调用 Draw Material to Render Target (Render Target = RT_320x240, Material = M_CopyToRT)
  - 然后把 RT 连接到最终后处理材质的 Texture 参数（在 BeginPlay 或首次初始化时创建 Dynamic Material Instance for M_PP_PSX_RT 并 Set Texture Parameter Value 将 RT 给到材质）

示例 BP 伪节点（Tick 方案）:
- Event BeginPlay -> Create Dynamic Material Instance (M_CopyToRT) -> Store MID_Copy
- Event Tick -> Draw Material to Render Target (RT_320x240, MID_Copy)
- （如果使用 M_PP_PSX_RT 的 Dynamic Instance）在 BeginPlay 创建 MID_PP = CreateDynamicMaterialInstance(M_PP_PSX_RT) -> Set Texture Parameter (ParamName = "LowResRT", Value = RT_320x240) -> 应用 MID_PP 到 PPV 的 Blendables 或 PostProcessComponent

提示：Draw Material to Render Target 的执行成本可能高，优先考虑在性能预算内调整更新频率。

---

五、我可以为你提交的文件（到仓库）
我已经把本仓库的 README.md 更新为包含基础后处理内容（之前已提交）。如果你想，我现在可以把这个更详细的 UE5.8 指南（本文件）也提交到仓库（例如路径：UE5.8_FULL_SETUP.md）。

是否现在把此文件提交到你的仓库（UE-PSX-Shader/UE5.8_FULL_SETUP.md）？

- 回答 “是” 我就会把本文件写入仓库并提交；
- 回答 “否” 我就把内容保留在聊天里，你可以复制粘贴。

如果同意提交，请确认并告诉我你希望的文件名（默认：UE5.8_FULL_SETUP.md）。
