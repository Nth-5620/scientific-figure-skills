# scientific-figure-skills

计算化学 / 材料学**论文级科学绘图**的三个技能,遵循通用的 Agent Skills 规范,适用于
任意支持该规范的 agent / harness。它们从作者的一个计算化学论文绘图工作流中固化而来
(具体研究体系已脱敏)。所有规则与数值均带实测依据,可直接迁移到其他体系。

技能正文为英文(含中文触发词),按 Agent Skills 渐进式披露规范组织:每个技能一个
目录,`SKILL.md` 为入口,`references/` 按需深入阅读。

## 三个技能

| 技能 | 职责 | 典型触发场景 |
|---|---|---|
| `mol-framework-figure` | Blender 分子框架渲染:逐原子球棍 / 管式模型、逐元素配色与金属材质、斜射三点布光、无缝纯白背景、Freestyle 描边(半透明覆盖下不破)、系列共享相机、Cycles 渲染与数值 QA | 球棍模型、分子框架 / 骨架渲染、团簇、金属取代系列、ball-and-stick |
| `isosurface-figure` | 标量场等值面:cube 文件解析(轴序 / bohr–Å 陷阱)、marching-cubes 提取、Taubin 平滑、按场类型选等值面值与配色、Fresnel 半透明 lobe 材质、值映射面(ESP 类)、与框架场景的正确合成 | EDD / 差分电荷密度、HOMO/LUMO 轨道图、IGMH/IGM/NCI/RDG 弱相互作用、静电势面、.cub 渲染 |
| `curve-panel-figures` | matplotlib 二维曲线图:渲染图 + 曲线按实测对齐的 panel 排版、多体系叠加图与图例、Times New Roman 加粗、按参考图实测比例复刻、对交付 PNG 逐像素验证 | 曲线图、叠加图 / overlay、图例、plane-integral / CDC 曲线、像素级排版 |

三者关系:`mol-framework-figure` 产出不透明框架场景 → `isosurface-figure` 把
等值面 lobe 加进同一场景 → `curve-panel-figures` 把渲染 PNG 与数据曲线拼成成图。

## 安装

三个技能**必须互为兄弟目录**(技能间交叉引用写作 `../<skill>/references/...`)。
支持 Agent Skills 的 agent 会在 `~/.agents/skills/`(用户级)或 `<项目>/.agents/skills/`
(项目级)等标准技能目录下发现它们;各 harness 的具体目录约定以其自身文档为准。

```bash
git clone https://github.com/Nth-5620/scientific-figure-skills.git
mkdir -p ~/.agents/skills
cp -r scientific-figure-skills/{mol-framework-figure,isosurface-figure,curve-panel-figures} ~/.agents/skills/
```

只装部分技能也可以:`curve-panel-figures` 可独立使用;`isosurface-figure` 依赖
`mol-framework-figure` 提供场景,建议成对安装。

## 依赖

### 1. Blender(两个 3D 技能必需)

| 项 | 说明 |
|---|---|
| Blender | **5.x 系列**(实测 5.2.1 LTS)。技能内置了 5.x API 变更说明(组合器改为节点组、`Specular IOR Level` 等输入名、`blend_method` 移除),见 `mol-framework-figure/references/environment.md` |
| 渲染器 | Cycles;`view_transform = "Standard"`(AgX 会把纯白压灰) |
| GPU(可选但推荐) | 实测环境为 OptiX + RTX 4070 Laptop:2400×2400 / 640 spp 约 10 s/帧;启动后首次渲染含 kernel 编译(~1 min),不要按第一次计时 |
| Blender 自带 Python | 自带 **numpy、bmesh**;**没有 PIL / scipy / skimage** —— 图像分析与 QA 必须在系统 Python 中做(这是技能里反复强调的硬约束) |

### 2. blender-mcp 联动(建模必需)

建模要求进入用户**打开的 GUI Blender 会话**,通过 blender-mcp 插件驱动:

- Blender 端:安装 [blender-mcp](https://github.com/ahujasid/mcp-for-blender) 插件(下载、
  安装与启动说明见该 GitHub 仓库)并在 3D 视图侧栏启动服务(默认监听 TCP
  `127.0.0.1:9876`);
