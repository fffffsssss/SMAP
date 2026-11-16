# SMAP 架构与开发指南

> 本文面向两类读者：需要理解代码并完成修改的 AI，以及需要维护、扩展和复现分析流程的开发者。内容以当前仓库代码为准；SMAP 是一个以 MATLAB 为实现语言的 Superresolution Microscopy Analysis Platform，重点服务 SMLM（单分子定位显微）数据。

## 1. 一句话理解 SMAP

SMAP 把“数据容器、参数系统、GUI 模块、插件和工作流引擎”组合成一个共享状态的桌面分析平台：文件加载器把图像或定位结果转换为统一数据结构，Localize 工作流完成逐帧峰值检测与拟合，Render/Process/Analyze/ROIs 插件在同一个定位数据对象上继续做渲染、校正、配准、统计和结构建模。

核心设计取向：

- **模块化**：功能以 MATLAB class/plugin 形式存在，按目录层级自动发现。
- **共享状态**：所有 GUI 和插件通常共享同一个 `interfaces.ParameterData`（全局参数）与 `interfaces.LocalizationData`（定位数据）。
- **可编排**：定位不是写死在主 GUI 中，而是由 `interfaces.Workflow` 组装 `interfaces.WorkflowModule` 节点。
- **GUI 与算法分离**：`GuiModuleInterface`/`DialogProcessor` 负责参数、控件、结果和历史；插件类负责具体算法。
- **兼容科研脚本**：除了面向 GUI 的类，也保留大量 `shared/`、`standalone/`、`other/` 和插件私有函数，便于脚本调用及复用旧算法。

## 2. 启动入口与顶层 GUI

启动脚本是 [`SMAP.m`](SMAP.m)：

1. 将 `shared` 加入 MATLAB path。
2. 清理旧的全局 GUI 对象（变量 `g`）。
3. 实例化 `gui.GuiMainSMAP` 并调用 `makeGui`。
4. 在非部署环境中打印 `git status`，便于记录版本状态。

主窗口 [`+gui/GuiMainSMAP.m`](+gui/GuiMainSMAP.m) 创建一个 tab group，并挂载六个主要工作区：

| Tab | 实现 | 主要职责 |
|---|---|---|
| File | `gui.GuiFile` | 加载/保存图像、定位、设置和站点数据；文件列表、分组、自动保存 |
| Localize | `gui.GuiLocalize` | 选择/编辑/运行定位工作流，预览单帧，批处理 |
| Render | `gui.GuiRender` | 多 layer 渲染、视图格式、滤波和显示刷新 |
| Process | `gui.GuiPluginWindow` | 调用 Process 类插件，如漂移校正、配准、通道赋值、字段变换 |
| Analyze | `gui.GuiPluginWindow` | 调用 Analyze 类插件，如分辨率、聚类、3D、追踪和统计 |
| ROIs | `siteexplorer.SiteExplorer`/相关 GUI | 站点/ROI 管理、分割、逐站点评估和汇总 |

`GuiMainSMAP` 初始化全局参数、设置目录、帮助目录和插件菜单；加载后通常调用 [`shared/initGuiAfterLoad.m`](shared/initGuiAfterLoad.m)，完成 regroup、filter、视野参数、文件列表、相机像素尺寸和 Site Explorer 同步。

## 3. 核心对象模型

### 3.1 全局参数：`interfaces.ParameterData`

[`+interfaces/ParameterData.m`](+interfaces/ParameterData.m) 是所有模块共享的参数对象，主要字段：

- `par`：参数名到参数记录的映射；记录通常包含 `content`、GUI 状态、同步信息和所属对象。
- `globalSettings`：全局安装/用户设置。
- `results`：模块结果集合。
- `autoresults`：按插件路径嵌套保存的自动结果。
- 事件 `sr_render`、`sr_display`、`backup4undo`：驱动渲染刷新和撤销备份。

模块通过 `ParameterInterface` 的 `attachPar/getPar/setPar/getAllParameters/setResults` 访问它。不要在插件中私自复制一份全局参数；应附着现有 `P`。

### 3.2 定位数据：`interfaces.LocalizationData`

