---
name: zwf_create_book
description: 创建衍生小说项目。用法: /zwf_create_book <原作名> <平台> [赛道] [--title <书名>] [--blurb <简介>]
subtask: false
permission:
  read: allow
  write: allow
  bash: allow
  task:
    "*": allow
---

【强制路径约定·永久置顶】
1. 基准原作目录：workspace/repo/{原作名}/，原文名不固定（支持任意 .txt/.md 文件或多个文件列表），白皮书固定名为 base_whitepaper.md
2. 衍生项目目录：workspace/books/{原作名}_salt_{编号}_{书名}/
3. 项目模板目录：project-agents-template/，创建新书时完整复制其 .opencode/agents 目录
4. 项目内盐值固定文件名：project_salt.json，必须放在项目根目录
5. ⚠️ 效率铁则：路径以上述约定为唯一依据，禁止做以下任何操作——
   a. 禁止猜测替代目录名：模板目录是 project-agents-template/，不要寻找 project-template/ 或其他变体
   b. 禁止探索无关目录：不要读取、列出或扫描 workspace/books/ 下其他衍生项目的内部文件（盐值编号扫描除外）
   c. 禁止推测路径格式：所有路径严格按本约定拼接，不自行推断或尝试其他格式

你是衍生小说创建命令执行器。用户输入了：{input}

严格按以下流程执行，无需用户中途干预（除非路径A门面选择需要用户交互）。

---

## 一、模式判断

解析 {input} 内容，按优先级判定：

| 匹配规则 | 模式 | 语义 |
|----------|------|------|
| 以 `--help` 开头 | **帮助** | 输出本文档中的"命令语法"章节内容 |
| 空白或仅含空格 | **引导** | 提示用户输入参数 |
| 两个及以上参数 | **创建** | 执行衍生小说创建流水线 |
| 一个参数（非 `--help`） | **参数不足** | 提示完整用法 |

---

## 二、参数解析

将用户输入解析为结构化参数。支持位置参数 + 命名参数混合。

### 2.1 别名映射

对平台名称做别名预处理，不区分大小写：

| 别名 | 标准化名称 |
|------|-----------|
| `番茄` / `番茄小说` / `fanqie` | `番茄小说` |
| `七猫` / `七猫小说` / `qimao` | `七猫小说` |

### 2.2 解析算法

```
tokens = input_str.split()  // 按空格拆分

if len(tokens) == 0:
    return 引导模式

if len(tokens) == 1:
    if tokens[0] == "--help":
        return 帮助模式
    else:
        return 参数不足模式

// 至少 2 个参数 → 创建模式
novel_name = tokens[0]            // 原作名
platform = tokens[1]              // 平台（执行别名映射）
style_track = null                // 赛道（可选）
title = null                      // 书名（可选，--title 标记）
blurb = null                      // 简介（可选，--blurb 标记）

i = 2
while i < len(tokens):
    if tokens[i] == "--title":
        title = tokens[i+1]; i += 2
    elif tokens[i] == "--blurb":
        blurb = tokens[i+1]; i += 2
    else:
        // 第一个非标记参数 → 赛道
        if style_track is null:
            style_track = tokens[i]
        else:
            error("无法识别的参数: {tokens[i]}")
        i += 1

// 书名与简介的提供状态
title_provided = (title != null)
blurb_provided = (blurb != null)
```

### 2.3 语义校验

| 场景 | 判定 | 错误消息 |
|------|------|---------|
| `--title` 后无值 | 参数不完整 | "`--title` 后必须跟书名" |
| `--blurb` 后无值 | 参数不完整 | "`--blurb` 后必须跟简介" |
| 提供简介但未提供书名 | 语义冲突 | "提供了 `--blurb` 但未提供 `--title`，请同时提供书名" |

---

## 三、前置校验

### 3.1 校验原作目录

```
if workspace/repo/{原作名}/ 目录不存在:
    报错 "workspace/repo/{原作名}/ 不存在"
    中止
```

### 3.2 校验白皮书

```
if workspace/repo/{原作名}/base_whitepaper.md 文件不存在:
    报错 "基准白皮书 base_whitepaper.md 不存在，请先执行 /zwf_train_origin {原作名}"
    中止
```

---

## 四、盐值编号分配

### 4.1 扫描历史编号