- Agent 端:在支持 MCP 的 harness 中配置 blender-mcp MCP server(工具
  `execute_blender_code` / `get_scene_info` / `get_viewport_screenshot`),或用任意
  实现该 socket 协议的客户端(发送
  `{"type": "execute_code", "params": {"code": "..."}}`,约 60 行即可自写,协议见
  `environment.md`);
- 无 GUI 的 `blender -b` 仅用于批量渲染与验证,不作为建模交付路径。

### 3. 系统 Python(QA / 图像分析 / 曲线图必需)

Python ≥ 3.10(实测 3.13):

| 包 | 用途 | 哪些技能用 |
|---|---|---|
| numpy | 格点数据、像素 QA | 全部 |
| scipy | 连通域滤波等 | isosurface-figure |
| scikit-image | `marching_cubes(method="lewiner")` | isosurface-figure |
| Pillow | 图像 IO、蒙太奇拼图、像素 QA | 全部 |
| matplotlib | 曲线图绘制 | curve-panel-figures |
| fontTools(可选) | 检查字体字形覆盖(如 `CO₂` 的 `U+2082`) | curve-panel-figures |

字体:**Times New Roman(含 Bold 字面)**——曲线图的硬性约定;缺失时 mathtext 与
加粗会静默回退(坑点见 `curve-panel-figures/references/pitfalls.md`)。

### 4. 数据来源(可选,按需)

技能消费 **Gaussian cube (`.cub`/`.cube`) 格式**的标量场与结构文件:

- **Multiwfn**(推荐):密度差、IGMH/δg、sign(λ₂)ρ、ESP、ELF 等均可导出 cube;
  本仓库解析器的实测对象即其输出(注意 z-fastest 轴序陷阱);
- **CP2K**:`&MO_CUBES` 直接输出轨道 cube(可用 `STRIDE` 控制网格);
- VASP `CHGCAR` / Gaussian `.fchk` / ORCA `.gbw`:先用 Multiwfn 或程序自带工具转
  cube,勿为二进制格式手写解析器。

### 5. 硬件与系统

- Windows(实测;Git Bash)+ NVIDIA GPU(OptiX)为实测路径,CUDA/CPU 亦可但未计时;
- 网格缓存与渲染产物较大(320³ cube 文本约 460 MB),预留磁盘空间。

## 使用

安装后新开 agent 会话即可自动触发,典型说法:

- "把这个 `.cub` 差分电荷密度用 Blender 渲染成论文图"(→ isosurface + framework)
- "画五个体系的球棍模型,共享相机,白底描边"(→ mol-framework-figure)
- "把渲染图和数据曲线拼成左边模型右边曲线的 panel,对齐参考图版式"(→ curve-panel-figures)

也可强制加载:在提供技能指令的 harness 中使用 `/skill` 类指令(如
`/skill isosurface-figure <任务>`)。

## 目录结构

```
scientific-figure-skills/
├── README.md
├── mol-framework-figure/
│   ├── SKILL.md                  # 入口:交付物、10 条 house rules、工作流
│   ├── references/               # 几何/材质光照描边/合成/相机/坑点/验证/环境
│   └── examples/reference-project/     # 完整工作实例(路径为源项目专属)
├── isosurface-figure/
│   ├── SKILL.md
│   ├── references/               # cube 解析/提取/场类型/配色/材质/值映射面/坑点/验证
│   └── examples/                 # reference-project 实例 + v1 历史快照(已过时,仅存档)
└── curve-panel-figures/
    ├── SKILL.md
    └── references/               # 坑点(13 条)/验证阈值
```

## 来源与维护

三个技能从作者的一个计算化学论文绘图工作流中固化而来(具体研究体系已脱敏;完整工作
实例见各技能的 `examples/`,结果数值已移除、绘图参数保留)。来源项目 `script/` 中另有
经过验证的实现脚本与两个校验器(`validate_skill.py` / `validate_skill_claims.py`,逐条
核对技能文本中的数值与代码一致)。在其他项目中使用时,规则与实测数值直接适用;项目
脚本需按 `references/` 中的配方重建。修改参数时,请同步更新技能文本与实现,避免
"文档与代码漂移"。
