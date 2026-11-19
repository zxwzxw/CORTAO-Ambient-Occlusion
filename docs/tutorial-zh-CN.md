# CORTAO Ambient Occlusion 1.2.1 — 使用教程

[English](tutorial-en.md) · [文档首页](../README.md)

CORTAO 是屏幕空间环境光遮蔽（AO）效果。它读取场景深度与延迟渲染的 GBuffer 法线，在常规后处理之前把可见度合成到场景颜色中。它能加强接触边缘、凹槽和墙角的层次感，但不替代灯光阴影、全局光照或材质 AO。

## 1. 确认兼容性

| 项目 | 要求与验证范围 |
| --- | --- |
| 平台 | **Windows 64 位（Win64）**。插件描述符和模块均限定为 Win64。 |
| 引擎 | 按 UE 次版本选择 `Cortao_5.6`、`Cortao_5.7` 或 `Cortao_5.8`。对应已测试的补丁版本为 **5.6.1、5.7.4、5.8.4**。对于其他补丁或自定义引擎构建，请使用目标引擎编译并验证；只改描述符不能解决二进制或 Renderer 私有接口兼容问题。 |
| 渲染路径 | 桌面**延迟渲染**，需提供场景颜色、深度和 GBuffer 法线；要求 **Shader Model 5 或更高**，并保留视图的 Lighting 和 Post Processing。 |
| 已验证的 GPU 组合 | 预编译安装版在 RTX 4070 Ti SUPER 上验证：UE 5.6.1 使用 D3D11/SM5，UE 5.7.4 和 5.8.4 使用 D3D12/SM6。这里列的是具体测试组合，不代表所有引擎/RHI 组合、GPU 或打包后的 Shipping 构建均已测试。 |
| 不支持的视图 | 前向渲染、移动渲染、低于 SM5 的特性级别、立体/VR 视图、场景捕获（包括 `SceneCapture2D`）、反射捕获及平面反射视图。专用服务器不渲染 CORTAO。 |

打开 **Edit（编辑）→ Project Settings（项目设置）→ Engine → Rendering**，搜索 **Forward Shading**。此插件要求它处于关闭状态；如果项目正在使用前向渲染，先评估切换对其他功能的影响。在 **Project Settings → Platforms → Windows** 中检查 **Default RHI** 以及对应的目标着色器格式。选用 SM5 或以上的格式，并在烘焙（Cook）时包含该格式。修改渲染路径或 RHI 后关闭并重启编辑器，等待着色器编译结束。排查兼容问题时，以启动日志报告的实际 RHI 和着色器平台为准。

## 2. 安装或升级

1. 关闭 Unreal Editor，先备份项目或提交当前修改。检查项目的 `Plugins` 目录及引擎插件安装位置是否已有 `Cortao`。只保留**一份**有效插件，不要把新文件直接合并到旧版目录。
2. **预编译安装版**：把 `Install` 中完整的 `Cortao` 目录复制到 `<项目>/Plugins/`。使用为当前引擎安装构建的二进制，不要混用不同版本的 `Binaries`、`Shaders` 或 `Source`。
3. **精简源码版**：选择 `Cortao_5.6`、`Cortao_5.7` 或 `Cortao_5.8`，将该目录中的内容复制到 `<项目>/Plugins/Cortao/`。源码版不含预编译二进制，需要目标引擎要求的 C++ 工具链，包括对应的 MSVC 和 Windows SDK。如果纯蓝图工程无法在打开时编译插件，改用对应的预编译版，或为该引擎准备 C++ 构建环境；不要借用其他版本的 DLL。
4. 确认描述符位于 `<项目>/Plugins/Cortao/Cortao.uplugin`，且 `Resources`、`Shaders`、`Source` 与它同级。不要拆散着色器和源码目录。
5. 打开项目，在 **Edit → Plugins** 中搜索 **CORTAO Ambient Occlusion**，启用并按提示重启。源码版要成功编译后才能加载。若提示缺少模块或不兼容，先按第 9 节排查。

也可以在项目之外，用目标引擎的 AutomationTool 构建源码版。以下命令在 Windows 命令提示符中运行，替换所有尖括号占位符，输出位置应在插件源码目录之外。构建产物只用于对应引擎；此命令不验证实际画面，也不会替游戏执行 Cook。