[`+interfaces/LocalizationData.m`](+interfaces/LocalizationData.m) 是 SMAP 的中心数据模型，典型字段包括：

- `loc`：未分组定位；每个字段是长度相同的向量，例如 `frame`、`xnm`、`ynm`、`znm`、`phot`、`locprecnm`、`channel`、`filenumber`。
- `grouploc`：按时间/空间规则连接后的分组定位；为性能缓存，不是每次动态计算。
- `layer`：层定义、筛选条件和分组筛选。
- `files.file`：输入文件及 `metadataSMAP` 信息。
- `SE`：关联的 Site Explorer 对象。
- `history`：处理历史和参数快照。

常用生命周期：

1. `clear` 初始化空数据和 Site Explorer。
2. `addfile` 注册文件，并初始化最小定位字段和元数据。
3. Loader 使用 `setloc/addloc` 写入定位字段。
4. `regroup` 生成/更新 `grouploc`。
5. `filter` 根据 layer 计算可见/可用定位。
6. `getloc` 按字段、文件、channel、layer、ROI/FOV 等条件取子集。
7. Process/Analyze 插件修改字段、附加结果或记录 history。
8. Saver 调用 `savesml` 或 CSV/HDF5/TIFF 等导出路径持久化。

字段一致性是关键约束：所有定位字段原则上都应与 `loc.frame` 等长；新增字段使用 `setloc`，部分定位更新时传入索引并让对象负责补齐/重新分组。直接拼接 struct 容易造成字段长度不一致，应谨慎。

### 3.3 图像与元数据

[`+interfaces/imageloaderSMAP.m`](+interfaces/imageloaderSMAP.m) 定义统一图像加载器接口，抽象方法为 `openi`、`getimagei`、`getmetadatatagsi`；支持普通读取、在线读取和多通道组合。

[`+interfaces/metadataSMAP.m`](+interfaces/metadataSMAP.m) 统一描述相机/图像元数据：宽高、ROI、相机像素尺寸、曝光、EM gain、conversion、offset、帧数、sCMOS 修正文件和 Bio-Formats/OME 元信息等。图像坐标通常以相机像素表示，定位坐标通常转换为 nm。

## 4. GUI 抽象层

GUI 类按继承层次组织：

```text
ParameterInterface
  └─ GuiParameterInterface
       └─ GuiModuleInterface
            └─ DialogProcessor
                 └─ LayerInterface
```

- `ParameterInterface`：参数读写、输入/输出参数声明、结果存取。
- `GuiParameterInterface`：把 MATLAB 控件映射到结构化 GUI 参数，支持保存/恢复和子模块递归同步。
- `GuiModuleInterface`：模块句柄、`guihandles`、`children`、插件信息、帮助、跨平台外观。
- `DialogProcessor`：标准 Info/Run/Show results 控件、结果窗口、`processgo`、history 和 undo 入口。
- `LayerInterface`：在 DialogProcessor 上增加 layer 前缀和层级参数约定。
- `Renderer`：定义 `render(locD,p)` 到 `renderfunction(locs,p)` 的渲染接口。
- `LocDataInterface`：让模块共享 `locData`。

插件通常实现以下方法：

- `guidef`：声明 GUI 参数和控件。
- `initGui`/`makeGui`/`updateGui`：创建和更新控件。
- `run` 或 `process`：执行算法。
- `prerun`：运行前准备、缓存和外部资源初始化。
- `getData`/`setData`/`preview` 等：按插件需要提供。

建议继承最贴近职责的基类，而不是从头创建 figure。GUI 子模块放进父对象的 `children`，这样参数保存、外观调整和销毁可以递归完成。

## 5. 插件系统

插件位于 [`plugins/`](plugins/) 下，用 MATLAB package 目录表示菜单路径：

```text
plugins/+File/+Load/...
plugins/+File/+Save/...
plugins/+Process/{+Assign2C,+Drift,+Images,+Modify,+Register}/...
plugins/+Analyze/{+calibrate,+cluster,+counting,+measure,+sr3D,+tracking,+render,...}/...
plugins/+ROIManager/{+Segment,+Analyze,+Evaluate}/...
plugins/+WorkflowModules/{+Loaders,+Peakfinders,+Filters,+Fitters,+LocProcessors,+IntensityCalculator,+Structure}/...
```