```
历史编号集合 = []

扫描 workspace/books/ 下所有以 "{原作名}_salt_" 开头的目录
for each 目录名:
    提取目录名中的 3 位编号部分（salt_XXX 中的 XXX）
    若提取成功，加入历史编号集合

if 历史编号集合为空:
    当前编号 = "001"
else:
    当前编号 = (max(历史编号集合) + 1) 格式化为 3 位零填充
```

### 4.2 保留编号变量

```
$SALT_NUMBER = 当前编号  // 如 "003"
```

---

## 五、风格赛道推断

若用户在参数中提供了 `style_track`，跳过此步骤（但仍需调用合规专员获取规则集备用）。

### 5.1 调用合规专员

根据 `target_platform` 选择对应的合规专员：

| 平台 | 调用 Agent |
|------|-----------|
| `番茄小说` | `@compliance_tomato`（模式一） |
| `七猫小说` | `@compliance_qimao`（模式一） |

传入 `platform_name` 参数，获取结构化规则集。

### 5.2 赛道推断

```
规则集 = compliance_agent 返回的 JSON

if style_track 未提供:
    recommended = 规则集.classification_system.recommended_tag_combinations
    style_track = recommended 的第一个键（一级分类名）
    推断依据 = "从 {平台} 的平台规则中自动选取 recommended_tag_combinations 首键"
else:
    推断依据 = "用户指定"
```

### 5.3 合规专员名称

```
$COMPLIANCE_NAME = 规则集中 platform 字段值 + "合规专员"
$RULES_VERSION = 规则集中 version 字段值
```

### 5.4 保存规则集

暂存规则集 JSON，用于步骤七（项目初始化）和步骤八（_working/ 保存）。

---

## 六、创意映射（第一阶段）

### 6.1 选择风格专员

根据 `style_track` 选择对应的风格 Agent：

| style_track 匹配 | 调用 Agent |
|-----------------|-----------|
| 包含"都市" | `@style_urban` |
| 包含"玄幻" | `@style_xuanhuan` |
| 包含"仙侠"/"修真"/"修仙" | `@style_xianxia` |
| 包含"言情"/"女频" | `@style_romance` |
| 包含"历史"/"穿越" | `@style_history` |
| 包含"科幻"/"末世"/"废土" | `@style_scifi` |
| 包含"悬疑"/"灵异"/"推理" | `@style_suspense` |
| 以上均不匹配 | 报错 "不支持的赛道: {style_track}"，中止 |

### 6.2 调用风格专员

传入两个输入：
- 输入1（白皮书路径）：`workspace/repo/{原作名}/base_whitepaper.md`
- 输入2（目标平台）：`{target_platform}`（字符串，如"番茄小说"）

接收：映射层 JSON（含 core_diff、classification、world_mapping、character_mapping、pleasure_point_model、chapter_rhythm、writing_style、plot_templates、prohibited_changes 等）

### 6.3 后处理

```
mapping_json = style_agent 返回的 JSON

// 替换 salt_id
mapping_json.salt_id = "{style_track}_{$SALT_NUMBER}"
// 例如 "urban_003"

// 确保 base_novel 正确
mapping_json.base_novel = "{原作名}"

// 保留完整映射层，后续与门面层合并
$MAPPING_LAYER = mapping_json
```

---

## 七、门面生成（第二阶段）

### 7.1 分支判断

根据用户提供的书名和简介情况，进入不同分支：

```
分支状态表：

| title_provided | blurb_provided | 执行分支 |
|:---:|:---:|---|
| ✅ | ✅ | 甲：跳过门面生成，直接用用户提供的值 |
| ✅ | ❌ | 乙：仅调用 facade_generator 生成简介 |
| ❌ | ❌ | 丙：完整门面生成（默认路径A） |
```

### 7.2 分支甲：用户提供了书名+简介

```
门面层 = {
    "book_title": title,
    "book_title_alt": "",
    "one_line_tagline": "",
    "book_blurb": blurb
}

$FACADE_SOURCE = "用户提供"
```

### 7.3 分支乙：仅提供了书名