```bat
"<UE_ROOT>\Engine\Build\BatchFiles\RunUAT.bat" BuildPlugin -Plugin="<PLUGIN_DIR>\Cortao.uplugin" -Package="<OUTPUT_DIR>" -TargetPlatforms=Win64
```

## 3. 确认第一次渲染效果

1. 打开 **Project Settings → Plugins → CORTAO**。打开 **Enabled**，把 **Quality** 设为 **Medium**、**Output** 设为 **Composite**。初次验证先保留其他默认值。
2. 在普通 **Lit** 透视编辑器或玩家主视图中，观察几米内可见的不透明物体接触边和角落。空天空、超出淡出距离的物体或没有 GBuffer 法线的视图不适合验证。
3. 将 **Output** 改为 **Before / After Comparison**，**Split Position** 设为 `0.5`。左侧是未应用 CORTAO 的场景，右侧是合成结果；项目其他光照和后处理仍在两侧生效。`0` 为整幅应用 CORTAO，`1` 为整幅原画面。编辑器快调面板上的对应标签是 **Before / after**。
4. 如果合成差异不明显，切到 **Ambient Occlusion Only**：白色为中性，越暗代表可见度越低。验证后切回 **Composite**。如果编辑器和游戏结果不同，检查运行时覆盖（第 6–8 节）、区域体积和控制台变量。

### 输出模式

| 模式 | 显示内容 |
| --- | --- |
| **Composite** | 正常场景颜色乘以 CORTAO 可见度，是常规游玩应使用的模式。 |
| **Ambient Occlusion Only** | 带距离淡出的灰度采样可见度，不用于预览合成模式下 **Intensity** 和 **Minimum Visibility** 的效果；这两个值请在合成或前后对比模式下调节。 |
| **Bent Normals** | 以 RGB 编码显示视图空间弯曲法线，用于诊断；不是世界空间法线，也不是材质或光照输出。 |
| **Before / After Comparison** | 左侧原始场景、右侧 CORTAO 合成结果。分界是每个视图各自归一化的位置，通过设置控制，不是可在视口直接拖动的覆盖层。 |

## 4. 调节并保存项目默认值

在未运行 PIE（Play In Editor）时，打开 **Window → CORTAO Quick Tuning**，它列在 Window 菜单的 **CORTAO** 分组中。这是编辑器面板，不是运行时面板。预设载入的是整组起始配置，而非只改一个参数：它会重新启用 AO，并把输出重置为 Composite，同时重设分界、淡出和进阶参数。请先选预设，再微调。**Restore plugin defaults** 会将内置默认值写回项目，而不是只撤销最后一次调整。

改动实时预览。松开滑块或提交数值时保存，预设和步进按钮立即保存；可以直接输入数值，也可用 **− / +** 调节。**AO Strength** 的步进为 `0.001`。**Advanced / fade & noise** 里的淡出滑块采用对数刻度，步进为 `10 cm`；当两端交叉时，另一端随之移动以保持有效区间。保存的默认值属于项目引擎配置（`Config/DefaultEngine.ini`），建议检查配置差异并纳入版本管理。运行时覆盖和关卡中的区域 Actor 不会通过该面板一同保存。

### 常用参数

| 参数 | 默认值 / 范围 | 怎么调 |
| --- | --- | --- |
| **Enabled** | 开启 | 当前 World 的基础值，附近的区域可混合出不同的启用状态。要全局关闭，使用 `r.Cortao.Enable 0`。 |
| **Quality** | Medium / Low 8、Medium 16、High 32 次每像素采样 | 先用 Medium；性能预算紧时试 Low，噪点难以接受时试 High。采样数不等同于帧率。 |
| **Radius** | `100 cm` / `1–1000 cm` | 世界空间采样范围。减小可突出局部接触细节，并抑制大范围暗晕。 |
| **Intensity** / **AO Strength** | `1` / `0–4` | 值越大，合成压暗越明显。`0` 让合成结果中性，但不会关闭诊断输出。 |
| **Minimum Visibility** | `0.05` / `0–1` | 提高可保留暗部细节。`1` 使合成结果中性。 |
| **Screen Radius** | `0.20` / `0.01–0.50` | 按视图高度限制投影采样范围。近处暗晕太宽时可调低。 |
| **Filter** | 开启 | 保留边缘的平滑滤波。排查颗粒或深度/法线边缘渗漏时，可暂时关闭比较。 |
| **Temporal Noise** | 关闭 | 关闭时采样图案稳定，打开后逐帧改变。建议配合 TAA/TSR 并检查镜头运动，CORTAO 不自行累积历史帧。 |
| **Fade Start / Fade End** | `5000 / 10000 cm` | 在这段**视图深度**范围内由完整贡献淡出至中性，End 必须大于 Start。缩短可限制远处伪影，但太近会把需要调节的效果一起淡掉。 |
| **Split Position** | `0.5` / `0–1` | 仅前后对比模式使用，不控制 AO 强度。 |

