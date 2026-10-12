# 百鬼夜行 V2 资源回迁实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 从「百鬼全自动2」(V2) 中提取 V1 缺失或已过时的识别图、规则 JSON 与模型，合并进 V1 的 `tasks/Hyakkiyakou`，且不引入 V2 的任何卡密/授权/注入组件。

**Architecture:** 纯资源迁移 + 声明同步。V1 的 Python 业务逻辑（`script_task.py`、`agent/`、`slave/`）保持不变；只搬运 V2 的 `<图片模板>`、`*.json` 规则、`*.onnx` 模型，并把 `assets.py` 的规则声明补齐/校正。V2 的所有编译产物（`*.pyd`）、授权模块、注入器、启动器、Python 运行时一律不进入 V1。

**Tech Stack:** Python 3.10、OpenCV 模板匹配、onnxruntime、OAS 的 `RuleImage` / `RuleClick` / `RuleOcr` / `RuleSwipe` 原子规则。

---

## 背景

- V1 = `G:\HELP\NEW`（OAS 源码仓库，本计划的**唯一写入目标**）。
- V2 = `G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit`（第三方绿色发行版，**只读来源**）。
- 两者同源：都基于 OnmyojiAutoScript 的百鬼夜行任务。V2 的百鬼逻辑被编译为 `*.pyd` 并叠加了卡密系统与免卡密注入补丁，本次**全部排除**，只取其中可独立复用的资源。

## 纳入范围 / 排除范围

**纳入（本次要迁的东西）**

| 类别 | 具体文件 |
| --- | --- |
| 新增识别图模板 | `hya/has_prob_up.png`、`hya/limit_time_0.png`、`hya/bean_0.png`、`hya/time_over.png`、`hya/gb_buff_1.png`、`hya/damo.png` |
| 替换识别图模板 | `hya/hya_haccess.png`、`hya/hya_hend.png`、`hya/hya_ensure_recall.png` |
| 规则 JSON | `hya/image1.json`、`hya/image5.json`、`hya/click.json` |
| 规则声明 | `tasks/Hyakkiyakou/assets.py` 对应条目 |
| 模型（需过闸门） | `agent/tracker.onnx`、`agent/detect_rarity.onnx` |

**排除（绝不迁移）**

- 授权与注入：`deploy/Identity.cp310-win_amd64.pyd`、`hyakki_license_bypass.py`、`sitecustomize.py`、`injector/**`（含 `HyakkiInjector.exe`、`payload/**`）、`Lib/site-packages/hyakki_injector_bootstrap.pth`。
- 产品外壳：`tasks/Hyakkiyakou/start_main.py`、`tasks/Hyakkiyakou/settings.json`、`百鬼启动器.exe`、`百鬼启动器.bat`。
- 编译业务逻辑：`ghosts.cp310-win_amd64.pyd`、`ghosts_ui.cp310-win_amd64.pyd`、`config.cp310-win_amd64.pyd`、`slave/hya_device.cp310-win_amd64.pyd`、`agent/tracker.cp310-win_amd64.pyd`、`agent/new_labels.cp310-win_amd64.pyd`、`agent/test.cp310-win_amd64.pyd`。
- 混淆脚本：V2 的 `slave/hya_slave.py`、`slave/hya_color.py`（base64+zlib+marshal 壳）。
- 运行时与依赖：V2 的 `Lib/`、`DLLs/`、`python*.dll`、`toolkit/`。

## Global Constraints

- V2 目录全程**只读**，不得就地修改、重命名或删除任何文件。
- 唯一写入目标：`G:\HELP\NEW\tasks\Hyakkiyakou` 及其子目录。
- 不迁移任何 `*.pyd`，不引入任何网络校验、卡密、`check_status` / `validate_secure_key` / `acquire_lock` 相关代码。
- 所有 `RuleImage` 声明必须使用 V1 的参数形式：`file="./tasks/Hyakkiyakou/hya/<name>.png"`；**不得**照抄 V2 的 `path="./tasks/Hyakkiyakou/"` + `file="hya/..."` 写法（V1 的 `module/atom/image.py:RuleImage.__init__` 没有 `path` 形参）。
- 换行符差异不算差异：`hya/ocr.json`、`hya/swipe.json`、`train/click.json`、`train/swipe.json`、`hya/image2.json`、`hya/image3.json`、`hya/image4.json`、`train/image.json` 与 V1 内容完全一致（仅 CRLF→LF），**保持 V1 原样，不做改动**。
- 每个 Task 结束提交一次，失败可单 Task 回滚。