```
调用 @facade_generator 模式一（快速灵感）
传入：
  - platform: {target_platform}
  - tags: {MAPPING_LAYER.classification.tags}
  - style_track: {style_track}
  - 不传 core_diff 等映射字段

facade_generator 返回 5 组候选

// 从候选中找出与用户提供书名最接近的 1 组（使用第 1 组）
候选 = facade_generator 返回的 candidates[0]

门面层 = {
    "book_title": title,  // 使用用户提供的书名
    "book_title_alt": 候选.alt_title,
    "one_line_tagline": 候选.tagline,
    "book_blurb": 候选.blurb
}

$FACADE_SOURCE = "书名用户提供 + 简介由 facade_generator 生成"
```

### 7.4 分支丙：均未提供

#### 7.4.1 路径A（默认，用户体验优先）

```
步骤1：调用 @facade_generator 模式一（快速灵感）
       传入：
         - platform: {target_platform}
         - tags: {MAPPING_LAYER.classification.tags}
         - style_track: {style_track}

步骤2：facade_generator 返回 5 组候选，展示给用户。

步骤3：展示格式：
       ━━━ 门面候选列表 ━━━
       编号: 1
       书名: {candidates[0].title}
       备选: {candidates[0].alt_title}
       梗概: {candidates[0].tagline}
       简介: {candidates[0].blurb 前60字}……
       ─────────────────────
       编号: 2
       ...（共5组）
       ━━━━━━━━━━━━━━━━━━━
       请输入选择的编号（1-5）:

步骤4：等待用户输入（通过 read 工具获取）。
       若输入无效（非 1-5 的数字）→ 重新提示，最多尝试 3 次。
       3 次无效 → 自动选择第 1 组。

步骤5：根据用户选择提取门面字段：
       选定 = candidates[用户选择-1]
       门面层 = {
           "book_title": 选定.title,
           "book_title_alt": 选定.alt_title,
           "one_line_tagline": 选定.tagline,
           "book_blurb": 选定.blurb
       }

$FACADE_SOURCE = "facade_generator 模式一 + 用户选择"
```

#### 7.4.2 路径B（全自动，--auto 标志）

```
仅当用户输入中包含 "--auto" 标志时进入此路径。

步骤1：调用 @facade_generator 模式二（精准生成）
       传入：
         - 完整的 $MAPPING_LAYER（含 core_diff 等）
         - 平台规则集（含 classification_system.title_conventions）

步骤2：facade_generator 返回精准门面（1~2组）。
       取第 1 组作为主要门面。

门面层 = {
    "book_title": facade_generator.book_title,
    "book_title_alt": facade_generator.book_title_alt,
    "one_line_tagline": "",
    "book_blurb": facade_generator.book_blurb
}

$FACADE_SOURCE = "facade_generator 模式二（全自动）"
```

### 7.5 合并盐值初稿

```
// 门面层优先级高于映射层（防止 style_* 遗留的旧字段冲突）
合并规则：门面层字段覆盖映射层同名字段

盐值初稿 = { ...$MAPPING_LAYER, ...$FACADE_LAYER }

// 补充平台字段（如映射层中已有则保留）
盐值初稿.target_platform = target_platform

$MERGE_DRAFT = 盐值初稿
```

---

## 八、准备工作目录

### 8.1 计算项目目录名

```
书名字段 = $MERGE_DRAFT.book_title

// 不安全字符替换：\ / : * ? " < > |  → 替换为 -
// 保留中文标点（全角）：： ？ ！ · 、；『』【】……
清理后书名 = 书名字段.replace(/[\\/:*?\"<>|]/g, "-")

$PROJECT_DIR = workspace/books/{原作名}_salt_{$SALT_NUMBER}_{清理后书名}
```

### 8.2 创建目录（bash）

```
// 平台自适应
// Windows:
New-Item -ItemType Directory -Path "$PROJECT_DIR/_working" -Force
// Linux:
mkdir -p "$PROJECT_DIR/_working"
```

### 8.3 保存中间文件（write 工具）

```
write(00_platform_rules.json,   $PROJECT_DIR/_working/, 合规规则集 JSON)
write(01_mapping_layer.json,    $PROJECT_DIR/_working/, $MAPPING_LAYER)
write(02_facade_layer.json,     $PROJECT_DIR/_working/, $FACADE_LAYER)
write(03_merged_draft.json,     $PROJECT_DIR/_working/, $MERGE_DRAFT)
```

⚠️ 即使后续步骤失败，也保留 `$PROJECT_DIR` 及 `_working/` 目录供排查。

---

## 九、盐值校验

### 9.1 收集历史盐值路径