`+` 是 MATLAB package 语法，不是普通文件名前缀。GUI 使用 `pluginnames` 枚举目录，再由 [`+gui/private/makeplugincallfile.m`](+gui/private/makeplugincallfile.m) 生成插件调用入口；运行时通常通过 `plugin(path1,path2,path3,...)` 实例化。插件帮助来自 `Documentation/help`，插件元信息保存在 `plugininfo`/`info` 结构中。

插件分类的语义：

- **File**：外部格式适配，不应把特定格式逻辑散落到主 GUI。
- **Process**：对现有定位/图像做可重复变换，常写回 `locData` 或产生结果。
- **Analyze**：只读或派生分析，通常在结果窗口显示统计、图像或曲线。
- **ROIManager**：围绕 Site Explorer 的分割、站点管理、逐站点分析和模型拟合。
- **WorkflowModules**：可连接的数据流节点，服务 Localize 和批处理。
- `plugins/shared`：插件间共享的算法、渲染、注册、拟合、导出和帮助函数。

新增插件的最低要求通常是：放入正确的 package 路径；继承合适接口；实现 `guidef` 与运行方法；声明 `inputParameters/outputParameters`；提供 `plugininfo`/帮助文本；用已有 `GuiModuleInterface` API 读写参数和定位数据。

## 6. SMLM 定位数据处理主链路

### 6.1 典型离线流程

```text
图像/定位文件
  → File Loader
  → LocalizationData.loc + files.file + metadataSMAP
  → layer/filter/regroup
  → Render / Process / Analyze / ROIs
  → Saver / 结果文件 / 报告
```

### 6.2 图像到定位（Localize）

Localize GUI 从 `settings/workflows/*.txt` 或 `.mat` 载入工作流。工作流常见节点顺序：

1. **Loaders**：`TifLoader`、`TifLoaderParallel`、`Loader4PiMat`、`CameraConverter`、模拟图像等。
2. **Peakfinders**：ROI 切割、背景估计、峰值检测、DeepSMLM/NMS 等。
3. **Filters**：小波背景、median background、图像滤波和归一化。
4. **Fitters**：径向对称、MLE、全局 spline、4Pi、GPU/Yiming、DECODE、FalconHD 等。
5. **IntensityCalculator/LocProcessors**：强度提取、定位收集、过滤、预览、自动保存。
6. **Structure**：splitter/combiner，用于多通道、多输入和分支合并。

工作流使用 `interfaces.Workflow` 保存模块列表、标签、输入连接和参数；普通 DialogProcessor 插件可以由 `interfaces.plugin4workflow` 包装成工作流节点。

### 6.3 工作流运行机制

`interfaces.WorkflowInterface` 提供轻量数据流图：

- 节点通过 `setInputModule`、`addNextModule` 建立边。
- 节点通过 `inputChannels/outputChannels` 声明输入/输出端口。
- `initialize` 收集参数并执行 `prerun`。
- 单输入直接调用 `run(data,p)`。
- 多输入使用 `SyncBuffer` 按 `frame` 或 `ID` 同步，完整后再调用 `run`。
- `output(data,channel)` 把结果推给下游节点。
- `WorkflowData` 携带 `data/frame/ID/numberInTag/eof` 等运行时信息。

`WorkflowBuffer`/`SyncBuffer` 负责缓存和多路对齐；批处理入口是 `plugins/+WorkflowModules/Batchprocessor.m`，可重复运行工作流处理目录中的多文件。

### 6.4 定位质量与后处理

处理插件覆盖：漂移校正（bead/DME/Picasso/XYZ）、多色配准和仿射变换、通道强度赋值、背景/相机校正、坐标变换、分组/解组、空间过滤、字段数学运算、NND/聚类、FRC/decorr 分辨率、3D 标定与显示、SPT/轨迹分析等。

## 7. Render 与 Layer 设计

`gui.GuiRender` 创建多 layer 标签页，每个 layer 由 `GuiChannel` 管理。`GuiChannel` 把 layer 参数、文件/通道选择、过滤、强度归一化、颜色范围和 renderer 选择组合起来；渲染器通常实现 `interfaces.Renderer`。