### 进阶参数

| 参数 | 默认值 / 范围 | 怎么调 |
| --- | --- | --- |
| **Mip Bias** | `-4` / `-8–0` | 调整采样 Mip 选择。除非要排查可复现的伪影，否则保持默认；在目标渲染分辨率下比较。 |
| **Slope Compensation** | `4` / `1–8` | 调节倾斜面补偿。排查斜面异常时独立调整。 |
| **Backface Threshold / Boost** | `0.5 / 0` / 都是 `0–1` | 调节背面贡献的进阶参数，没有具体需求时保持默认。 |
| **Filter Depth Sensitivity** | `512` / `1–2048` | 越高越限制滤波跨越深度断层。开启 Filter 时有效。 |
| **Filter Normal Sensitivity** | `128` / `0–512` | 越高越限制滤波跨越法线转折。开启 Filter 时有效。 |

**预设怎么选：** **Balanced** 适合起步。**Character Soft** 和 **Face and Hair Soft** 采用更短、更柔和的遮蔽以保留细节，但仍影响整个视图。**Fine Detail** 偏向短半径、高质量。**Contact Shadows** 强化接触处，但不会生成定向灯光阴影。**Interior** 和 **Exterior** 针对封闭和开阔环境调整作用范围与淡出。**Performance** 使用 Low 且关闭边缘滤波，要检查噪点和边缘。**Cinematic** 使用 High，需在目标分辨率下评估开销。

**建议顺序：**先选预设并使用正常的拍摄距离对比，再设 Radius、Screen Radius，最后调整 AO Strength 和 Minimum Visibility。检查多个距离和镜头运动后，再决定是否改淡出、滤波或质量。一次只改一组参数，对比时固定曝光、内部渲染分辨率（Screen Percentage）、镜头和其他光照设置。若项目已使用 Unreal AO 或 Lumen 的屏幕空间遮蔽，分别开关对比叠加结果；CORTAO 不会自动关闭其他效果。

## 5. 用区域体积配置镜头经过时的效果

1. 在关卡编辑器的 **Place Actors** 搜索 **CORTAO Region Volume** 并放入关卡。使用 Actor 的变换和 **Bounds** 组件调整盒子的位置、方向和大小。这个 Actor 不是碰撞触发器。
2. 在 Details 中先选 **Preset**。如需逐项控制，开启 **Use Custom Options**，填写 **Custom Options**。自定义参数是独立的一组完整值；改 Preset 不会自动更新自定义参数。
3. 设置 **Blend Distance**，单位厘米，默认 `300`。盒内为满权重，离开盒子后在此距离内平滑降至零，超过后不影响画面。`0` 为硬边界。权重取决于**相机位置**，而不是角色重叠或像素对应物体的位置。
4. 设置 **Priority**，默认 `0`。高优先级后应用，在覆盖权重较大时可压过低优先级。相同优先级重叠时，数值参数按权重混合；Quality、Filter、Temporal Noise 等离散值是切换而非连续插值，尽量避免在镜头过渡处设置冲突。
5. 让测试相机缓慢穿过混合带和盒子边界。区域改变的是**整幅画面**，包括盒外物体；输出模式和对比分界仍由 World/控制台控制。关闭区域的 Enabled 会向无效果过渡；开启的区域也能在 World 基础值关闭时局部打开 AO。全局关闭开关仍然有效。

保存关卡来保存区域的位置和属性。运行时可修改实际渲染 World 中的区域 Actor。流送载入或卸载 Actor 时，参与混合的区域可能变化，建议沿实际镜头路径验证。

## 6. 用蓝图控制 CORTAO