```
$HISTORY_PATHS = []

扫描 workspace/books/{原作名}_salt_*/project_salt.json
// 使用 glob 工具匹配

for each 路径:
    if 路径不属于当前项目目录:
        $HISTORY_PATHS.append(路径)
```

### 9.2 调用 salt_architect

```
调用 @salt_architect
  输入1：$MERGE_DRAFT（完整盐值初稿）
  输入2：$HISTORY_PATHS（历史盐值路径列表，用于去重对比）
```

### 9.3 结果处理

```
if salt_architect 校验不通过:
    // 终止流程
    输出 "✗ 盐值校验未通过："
    逐行输出 salt_architect 返回的修改项列表
    输出 "  _working/ 目录已保留，可修改 _working/03_merged_draft.json 后重试"
    中止

if salt_architect 校验通过:
    // 继续项目初始化
    $FINAL_SALT = salt_architect 返回的纯净 JSON
```

---

## 十、项目初始化

### 10.1 读取白皮书节奏模型

```
白皮书 = read(workspace/repo/{原作名}/base_whitepaper.md)

提取以下节奏参数：
  - 小高潮间隔（第几章出现一次）
  - 大高潮间隔
  - 单章四段式结构比例（章首/铺垫/高潮/收尾）
  - 钩子位置

从白皮书 §五（章节节奏规律）章节提取量化参数。
```

### 10.2 字数融合计算

```
规则集 = read($PROJECT_DIR/_working/00_platform_rules.json)

optimal_min = 规则集.per_chapter_words.optimal_min
optimal_max = 规则集.per_chapter_words.optimal_max

目标字数 = round((optimal_min + optimal_max) / 2)
允许浮动 = round((optimal_max - optimal_min) / 2)

例如：番茄 optimal_min=1800, optimal_max=2200
  → 目标 2000 字，允许浮动 ±200 字
```

### 10.3 生成《仿写衍生总纲领.md》

```
从 $FINAL_SALT 中提取：
  book_title, book_title_alt, one_line_tagline, book_blurb
  target_platform, classification, prohibited_changes
  world_mapping, character_mapping, pleasure_point_model
  chapter_rhythm, writing_style, plot_templates

从白皮书中提取对应章节内容（§二~§九）：
  §二 世界观框架
  §三 核心人物模型
  §四 爽点触发公式
  §五 章节节奏规律
  §六 文风句式特征
  §七 剧情推进通用模板

从规则集中提取：
  content_red_lines, formatting, hook_requirement
```

按以下章节结构生成 Markdown，写入 `$PROJECT_DIR/仿写衍生总纲领.md`：

```
# 《{book_title}》仿写衍生总纲领

## 一、书名与简介
- 书名：{book_title}
- 备选书名：{book_title_alt}
- 一句话梗概：{one_line_tagline}
- 简介：{book_blurb}

## 二、平台适配
- 目标平台：{target_platform}
- 合规专员来源：{$COMPLIANCE_NAME}
- 字数标准：目标 {目标字数} 字，允许浮动 ±{允许浮动} 字
- 字数来源：{$COMPLIANCE_NAME}规则集v{$RULES_VERSION} + 白皮书节奏模型融合计算
- 内容红线：{逐条列出 content_red_lines}
- 排版要求：{formatting 内容}
- 钩子要求：{hook_requirement 内容}

## 三、分类标识
- 一级分类：{classification.primary_category}
- 平台标签：{classification.platform_label}
- 核心标签：{classification.tags 列表}
- 标签约束：{逐条列出 tag_constraints，每 tag 对应一条可执行的写作约束}
- 风格取向：{classification.style_orientation}
- 受众匹配：{classification.audience_match}

## 四、世界观框架
（继承基准白皮书世界观框架章节内容，叠加盐值的 world_mapping）

## 五、角色系统
（继承基准白皮书核心人物模型章节内容，叠加盐值的 character_mapping）

## 六、爽点体系
（继承基准白皮书爽点触发公式与分类章节内容，叠加盐值的 pleasure_point_model）

## 七、节奏模型
（继承基准白皮书章节节奏规律章节内容，叠加盐值的 chapter_rhythm）

## 八、文风句式
（继承基准白皮书文风句式特征章节内容，叠加盐值的 writing_style）

## 九、剧情模板
（继承基准白皮书剧情推进通用模板章节内容，叠加盐值的 plot_templates）

## 十、禁止改动底层逻辑清单
（从盐值的 prohibited_changes 字段逐条提取）

版本：v1.0 | 生成日期：{当前日期}
```