典型刷新分为两种：

- `P` 触发 `sr_render`：重新计算图像。
- `P` 触发 `sr_display`：不重新计算，仅切换可见层、颜色或视图。

渲染输入应优先通过 `LocalizationData.getloc` 获取，遵守当前 FOV、layer、channel 和过滤状态；不要绕过过滤直接读取整个 `loc`，除非插件明确需要原始数据。

## 8. ROI / Site Explorer

Site Explorer 相关接口位于 [`+interfaces/SiteExplorer.m`](+interfaces/SiteExplorer.m)、`SEsites.m` 和 [`+siteexplorer/`](+siteexplorer/)，GUI 位于 `+gui/SE*.m`。其职责是把定位集合组织成 sites/ROIs/cells，并提供：

- 分割与站点导入/导出。
- 站点筛选、排序、注释和质量控制。
- 逐站点 Evaluate 插件。
- AverageSites、NPC/CME/fibril 等领域分析。
- LocMoFit/SMLMModelFit 等模型拟合入口。

继承 `interfaces.SEProcessor` 或 `interfaces.SEEvaluationProcessor` 的插件可以直接访问 `locData.SE` 和站点上下文。此层与普通 Analyze 的区别是：计算单位不再是整个定位表，而是 site/cell/ROI 子集。

## 9. LocMoFit 与外部/高性能后端

`LocMoFit/` 是相对独立的模型拟合子系统，包含：

- `@LocMoFit`：拟合控制、坐标/图像转换、结果和绘图。
- `@SMLMModel`、`@imageModel`、`@geometricModel`、`@parametricModel`：模型类型。
- `helperClass`、`helperFcn`、`external`：管理器、优化、插值、聚类和数值工具。
- `template/`、`script/`、`rstGeneration/`：模型模板、示例和文档生成。

高性能或平台相关代码散落在：

- `shared/` 和 `plugins/shared/`：通用算法、渲染、数据读写、数学兼容函数。
- `shared/myMATLABfunctions/`：对 MATLAB 版本差异的兼容实现。
- `standalone/driftcorrection3D/`：可独立使用的漂移校正实现，含 C/MEX。
- `fit3Dcspline/`：3D spline 标定辅助。
- `PSFAnalyses/`：PSF/FWHM 分析。
- `.mexw64`、`.mexmaci64`、`.mexa64`、`.c/.cpp`：平台相关加速代码；修改后需分别编译并保留 CPU fallback。

`shared/adddirectoriesforcompile.m` 说明了部署/编译时的额外路径和外部依赖（Micro-Manager、Bio-Formats、Fiji/u-track、GPU/4Pi/VectorPSF 等）。不要把这些可选依赖当作所有环境都存在；插件应在缺依赖时给出明确错误或回退路径。

## 10. 数据、设置与持久化约定

- `settings/`：全局设置、相机标定、工作流配置、CSV 字段映射。
- `settings/workflows/`：推荐的定位工作流（`.txt` 可读配置，`.mat` 参数快照）。
- `Documentation/Global_parameters_structure_names.md`、`ParameterNames.txt`、`LocData_fields.txt`：参数与定位字段参考。
- `shared/savesml.m`、`plugins/+File/+Save/`：SML、CSV、Picasso HDF5、TIFF、站点和设置导出。
- 自动保存通常写入 `settings/temp/autosave_sml.mat` 一类路径。
- `history` 应记录会改变定位数据的处理及其 GUI 参数，便于审计和复现。

## 11. 开发与修改指南（给 AI 和开发者）

### 11.1 先判断你要改哪一层

1. 入口/布局问题：看 `+gui/GuiMainSMAP.m` 及对应 `Gui*.m`。
2. 参数/控件保存问题：看 `+interfaces/ParameterInterface.m`、`GuiParameterInterface.m`。
3. 数据字段、筛选、分组、保存问题：看 `LocalizationData.m`。
4. 图像到定位流程问题：看 `Workflow*.m` 和 `plugins/+WorkflowModules/`。
5. 单步后处理：看 `plugins/+Process/`。
6. 统计/可视化：看 `plugins/+Analyze/`。
7. 站点/ROI：看 `+interfaces/SiteExplorer.m`、`+gui/SE*.m`、`plugins/+ROIManager/`。
8. 模型拟合：看 `LocMoFit/` 或 `SMLMModel*` 相关插件。

