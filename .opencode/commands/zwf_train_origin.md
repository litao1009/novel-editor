---
name: zwf_train_origin
description: 训练基准小说白皮书。用法: /zwf_train_origin batch（批量扫描）/zwf_train_origin {原作名} [源文件...]（指定训练，支持覆盖）
subtask: false
---

【强制路径约定】
1. 基准原作目录：workspace/repo/{原作名}/
2. 源文件名：不固定，自动检测或由参数指定；支持多个文件列表
3. 白皮书输出名：固定为 base_whitepaper.md（不可变）
4. 调用 @original_analyst 时传入源文件路径列表 + 输出目录

你是基准小说训练命令执行器。用户输入了：{input}

严格按以下规则执行。

---

## 一、模式判断

解析 {input} 内容，按优先级判定：

| 匹配规则 | 模式 | 语义 |
|----------|------|------|
| 以 `--help` 开头 | **帮助** | 输出本文件中的"命令语法"章节内容 |
| 仅一个词且为 `batch` | **批量扫描** | 全量自动训练 |
| 仅一个词（如 `娘娘本纪`） | **指定训练·自动检测** | 自动检测目录中的源文件 |
| 两个词以上（如 `娘娘本纪 全文.txt`） | **指定训练·显式指定** | 使用指定文件作为源文件 |
| {input} 为空 | **引导** | 提示用户输入 batch 或作品名 |

---

## 二、源文件自动检测算法

当用户未显式指定源文件时，执行自动检测：

```
function detect_source_files(dir):
    files = ls(dir)
    candidates = []

    for file in files:
        if file == "base_whitepaper.md": continue      # 排除白皮书输出
        if file starts with ".": continue               # 排除隐藏文件
        if file.extension in [".txt", ".md"]:           # 仅接受文本类
            candidates.append(file)

    if candidates is empty:
        raise Error("在 {dir} 中未找到源文件（已排除 base_whitepaper.md）")

    return sorted(candidates)  # 按文件名排序，保证顺序稳定
```

---

## 三、模式 A：批量扫描

**Step 1 — 扫描源文件**
扫描 workspace/repo/ 下所有子目录，对每个子目录执行源文件自动检测。

**Step 2 — 筛选待训练列表**
满足以下条件的目录加入待训练列表：
- 检测到至少一个源文件（排除 base_whitepaper.md 后的 .txt/.md 文件）
- 不存在 base_whitepaper.md

若待训练列表为空，输出"所有基准小说已完成训练，无需增量"。

**Step 3 — 逐个训练（串行）**
遍历待训练列表，对每个 {原作名}：
- 调用 @original_analyst
  → 输入：该目录下检测到的所有源文件（作为文件列表）
  → 输出：workspace/repo/{原作名}/base_whitepaper.md
- 记录结果：成功/失败（含原因）

**Step 4 — 汇总报告**
输出完整的训练汇总报告（详见输出格式章节）。

---

## 四、模式 B：指定训练

**Step 1 — 校验原作目录**
- 验证 workspace/repo/{原作名}/ 目录是否存在
- 不存在 → 报错：workspace/repo/{原作名}/ 不存在

**Step 2 — 确定源文件**
- 若用户显式指定了源文件名 → 逐文件验证存在性
  - 任一文件不存在 → 报错：指定的源文件 {路径} 不存在
- 若用户未指定源文件 → 执行自动检测
  - 检测为空 → 报错：未找到源文件（排除 base_whitepaper.md）

**Step 3 — 执行训练**
调用 @original_analyst
→ 输入：检测到的或用户指定的源文件列表
→ 输出：workspace/repo/{原作名}/base_whitepaper.md（始终覆盖写入）

**Step 4 — 验证**
确认 base_whitepaper.md 已成功写入该目录。

**Step 5 — 输出结果**
输出训练结果报告（详见输出格式章节）。

---

## 五、调用 original_analyst 的约定

@original_analyst 接收以下参数：

```
- 原作目录路径：workspace/repo/{原作名}/
- 源文件列表：  [全文.txt] 或 [上卷.txt, 下卷.txt] 或 [source.txt] 等
- 输出文件名：  base_whitepaper.md（固定，写入原作目录）
```