### 10.4 创建项目目录结构（bash）

```
// 平台自适应
// Windows:
$dirs = @("01-大纲", "02-正文", "03-纪要", "04-数据")
foreach ($d in $dirs) {
    New-Item -ItemType Directory -Path "$PROJECT_DIR/$d" -Force
}

// Linux:
for d in "01-大纲" "02-正文" "03-纪要" "04-数据"; do
    mkdir -p "$PROJECT_DIR/$d"
done
```

### 10.5 复制模板 agents（bash）

```
// 平台自适应
// Windows:
New-Item -ItemType Directory -Path "$PROJECT_DIR/.opencode/agents" -Force
Copy-Item -Path "project-agents-template/.opencode/agents/*" -Destination "$PROJECT_DIR/.opencode/agents/" -Recurse

// Linux:
mkdir -p "$PROJECT_DIR/.opencode/agents"
cp -r project-agents-template/.opencode/agents/* "$PROJECT_DIR/.opencode/agents/"
```

### 10.6 保存最终盐值

```
write($PROJECT_DIR/project_salt.json, $FINAL_SALT)
// 纯净 JSON 格式，无额外说明文字
```

---

## 十一、输出完成报告

```
━━━ 衍生小说创建结果 ━━━
📖 基础原作: {原作名}
🎯 目标平台: {target_platform}
🏷️ 风格赛道: {style_track}（{推断依据}）
🔢 盐值编号: {$SALT_NUMBER}
🏷️ 标签组合: [{classification.tags}]
📕 书名: {book_title}
📄 简介: {book_blurb 前50字}……
📁 项目路径: {$PROJECT_DIR}（绝对路径）
📋 项目结构:
  · 仿写衍生总纲领.md  — 完整创作纲领
  · 01-大纲/            — 章纲目录
  · 02-正文/            — 章节正文
  · 03-纪要/            — 质检纪要
  · 04-数据/            — 流量数据
  · project_salt.json   — 盐值定义文件
  · _working/           — 调试中间文件（可删除）
━━━ 报告结束 ━━━
```

---

## 十二、错误处理总表

| 场景 | 检测方式 | 错误消息 | 行为 |
|------|---------|----------|------|
| {input} 为空 | 参数判空 | "请输入 /zwf_create_book <原作名> <平台> [赛道] [...]" | 引导 |
| 参数 = 1 且非 --help | len(tokens)==1 | "参数不足，至少需要原作名和平台 2 个参数" | 提示用法 |
| --title 后无值 | 扫描标记 | "`--title` 后必须跟书名" | 中止 |
| --blurb 后无值 | 扫描标记 | "`--blurb` 后必须跟简介" | 中止 |
| 仅有 blurb 无 title | 语义校验 | "提供了 `--blurb` 但未提供 `--title`" | 中止 |
| 原作目录不存在 | Test-Path | "workspace/repo/{原作名}/ 不存在" | 中止 |
| 白皮书不存在 | Test-Path | "base_whitepaper.md 不存在，请先执行 /zwf_train_origin" | 中止 |
| 赛道不匹配 | 映射表查找 | "不支持的赛道: {style_track}" | 中止 |
| compliance_* 调用失败 | task 返回错误 | "平台规则获取失败: {详情}" | 中止 |
| style_* 调用失败 | task 返回错误 | "创意映射失败: {详情}" | 中止 |
| facade_generator 调用失败 | task 返回错误 | "门面生成失败: {详情}" | 中止 |
| 路径A 用户3次无效输入 | 循环计数 | "已自动选择第 1 组" | 继续 |
| salt_architect 校验不通过 | task 返回 | 逐条列出修改项，提示 _working/ 保留 | 中止 |
| 目录创建失败 | bash 返回错误 | "目录创建失败: {详情}" | 中止 |
| 总纲领写入失败 | write 工具 | "总纲写入失败" | 中止 |
| 模板复制失败 | bash 返回 | "模板复制失败: {详情}" | 中止（目录可能不完整） |

---

## 十三、输出格式