### 11.2 编写插件时保持的契约

- 使用 package 路径表达菜单分类，不要把插件直接塞进根目录。
- 通过 `getPar/setPar` 访问共享参数；声明 `inputParameters/outputParameters`。
- 通过 `attachLocData` 使用共享定位对象；不要复制出隐藏的第二份状态。
- 修改定位字段优先使用 `setloc`、`addloc`，并考虑 `regroup/filter/history`。
- GUI 控件句柄必须登记在 `guihandles`；子模块必须登记在 `children`。
- 工作流节点实现 `run(data,p)`，多输入节点明确 `inputChannels` 与同步模式。
- 结果写入结果窗口或 `P.results/P.autoresults`，不要只依赖 `disp`。
- 对外部库、GPU 和 MEX 提供检测与 CPU/纯 MATLAB fallback。
- 遵守现有 MATLAB 版本兼容性（README 要求 MATLAB 2019a+）。

### 11.3 调试顺序

建议从最小闭环开始：

1. 运行 `SMAP.m`，确认设置目录和插件菜单生成成功。
2. 加载一个小的定位文件或 TIFF，检查 `locData.loc`、`files.file` 和 `metadataSMAP`。
3. 检查 `regroup`/`filter` 后的定位数量及关键字段长度。
4. 单独运行插件的 `processgo` 或工作流的预览，再运行批处理。
5. 对 GUI 问题检查 `P.par`、`guihandles`、`children` 和事件监听器。
6. 对结果问题检查 `P.results`、history、导出文件和当前 layer/FOV 是否一致。

当前仓库未见统一的自动化测试入口；修改高风险数值代码时，应使用小型示例数据、已有 workflow 和手工结果对照验证。

## 12. 给 AI 的快速上下文

处理本仓库任务时，默认假设：

- 这是 MATLAB 项目，不要按 Python/JavaScript 项目习惯改造目录。
- `+Name` 目录是 package；`@Class` 目录是旧式 MATLAB class folder。
- `P` 和 `locData` 是跨模块共享的核心状态。
- “插件”通常意味着一个继承接口的 class，而不只是一个独立函数。
- 任何定位数据修改都要考虑字段长度、grouped/ungrouped、layer/filter、history 和保存兼容性。
- 任何 GUI 修改都要考虑跨平台尺寸、`guihandles` 登记、子模块递归保存以及事件刷新。
- 优先复用 `shared/`、`plugins/shared/` 和现有接口，不要重复实现已有的读写、渲染、参数和同步逻辑。

## 13. 参考入口

- 启动：[`SMAP.m`](SMAP.m)
- 主 GUI：[`+gui/GuiMainSMAP.m`](+gui/GuiMainSMAP.m)
- 定位数据：[`+interfaces/LocalizationData.m`](+interfaces/LocalizationData.m)
- 参数系统：[`+interfaces/ParameterData.m`](+interfaces/ParameterData.m)、[`+interfaces/ParameterInterface.m`](+interfaces/ParameterInterface.m)
- GUI 基类：[`+interfaces/GuiModuleInterface.m`](+interfaces/GuiModuleInterface.m)、[`+interfaces/DialogProcessor.m`](+interfaces/DialogProcessor.m)
- 工作流：[`+interfaces/Workflow.m`](+interfaces/Workflow.m)、[`+interfaces/WorkflowInterface.m`](+interfaces/WorkflowInterface.m)、[`+interfaces/WorkflowModule.m`](+interfaces/WorkflowModule.m)
- 图像加载：[`+interfaces/imageloaderSMAP.m`](+interfaces/imageloaderSMAP.m)
- 加载后同步：[`shared/initGuiAfterLoad.m`](shared/initGuiAfterLoad.m)
- 插件发现：[`+gui/private/makeplugincallfile.m`](+gui/private/makeplugincallfile.m)
- 用户文档：[`README.md`](README.md)、[`Documentation/`](Documentation/)