若 original_analyst 当前只接受单个文件路径：
- 当只有一个源文件时，直接传入
- 当有多个源文件时，先按文件名顺序合并为一个临时文件再传入
  （临时文件放在原作目录下，训练完成后清理）

---

## 六、错误处理

| 场景 | 检测方式 | 错误消息 | 处理行为 |
|------|---------|----------|---------|
| {input} 为空 | 参数判空 | "请输入 batch 或指定作品名。使用 --help 查看帮助" | 引导用户 |
| 原作目录不存在 | Test-Path dir | "workspace/repo/{原作名}/ 不存在" | 中止 |
| 未检测到源文件 | 自动检测为空 | "未找到源文件，请确认目录中包含 .txt 或 .md 文件（已排除 base_whitepaper.md）" | 中止 |
| 显式指定的文件不存在 | Test-Path file | "指定的源文件 {路径} 不存在" | 中止 |
| original_analyst 调用失败 | task 返回错误 | "训练失败：{错误详情}" | batch: 跳过继续; single: 中止 |
| 输出文件写入验证失败 | 完成时确认 | "白皮书写入验证失败：workspace/repo/{原作名}/base_whitepaper.md 未生成" | 标记失败 |

---

## 七、输出格式

### 批量模式输出
```
━━━ 基准小说训练报告 ━━━
📊 总计: N 部待训练
✅ 成功: N 部
❌ 失败: N 部
📋 明细:
  ✅ {原作名} → base_whitepaper.md（源文件: {文件名列表}）
  ❌ {原作名} → {失败原因}
━━━ 报告结束 ━━━
```

### 指定模式输出
```
━━━ 基准小说训练结果 ━━━
📖 作品: {原作名}
📂 源文件: [{文件名列表}]
✅ 状态: 训练成功
📄 白皮书: workspace/repo/{原作名}/base_whitepaper.md
━━━ 报告结束 ━━━
```

### 帮助输出（--help）
```
zwf_train_origin — 训练基准小说白皮书

用法:
  /zwf_train_origin batch
      批量扫描所有未训练的基准小说
      （自动检测各目录中的源文件，排除已有白皮书的目录）

  /zwf_train_origin {原作名}
      训练指定小说，自动检测目录中的源文件
      如: /zwf_train_origin 娘娘本纪

  /zwf_train_origin {原作名} {源文件}
      显式指定单个源文件
      如: /zwf_train_origin 娘娘本纪 全文.txt

  /zwf_train_origin {原作名} {文件1} {文件2} ...
      显式指定多个源文件
      如: /zwf_train_origin 娘娘本纪 上卷.txt 下卷.txt

规则:
  - 源文件不限制文件名，支持 .txt / .md 及多个文件
  - 输出文件固定为 base_whitepaper.md
  - 指定训练始终覆盖重写
```

---

## 八、使用示例

### 批量扫描所有未训练的基准小说
```
> /zwf_train_origin batch
━━━ 基准小说训练报告 ━━━
📊 总计: 1 部待训练
✅ 成功: 1 部
❌ 失败: 0 部
📋 明细:
  ✅ 新书 → base_whitepaper.md（源文件: 全文.txt）
━━━ 报告结束 ━━━
```

### 训练指定小说（自动检测源文件）
```
> /zwf_train_origin 娘娘本纪
━━━ 基准小说训练结果 ━━━
📖 作品: 娘娘本纪
📂 源文件: [source.txt]
✅ 状态: 训练成功
📄 白皮书: workspace/repo/娘娘本纪/base_whitepaper.md
━━━ 报告结束 ━━━
```

### 显式指定多个源文件
```
> /zwf_train_origin 娘娘本纪 上卷.txt 下卷.txt
━━━ 基准小说训练结果 ━━━
📖 作品: 娘娘本纪
📂 源文件: [上卷.txt, 下卷.txt]
✅ 状态: 训练成功
📄 白皮书: workspace/repo/娘娘本纪/base_whitepaper.md
━━━ 报告结束 ━━━
```

### 错误场景
```
> /zwf_train_origin 不存在的小说
✗ 错误: workspace/repo/不存在的小说/ 不存在

> /zwf_train_origin 空白目录
✗ 错误: 未找到源文件，请确认目录中包含 .txt 或 .md 文件（已排除 base_whitepaper.md）
```