## 差异清单（实测，SHA256 比对结果）

**A. V2 独有、V1 没有的检查点**

| 规则名 | 图片 | roi_front | roi_back | threshold | 含义 |
| --- | --- | --- | --- | --- | --- |
| `I_HAS_PROB_UP` | `has_prob_up.png` | `172,9,96,32` | `157,1,843,59` | 0.8 | 概率 UP |
| `I_LIMIT_TIME_0` | `limit_time_0.png` | `611,15,40,40` | `611,15,40,40` | 0.8 | 倒计时归零 |
| `I_BEAN_0` | `bean_0.png` | `129,642,28,36` | `129,642,28,36` | 0.8 | 豆子已空 |
| `I_TIME_OVER` | `time_over.png` | `650,328,46,59` | `650,328,46,59` | 0.8 | 时间终了 |
| `I_BUFF_1` | `gb_buff_1.png` | `363,32,32,46` | `344,14,128,76` | 0.8 | 庭院左上角加成 |
| `I_CHECK_DAMO` | `damo.png` | `102,23,125,36` | `102,23,125,36` | 0.8 | 达摩武场 |
| `C_HSELECT_TEMP` | —（点击规则） | `0,0,0,0` | `0,0,0,0` | — | 临时变量 |

**B. 两边都有、但 V1 已过时**

| 项 | V1 现状 | V2 新值 |
| --- | --- | --- |
| `I_HACCESS` 模板 | `hya_haccess.png`（19250 B） | `hya_haccess.png`（5761 B，已重截） |
| `I_HACCESS` ROI/阈值 | `(1059,554,100,100)`, 0.8 | `(999,529,202,152)`, 0.7 |
| `I_HEND` 模板 | `hya_hend.png`（36215 B） | `hya_hend.png`（7426 B，已重截），ROI 不变 |
| `I_ENSURE_RECALL` 模板 | `hya_ensure_recall.png`（8624 B） | `hya_ensure_recall.png`（2461 B，已重截），ROI 不变 |
| `C_FRIEND_1` | `(466,227,161,67)` | `(479,275,140,12)` |
| `C_FRIEND_2` | `(738,229,158,64)` | `(738,275,140,12)` |
| `I_FRIEND_SAME_1_RECALL` roiBack | `(114,102,132,94)` | `(154,105,132,94)` |
| `I_FRIEND_SAME_2_RECALL` roiBack | `(107,114,133,80)` | `(152,112,133,80)` |

**C. V2 独有、需评估的模型**

| 文件 | 大小 | 说明 |
| --- | --- | --- |
| `agent/tracker.onnx` | 45,266,024 B | V2 自带的跟踪模型 |
| `agent/detect_rarity.onnx` | 45,248,639 B | V2 自带的稀有度识别模型 |

V1 与 V2 的 `oashya` 包（0.0.7）**完全一致**（`oashya_fp32.onnx` 均为 12,943,564 B，`tracker.cp310-win_amd64.pyd` 均为 490,496 B）。因此 C 类模型只有在 V2 的 `agent/tracker.pyd` 配合下才能被使用，而那是编译业务逻辑、属于排除项。Task 6 必须先用签名比对决定是否落地，**不允许直接覆盖 V1 的 `oashya_fp32.onnx`**。

## Non-Goals（本计划明确不做）

- 不实现使用新检查点的新玩法逻辑（如「倒计时归零改策略」「豆空处理」）。这些判断在 V2 里位于 `ghosts*.pyd`，不可读；本计划只把资源和声明搬进来，保持为**已声明待调用**状态，接线上逻辑另立计划。
- 不迁移、不重写 V2 的 `hya_slave` / `hya_color` / `ghosts` / `ghosts_ui`。
- 不触碰 V1 的授权、调度器、WebUI、其他任务。

---

### Task 1: 冻结 V1 基线并落盘差异清单

**Files:**
- Modify: 无（仅提交与记录）
- Test: `G:\HELP\NEW\docs\superpowers\plans\2026-10-12-hyakkiyakou-v2-resource-migration.md`（本文件）