在**目标 World** 的关卡蓝图或本地玩家蓝图里获取 **Cortao World Subsystem**：使用 **Get World Subsystem**，Class 选择 `CortaoWorldSubsystem`，或添加节点菜单中对应的类型化获取节点。从返回的子系统引用拖线搜索以下节点，连接 **Target**。用 **BeginPlay** 设置初始值，用明确的设置事件或输入事件更改，不必每 Tick 调用。多人游戏请在实际渲染画面的客户端执行；这些运行时设置不会自动复制。每个 PIE 实例和 World 都有自己的覆盖层。

| 节点 | 引脚与作用 |
| --- | --- |
| **Set CORTAO Enabled** | 接执行线、子系统 Target 和布尔值 **Enabled**。保留当前质量、模式和外观参数。 |
| **Is CORTAO Enabled** | 纯节点，读取 World 基础启用值，不包含区域混合、渲染兼容性或全局开关。 |
| **Set CORTAO Output Mode** | 接执行线、Target 和 **Output Mode**。保留启用值和其他参数，切换模式不会顺便打开 AO。 |
| **Get Options** | 纯节点，返回蓝图覆盖、运行时调试或项目默认值中的当前**基础**配置，而不是按相机混合后的配置。 |
| **Set CORTAO Options** | 接执行线、Target 和 **Options**（`Cortao Options` 结构体）。整组设置替换 World 的覆盖值。 |
| **Has CORTAO World Override** / **Has CORTAO Blueprint Override** | 纯节点。前者表示是否存在蓝图或运行时调试覆盖，后者只检测蓝图覆盖，均不检测区域混合。 |
| **Reset To Project Settings** | 接执行线和 Target。同时清除这个 World 的**蓝图覆盖和运行时调试覆盖**；区域和全局命令仍有效。 |

**只改少量字段：**将 `Get Options → Return Value` 接到 `Set Members in Cortao Options → Struct Ref`（若需要可写引用，先把返回值存成局部变量）。在节点 Details 中只勾出要修改的字段，例如 **Radius** 和 **Intensity**。执行流经过 **Set Members** 再进入 **Set CORTAO Options**，把修改后的结构体接到 **Options**。不要在想保留其他值时从 **Make Cortao Options** 新建结构体，否则未设置的字段可能回到内置默认值。

**优先级：**基础配置按 **蓝图覆盖 > 运行时调试覆盖 > 项目设置** 选取，再按相机叠加区域混合。全局控制台覆盖和渲染资格判断独立生效，例如 `Set CORTAO Enabled(true)` 不能在前向渲染中强行启用，也不能覆盖 `r.Cortao.Enable 0`。运行时覆盖仅在内存中生效；若希望跨 World 或应用重启保留玩家偏好，请由项目自行保存，并在新 World 重新应用。

## 7. 使用运行时快调面板

1. 启动 **PIE**、**Standalone** 或带本地 PlayerController 的 Development 游戏，在控制台输入 `Cortao.Debug`。它切换当前 World 中本地控制器的面板；未进入游戏时请用编辑器面板。
2. 若要绑定自己的调试输入，调用 **Toggle CORTAO Quick Panel**、**Open CORTAO Quick Panel** 或 **Close CORTAO Quick Panel**，并把本地控制器接到 **Player Controller**。插件没有用于全局打开面板的 F7 绑定；当面板有焦点时，**Escape** 或 **F7** 可关闭。
3. 修改会写入当前 World 的临时调试覆盖，关闭面板不等于清除。**Reset to project** 只清除调试层。蓝图接管时运行时控件会锁定，面板重置也不会清除蓝图；需要恢复项目默认值时调用蓝图重置节点。
4. 打开面板会切到 **UI Only** 并显示鼠标。关闭时恢复 **Game Only** 和之前的鼠标可见性，而不是恢复任意自定义输入模式。若项目使用 **Game and UI** 或已有菜单栈，建议仅在调试时开放面板，并在使用后显式恢复项目输入状态。不要假设 Shipping 玩家能够使用控制台或调试快捷键。

## 8. 控制台命令