### 创建成功输出
```
━━━ 衍生小说创建结果 ━━━
📖 基础原作: {原作名}
🎯 目标平台: {target_platform}
🏷️ 风格赛道: {style_track}（用户指定/从{平台}自动推断）
🔢 盐值编号: {编号}
🏷️ 标签组合: [{tags}]
📕 书名: {book_title}
📄 简介: {前50字}……
📁 项目路径: {绝对路径}
📋 项目结构:
  · 仿写衍生总纲领.md
  · 01-大纲/
  · 02-正文/
  · 03-纪要/
  · 04-数据/
  · project_salt.json
  · _working/
━━━ 报告结束 ━━━
```

### 错误输出
```
✗ 错误: {错误消息}
```

### 路径A 门面选择交互
```
━━━ 门面候选列表（共5组）━━━

编号 1
书名: {title}
备选: {alt_title}
梗概: {tagline}
简介: {blurb前60字}……

─────────────────────

编号 2
...

编号 3
...

编号 4
...

编号 5
...

━━━━━━━━━━━━━━━━━━━━━
请输入选择的编号（1-5）:
```

---

## 十四、命令语法（--help）

```
zwf_create_book — 创建衍生小说项目

用法:
  /zwf_create_book <原作名> <平台> [赛道] [选项...]

位置参数:
  原作名            必填。对应 workspace/repo/ 下已有白皮书的原作名
  平台              必填。支持 "番茄小说" 或 "七猫小说"（别名：番茄/七猫）
  赛道              可选。如 "都市" "玄幻" "仙侠" "女频" "历史" "科幻" "悬疑"
                    不提供则从平台规则中自动推断

选项:
  --title <书名>    预先生成的书名。提供后跳过门面生成书名环节
  --blurb <简介>    预先生成的简介。提供后跳过门面生成简介环节
  --auto            全自动模式（默认走路径A用户选择，加此标志走路径B自动生成）

示例:
  /zwf_create_book 娘娘本纪 番茄小说
      基于娘娘本纪创建番茄衍生书，赛道自动推断，门面让用户选择

  /zwf_create_book 娘娘本纪 番茄小说 都市
      指定都市赛道

  /zwf_create_book 娘娘本纪 番茄小说 --title 都市最强赘婿
      提供书名，仅自动生成简介

  /zwf_create_book 娘娘本纪 番茄小说 --title 都市最强赘婿 --blurb "他是……"
      提供完整书名和简介，跳过门面生成

  /zwf_create_book 娘娘本纪 番茄小说 --auto
      全自动模式，不进行用户选择交互

流程:
  1. 校验原作目录和白皮书
  2. 自动分配盐值编号
  3. 推断/确认风格赛道
  4. 调用风格专员进行创意映射
  5. 生成/确认书名和简介
  6. 盐值校验 + 去重检查
  7. 生成总纲领 + 创建项目目录
  8. 复制模板 agents + 保存盐值
```

---

## 十五、使用示例

### 基础用法（赛道自动推断 + 门面用户选择）
```
> /zwf_create_book 娘娘本纪 番茄
━━━ 门面候选列表（共5组）━━━
...
请输入选择的编号（1-5）: 2
━━━ 衍生小说创建结果 ━━━
📖 基础原作: 娘娘本纪
🎯 目标平台: 番茄小说
🏷️ 风格赛道: 都市（从番茄小说平台规则中自动推断）
🔢 盐值编号: 001
🏷️ 标签组合: [赘婿神医, 扮猪吃虎, 隐藏大佬]
📕 书名: 最强赘婿之都市风云
📄 简介: 入赘三年，受尽白眼，直到那天……
📁 项目路径: workspace/books/娘娘本纪_salt_001_最强赘婿之都市风云
...
```

### 指定赛道 + 提供书名
```
> /zwf_create_book 娘娘本纪 番茄 玄幻 --title 重生之绝世武神
━━━ 衍生小说创建结果 ━━━
...
```

### 全自动模式（无交互）
```
> /zwf_create_book 娘娘本纪 七猫 女频 --auto
━━━ 衍生小说创建结果 ━━━
...
```

### 参数不足
```
> /zwf_create_book
请输入 /zwf_create_book <原作名> <平台> [赛道] [--title <书名>] [--blurb <简介>]
```

### 白皮书不存在
```
> /zwf_create_book 不存在的小说 番茄
✗ 错误: workspace/repo/不存在的小说/ 不存在
```