**Interfaces:**
- Consumes: 无
- Produces: 一个可回滚的 Git 基线提交，后续所有 Task 都从它出发。

- [ ] **Step 1: 确认工作区干净**

Run:
```powershell
git -C G:\HELP\NEW status --short
```
Expected: 若 `tasks/Hyakkiyakou` 下有未提交改动，先自行确认归属，不要覆盖；否则无输出。

- [ ] **Step 2: 打基线标签**

Run:
```powershell
git -C G:\HELP\NEW commit --allow-empty -m "chore(Hyakkiyakou): 资源回迁基线"
git -C G:\HELP\NEW tag hya-baseline-20261012
```
Expected: 标签创建成功，`git tag -l hya-*` 能列出。

- [ ] **Step 3: 把本计划书提交入库**

Run:
```powershell
git -C G:\HELP\NEW add docs/superpowers/plans/2026-10-12-hyakkiyakou-v2-resource-migration.md
git -C G:\HELP\NEW commit -m "docs(Hyakkiyakou): 新增 V2 资源回迁计划"
```
Expected: 提交成功。

---

### Task 2: 迁入 6 张新增识别图模板

**Files:**
- Create: `tasks/Hyakkiyakou/hya/has_prob_up.png`
- Create: `tasks/Hyakkiyakou/hya/limit_time_0.png`
- Create: `tasks/Hyakkiyakou/hya/bean_0.png`
- Create: `tasks/Hyakkiyakou/hya/time_over.png`
- Create: `tasks/Hyakkiyakou/hya/gb_buff_1.png`
- Create: `tasks/Hyakkiyakou/hya/damo.png`