| 命令 | 值与作用 |
| --- | --- |
| `r.Cortao.Enable` | `0` 全局关闭；`1` 按正常 World/区域规则运行，默认 `1`。 |
| `r.Cortao.Output` | `-1` 跟随 World（默认）；`0` 合成、`1` 仅 AO、`2` 弯曲法线、`3` 前后对比。它不绕过启用检查，对比分界仍读取 World 的 Split Position。 |
| `Cortao.Debug` | 切换运行时快调面板，需要 PIE 或游戏 World 和本地 PlayerController。 |

在编辑器或游戏控制台输入；只输 CVar 名称可检查当前值。调试结束后恢复 `r.Cortao.Enable 1` 和 `r.Cortao.Output -1`。两个 CVar 是进程级的，同一编辑器进程里的多个 PIE 视图都可能受影响，不能替代按 World 控制的蓝图逻辑。若每次启动时变量被改回，检查项目配置和启动参数。

## 9. 打包、验收与排查

- **打包前**：确认项目启用了正确版本的插件，目标为 Win64，选中了预期的桌面 RHI 和着色器格式，并会 Cook 要测试的地图。保存项目默认值，并在正常游玩时使用 Composite。清除启动诊断设置，不要把灰度或对比模式保存为正式游戏的默认输出。
- **构建和烘焙**：使用项目对应的引擎。若从源码构建，按项目需要的配置编译插件；只有 Editor DLL 不能代替游戏目标。Cook 阶段保留完整着色器树，由引擎处理 staging。遇到失败先找构建或 Cook 日志中的第一条具体错误，不要只看结尾的通用失败提示。
- **运行产物**：先启动打包后的 Win64 Development，记录实际 RHI，检查首张地图、移动相机、区域边界、设置重置，以及调试面板关闭后的输入状态。再在代表性硬件上测试预期的 Shipping 构建，不依赖编辑器或控制台。用项目设置或本地玩法逻辑，在相同视角下对比 CORTAO 开关。用引擎 GPU 分析工具按预期内部渲染分辨率测量；耗时取决于视图尺寸、质量、滤波、硬件及其余渲染负载。

| 现象 | 排查方法 |
| --- | --- |
| 插件未出现在列表，或提示缺模块 | 确认 `.uplugin` 在项目 `Plugins` 树下，没有与另一份混用；检查 Win64 和引擎构建是否匹配。源码版要确认 C++ 工具链并重建；只改 `EngineVersion` 不会重新编译模块。 |
| 所有输出都看不到效果 | 确认插件已启用且重启。使用具有 Lighting、Post Processing 和有效 GBuffer 的延迟渲染 Lit 主视图；检查 `r.Cortao.Enable`、World/区域启用状态、模式及分界。观察尚未淡出的物体并检查 AO Only。 |
| 只有捕获、反射或 VR 视图没效果 | 先确认普通主视图可用。这些捕获、反射、立体视图会被主动跳过。 |
| 改项目设置或快调面板没反应 | 查 World 覆盖、附近区域体积及 `r.Cortao.Output`。编辑器面板改的是默认值，蓝图覆盖优先于运行时调节。 |
| 颗粒、暗晕或过暗 | 暗晕先降 Radius/Screen Radius；过暗先降 Intensity 或升 Minimum Visibility，再比较滤波灵敏度和 Quality。排查时先关 Temporal Noise，开回去后结合 TAA/TSR 检查镜头运动。 |
| 到画面边缘或几何出画时 AO 变化 | 屏幕空间算法看不到出画或被挡住的几何。减小半径或强度并检查完整镜头路径；提高质量不能补回缺失的场景数据。 |
| 自发光、半透明、皮肤或毛发暗了 | 插件乘的是场景颜色，不只影响间接光。已合成的颜色可能被压暗；不写深度和法线的材质不一定能充当遮挡物。试柔和预设、缩小半径、提高 Minimum Visibility 或按场景关闭；本版本没有材质/物体遮罩，也不会单独保护皮肤。 |
| 着色器错误或只在打包后异常 | 查看日志中第一条 CORTAO/Global Shader 错误和启动 RHI。确保构建/Cook 时 `Shaders` 完整，目标着色器格式已 Cook，且引擎和插件版本一致。替换不匹配文件后重新构建及 Cook。 |
| 运行时面板打不开，或关闭后输入模式改变 | 在 PIE/Standalone/Game 中使用本地 PlayerController，并把它接到面板节点。部分构建没有控制台；若项目所需输入模式不是 Game Only，关闭面板后自行恢复。 |