**Interfaces:**
- Consumes: V2 只读目录 `G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\hya\`
- Produces: 6 张图片文件，供 Task 4 的 `assets.py` 声明引用。

- [ ] **Step 1: 复制模板文件**

Run:
```powershell
$src='G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\hya'
$dst='G:\HELP\NEW\tasks\Hyakkiyakou\hya'
foreach($n in 'has_prob_up.png','limit_time_0.png','bean_0.png','time_over.png','gb_buff_1.png','damo.png'){ Copy-Item -LiteralPath "$src\$n" -Destination "$dst\$n" -Force }
```
Expected: 6 个文件出现在 V1 的 `hya/` 下。

- [ ] **Step 2: 校验字节一致**

Run:
```powershell
$src='G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\hya'
$dst='G:\HELP\NEW\tasks\Hyakkiyakou\hya'
foreach($n in 'has_prob_up.png','limit_time_0.png','bean_0.png','time_over.png','gb_buff_1.png','damo.png'){
  $a=(Get-FileHash -LiteralPath "$src\$n" -Algorithm SHA256).Hash
  $b=(Get-FileHash -LiteralPath "$dst\$n" -Algorithm SHA256).Hash
  "{0}  {1}" -f $n, ($a -eq $b)
}
```
Expected: 六行全部为 `True`。

- [ ] **Step 3: 提交**

Run:
```powershell
git -C G:\HELP\NEW add tasks/Hyakkiyakou/hya/has_prob_up.png tasks/Hyakkiyakou/hya/limit_time_0.png tasks/Hyakkiyakou/hya/bean_0.png tasks/Hyakkiyakou/hya/time_over.png tasks/Hyakkiyakou/hya/gb_buff_1.png tasks/Hyakkiyakou/hya/damo.png
git -C G:\HELP\NEW commit -m "feat(Hyakkiyakou): 迁入 V2 新增识别图模板"
```
Expected: 6 个文件入提交。

---

### Task 3: 替换 3 张已过时识别图模板

**Files:**
- Modify: `tasks/Hyakkiyakou/hya/hya_haccess.png`
- Modify: `tasks/Hyakkiyakou/hya/hya_hend.png`
- Modify: `tasks/Hyakkiyakou/hya/hya_ensure_recall.png`

**Interfaces:**
- Consumes: V2 同名文件
- Produces: V1 中这三个模板与新 ROI 配套（ROI 在 Task 4/5 更新），**替换后必须同批完成 Task 4**，否则模板与 ROI 不匹配会导致识别回归。

- [ ] **Step 1: 记录替换前后哈希**

Run:
```powershell
$dst='G:\HELP\NEW\tasks\Hyakkiyakou\hya'
foreach($n in 'hya_haccess.png','hya_hend.png','hya_ensure_recall.png'){ "{0}  BEFORE {1}  {2}B" -f $n,(Get-FileHash -LiteralPath "$dst\$n" -Algorithm SHA256).Hash,(Get-Item -LiteralPath "$dst\$n").Length }
```
Expected: 打印三个旧哈希与旧大小（应为 19250 / 36215 / 8624 字节）。

- [ ] **Step 2: 覆盖为新模板**

Run:
```powershell
$src='G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\hya'
$dst='G:\HELP\NEW\tasks\Hyakkiyakou\hya'
foreach($n in 'hya_haccess.png','hya_hend.png','hya_ensure_recall.png'){ Copy-Item -LiteralPath "$src\$n" -Destination "$dst\$n" -Force }
```
Expected: 文件被替换，大小变为 5761 / 7426 / 2461 字节。

- [ ] **Step 3: 校验字节一致**

Run:
```powershell
$src='G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\hya'
$dst='G:\HELP\NEW\tasks\Hyakkiyakou\hya'
foreach($n in 'hya_haccess.png','hya_hend.png','hya_ensure_recall.png'){ "{0}  {1}" -f $n,((Get-FileHash -LiteralPath "$src\$n" -Algorithm SHA256).Hash -eq (Get-FileHash -LiteralPath "$dst\$n" -Algorithm SHA256).Hash) }
```
Expected: 三行全部 `True`。

---

### Task 4: 同步 `assets.py` 规则声明

**Files:**
- Modify: `tasks/Hyakkiyakou/assets.py`

**Interfaces:**
- Consumes: Task 2/3 的图片文件；V1 的 `module/atom/image.py:RuleImage.__init__(roi_front, roi_back, method, threshold, file)`。
- Produces: `HyakkiyakouAssets` 上新增 `I_HAS_PROB_UP` / `I_LIMIT_TIME_0` / `I_BEAN_0` / `I_TIME_OVER` / `I_BUFF_1` / `I_CHECK_DAMO` / `C_HSELECT_TEMP`，并修正 `I_HACCESS`。

- [ ] **Step 1: 修正 `I_HACCESS` 的 ROI 与阈值**

把当前行：
```python
	I_HACCESS = RuleImage(roi_front=(1059,554,100,100), roi_back=(1059,554,100,100), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/hya_haccess.png")
```
改为：
```python
	I_HACCESS = RuleImage(roi_front=(999,529,202,152), roi_back=(999,529,202,152), threshold=0.7, method="Template matching", file="./tasks/Hyakkiyakou/hya/hya_haccess.png")
```

- [ ] **Step 2: 新增 6 条 ImageRule（V1 `file=` 写法）**

在 `I_ENSURE_RECALL` 之后新增：
```python
	# 概率up
	I_HAS_PROB_UP = RuleImage(roi_front=(172,9,96,32), roi_back=(157,1,843,59), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/has_prob_up.png")
	# 倒计时0
	I_LIMIT_TIME_0 = RuleImage(roi_front=(611,15,40,40), roi_back=(611,15,40,40), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/limit_time_0.png")
	# 豆子已空
	I_BEAN_0 = RuleImage(roi_front=(129,642,28,36), roi_back=(129,642,28,36), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/bean_0.png")
	# 时间终了
	I_TIME_OVER = RuleImage(roi_front=(650,328,46,59), roi_back=(650,328,46,59), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/time_over.png")
	# 庭院左上角的加成
	I_BUFF_1 = RuleImage(roi_front=(363,32,32,46), roi_back=(344,14,128,76), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/gb_buff_1.png")
	# 达摩武场
	I_CHECK_DAMO = RuleImage(roi_front=(102,23,125,36), roi_back=(102,23,125,36), threshold=0.8, method="Template matching", file="./tasks/Hyakkiyakou/hya/damo.png")
```

- [ ] **Step 3: 新增 `C_HSELECT_TEMP` 点击规则**

在 `C_HSELECT_1` 之前新增：
```python
	# 临时变量
	C_HSELECT_TEMP = RuleClick(roi_front=(0,0,0,0), roi_back=(0,0,0,0), name="hselect_temp")
```

- [ ] **Step 3b: 补齐差异清单 B 中遗漏的 4 条朋友 ROI 声明**

在 `assets.py` 中同步以下 4 条声明，均保持 V1 的 `file=` 形式：

- `C_FRIEND_1`：`roi_front`/`roi_back` 从 `(466,227,161,67)` 改为 `(479,275,140,12)`。
- `C_FRIEND_2`：`roi_front`/`roi_back` 从 `(738,229,158,64)` 改为 `(738,275,140,12)`。
- `I_FRIEND_SAME_1_RECALL`：`roi_back` 从 `(114,102,132,94)` 改为 `(154,105,132,94)`（`roi_front` 保持 `(167,123,106,62)`）。
- `I_FRIEND_SAME_2_RECALL`：`roi_back` 从 `(107,114,133,80)` 改为 `(152,112,133,80)`（`roi_front` 保持 `(168,126,100,60)`）。

- [ ] **Step 4: 导入校验**

Run:
```powershell
& G:\HELP\NEW\toolkit\python.exe -c "import sys; sys.path.insert(0, r'G:\HELP\NEW'); from tasks.Hyakkiyakou.assets import HyakkiyakouAssets as A; print(A.I_HAS_PROB_UP.roi_front, A.I_HACCESS.threshold, A.C_HSELECT_TEMP.name)"
```
Expected: 输出 `(172, 9, 96, 32) 0.7 hselect_temp`，无导入异常。

- [ ] **Step 5: 提交**

Run:
```powershell
git -C G:\HELP\NEW add tasks/Hyakkiyakou/assets.py tasks/Hyakkiyakou/hya/hya_haccess.png tasks/Hyakkiyakou/hya/hya_hend.png tasks/Hyakkiyakou/hya/hya_ensure_recall.png
git -C G:\HELP\NEW commit -m "feat(Hyakkiyakou): 同步 V2 识别图声明并更新过时模板"
```
Expected: 图片与 `assets.py` 同批入提交（模板与 ROI 必须同提交，避免中间态回归）。

---

### Task 5: 合并规则 JSON（image1 / image5 / click）

**Files:**
- Modify: `tasks/Hyakkiyakou/hya/image1.json`
- Modify: `tasks/Hyakkiyakou/hya/image5.json`
- Modify: `tasks/Hyakkiyakou/hya/click.json`

**Interfaces:**
- Consumes: 与 Task 4 相同的数值来源
- Produces: JSON 与 `assets.py` 数值保持一一对应。

- [ ] **Step 1: `image1.json` —— 改 `haccess` 并追加 5 条**

将 `haccess` 的 `roiFront`/`roiBack` 改为 `"999,529,202,152"`，`threshold` 改为 `0.7`；并在 `ensure_recall` 之后追加 `has_prob_up`、`limit_time_0`、`bean_0`、`time_over`、`buff_1` 五条，字段与 Task 4 Step 2 一一对应（`imageName` 分别为 `has_prob_up.png`、`limit_time_0.png`、`bean_0.png`、`time_over.png`、`gb_buff_1.png`）。

- [ ] **Step 2: `image5.json` —— 改两个 roiBack 并追加 `check_damo`**

`friend_same_1_recall.roiBack` 改为 `"154,105,132,94"`；`friend_same_2_recall.roiBack` 改为 `"152,112,133,80"`；末尾追加 `check_damo`（`imageName: "damo.png"`，roi `"102,23,125,36"`，threshold `0.8`）。

- [ ] **Step 3: `click.json` —— 追加 `hselect_temp` 并改两个朋友 ROI**

文件开头追加 `hselect_temp`（roi 全 `0,0,0,0`，`description: "临时变量"`）；`friend_1` 改为 `"479,275,140,12"`；`friend_2` 改为 `"738,275,140,12"`。

- [ ] **Step 4: JSON 合法性与键集校验**

Run:
```powershell
$d='G:\HELP\NEW\tasks\Hyakkiyakou\hya'
foreach($f in 'image1.json','image5.json','click.json'){ $j=Get-Content -LiteralPath "$d\$f" -Raw -Encoding UTF8 | ConvertFrom-Json; "OK $f  items=$($j.Count)" }
```
Expected: 三行 `OK`，无 JSON 解析异常。

- [ ] **Step 5: 提交**

Run:
```powershell
git -C G:\HELP\NEW add tasks/Hyakkiyakou/hya/image1.json tasks/Hyakkiyakou/hya/image5.json tasks/Hyakkiyakou/hya/click.json
git -C G:\HELP\NEW commit -m "feat(Hyakkiyakou): 合并 V2 规则 JSON（新增检查点与修正 ROI）"
```
Expected: 三个 JSON 入提交。

---

### Task 6: 评估并（有条件地）迁入模型

**Files:**
- Read: `G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\agent\tracker.onnx`
- Read: `G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\agent\detect_rarity.onnx`
- Read: `G:\HELP\NEW\toolkit\Lib\site-packages\oashya\oashya_fp32.onnx`
- Create（仅当闸门通过）: `tasks/Hyakkiyakou/agent/tracker.onnx`

**Interfaces:**
- Consumes: `onnxruntime.InferenceSession` 读取输入/输出签名
- Produces: 一份书面的闸门结论；仅当签名与 V1 的 `oashya` Tracker 兼容时才落地文件。

**闸门规则（必须逐条判定，不可跳过）：**

1. 若 V2 `tracker.onnx` 的输入名/形状、输出名/形状与 V1 `oashya_fp32.onnx` **完全一致** → 可作为可替换权重迁入。
2. 若签名不同 → **不迁入**。理由：能驱动它的 `agent/tracker.pyd` 属于排除项，V1 的 `oashya.tracker.Tracker` 无法加载，强行放入只会得到一个永不生效的死文件。
3. `detect_rarity.onnx` 同理；它与 V2 的 `C_HSELECT_TEMP` + 编译逻辑绑定，V1 现有代码没有对应推理入口。

- [ ] **Step 1: 打印三份模型签名**

Run:
```powershell
$py='G:\HELP\NEW\toolkit\python.exe'
$code = @'
import onnxruntime as ort
paths = {
 "v1_oashya": r"G:\HELP\NEW\toolkit\Lib\site-packages\oashya\oashya_fp32.onnx",
 "v2_tracker": r"G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\agent\tracker.onnx",
 "v2_rarity": r"G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\agent\detect_rarity.onnx",
}
for k, p in paths.items():
    s = ort.InferenceSession(p, providers=["CPUExecutionProvider"])
    print(k, [ (i.name, i.shape) for i in s.get_inputs() ], [ (o.name, o.shape) for o in s.get_outputs() ])
'@
& $py -c $code
```
Expected: 打印三行签名，用于比对。

- [ ] **Step 2: 记录判定结论**

在本计划书末尾「执行记录」小节追加一行：日期、模型、签名是否一致、是否落地、理由。

- [ ] **Step 3: 仅在闸门 1 通过时复制并提交**

Run（条件执行）:
```powershell
Copy-Item -LiteralPath 'G:\百鬼全自动2(1)\百鬼全自动\toolkit-不能用模拟器\toolkit\tasks\Hyakkiyakou\agent\tracker.onnx' -Destination 'G:\HELP\NEW\tasks\Hyakkiyakou\agent\tracker.onnx' -Force
git -C G:\HELP\NEW add tasks/Hyakkiyakou/agent/tracker.onnx
git -C G:\HELP\NEW commit -m "feat(Hyakkiyakou): 迁入签名兼容的 V2 跟踪模型"
```
Expected: 闸门未通过时不执行本步，改在本计划书记录「不迁入」。

---

### Task 7: 端到端验证与收尾

**Files:**
- Test: `tasks/Hyakkiyakou/assets.py` 的导入与资源存在性
- Test: 实机/模拟器跑一轮百鬼

**Interfaces:**
- Consumes: 前六个 Task 的产物
- Produces: 一份验证通过记录

- [ ] **Step 1: 资源存在性全量校验**

Run:
```powershell
$py='G:\HELP\NEW\toolkit\python.exe'
$code = @'
import sys, os
sys.path.insert(0, r"G:\HELP\NEW")
from tasks.Hyakkiyakou.assets import HyakkiyakouAssets as A
names = ["I_HAS_PROB_UP","I_LIMIT_TIME_0","I_BEAN_0","I_TIME_OVER","I_BUFF_1","I_CHECK_DAMO"]
for n in names:
    r = getattr(A, n)
    p = os.path.join(r"C:\", r.file.lstrip("./").replace("/", os.sep))
    print(n, os.path.exists(p), r.roi_front)
'@
& $py -c $code
```
Expected: 六行 `True`。

- [ ] **Step 2: 确认未引入任何授权相关文件**

Run:
```powershell
git -C G:\HELP\NEW status --short | Select-String -Pattern 'Identity|bypass|injector|sitecustomize|\.pyd|settings\.json|start_main'
```
Expected: 无输出（确认排除项一个都没进来）。

- [ ] **Step 3: 跑一轮百鬼任务**

Run: 通过 V1 的 `oas.exe` / WebUI 启动一次 `Hyakkiyakou`，至少完成 1 局。
Expected: 能正常识别进入按钮（`I_HACCESS` 新 ROI 生效）、正常开砸豆、正常结束退出，日志无 `Lost access page` 三连败或黑屏报错。

- [ ] **Step 4: 记录结果并收尾提交**

Run:
```powershell
git -C G:\HELP\NEW add -A tasks/Hyakkiyakou docs/superpowers/plans/2026-10-12-hyakkiyakou-v2-resource-migration.md
git -C G:\HELP\NEW commit -m "test(Hyakkiyakou): V2 资源回迁验证通过"
```
Expected: 收尾提交完成。

## 风险与回退

- **`I_HACCESS` 是高频页判断**：ROI 从 (1059,554,100,100) 改为 (999,529,202,152)、阈值 0.8→0.7，属行为敏感改动。若 Task 7 Step 3 出现进入页误判/漏判，回退 Task 3+4 中对 `hya_haccess.png` 与 `I_HACCESS` 的改动即可，其余新增资源不受影响。
- **新增检查点默认不生效**：它们只是被声明，V1 现有流程不会调用。若要真正启用（例如倒计时归零、豆空、时间终了、达摩武场、庭院加成），需要另立计划在 `script_task.py` / `slave` 中接线。
- **模型不可用风险**：见 Task 6 闸门。签名不一致时坚决不落地，避免留下死文件。
- **整体回退**：`git reset --hard hya-baseline-20261012`（仅在用户明确要求时使用）。

## 执行记录

### Task 6 模型闸门结论（2026-10-12）

- 判定：**不迁入** V2 的 `agent/tracker.onnx` 与 `agent/detect_rarity.onnx`。
- 依据：
  - V1 的 `toolkit/Lib/site-packages/oashya/oashya_fp32.onnx` 不是明文 ONNX，而是 Fernet 加密封装（文件头 `gAAAAA…`），由 oashya 的 `tracker.cp310-win_amd64.pyd` 在加载时解密，无法读取其输入/输出签名与 V2 模型比对，因此闸门规则 1 无法成立。
  - V2 的两个模型是真实 ONNX（producer = pytorch）：
    - `tracker.onnx`：in `images [1,3,736,1280]` → out `output0 [1,197,19320]`
    - `detect_rarity.onnx`：in `images [1,3,736,1280]` → out `output0 [1,186,19320]`
  - 两者输入为 736×1280（与游戏帧 720×1280 不同，需要专门预处理），且由被排除的编译模块 `agent/tracker.pyd` 驱动；V1 现有的 `oashya` 推理入口无法加载它们，落盘即死文件。
- 结论：按闸门规则 2 不落地，避免引入永不生效的文件。

### Task 7 验证结果（2026-10-12）

- 资源存在性：6 张新增模板 + 3 张替换模板全部存在，字节数与 V2 源一致。
- 声明一致性：`assets.py` 与 `image1.json` / `image5.json` / `click.json` 共 12 项 ROI/阈值全部一致（脚本校验 12/12 OK）。
- 排除项审计：分支 `main..HEAD` 的变更文件中未出现任何 `Identity` / `bypass` / `injector` / `sitecustomize` / `*.pyd` / `settings.json` / `start_main` / `ghosts` 路径。
- 未误改：`ocr.json`、`swipe.json`、`image2/3/4.json`、`train/*.json` 未被修改（它们与 V2 仅为换行符差异）。
- 待人工完成：**实机跑一轮百鬼**。本执行环境无法驱动模拟器/游戏，Task 7 Step 3 的真机冒烟测试需在 `G:\HELP\NEW` 上手动执行，重点确认新的 `I_HACCESS` ROI/阈值不会造成进入页误判。
