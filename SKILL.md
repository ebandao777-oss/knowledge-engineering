---
name: knowledge-engineering
description: "工业级RAG切片工具「可落地、可量化、可优化」,将RAG知识库长文档拆解为语义完整、检索就绪的原子化知识切片，内置多层质量门禁（校验→审计→检索可达性评估），确保切片可用性与RAG检索命中率。"
keywords:
  [
    "rag",
    "切片",
    "知识库",
    "语义分割",
    "检索增强生成",
    "chunking",
    "文档拆分",
    "质量门禁",
    "embedding-hint",
    "self-check",
    "PurePythonEmbedder",
    "retrieval-evaluation",
    "cross-refs",
  ]
version: "1.0.13"
license: MIT
allowed-tools:
  - Read
  - Grep
  - Glob
  - Shell
  - Edit
  - Write
metadata:
  domain: "knowledge-engineering"
  author: "智慧半岛"
  platform:
    windows: full
    linux: full
    macos: full
  openclaw:
    requires:
      bins:
        - python
    emoji: "📚"
---

> 文档结构：SKILL.md = 核心运行时指令（Agent 执行必读），REFERENCE.md = 扩展参考与运维细节。

## 📖 分级导航

| 级别   | 内容                                                                                                                                            | 适用场景                                   |
| :----- | :---------------------------------------------------------------------------------------------------------------------------------------------- | :----------------------------------------- |
| **L1** | [核心红线](#-核心红线-critical-constraints) + [速查工作流](#速查标准工作流-quick-reference)                                                     | 首次加载，建立安全边界和执行路径           |
| **L2** | [§0 预处理CoT](#0-预处理思维链-pre-computation-cot-强制执行) → [§4 自检协议](#4-输出后自检协议-post-generation-self-correction)（核心执行流程） | 执行切片任务时必须完整遵循                 |
| **L3** | [REFERENCE.md](REFERENCE.md)（详细表格/完整规则/FAQ/运维细节）                                                                                  | 遇到边界情况、异常或需要完整规则时按需读取 |

---

# RAG知识库原子化切片控制器 (Auto-Archiving & Semantic-Preservation Edition)

> 🟢 **最小可用示例**：`python scripts/slice_generator.py input.md slices/`。完整工作流见 §0-§11。

## 速查：标准工作流 (Quick Reference)

从源文件到检索就绪切片的完整路径，5 步闭环：

| 步骤   | 动作                                       | 工具/方式                                    | 输出                     | 对应章节 |
| :----- | :----------------------------------------- | :------------------------------------------- | :----------------------- | :------- |
| ① 分析 | 结构扫描 + Token预算 + 熔断预演 + 目录决策 | `<analysis>` CoT                             | 切片计划清单             | §0       |
| ② 计划 | 自动生成分类路由与拆分方案                 | `slice_generator.py --json`                  | JSON 切片计划            | §8       |
| ③ 生成 | 逐切片填充语义内容 + 写入磁盘              | Agent 按 §3 模板生成                         | 带完整 YAML 的 .md 切片  | §3, §6-7 |
| ④ 校验 | 单切片完整性 + 跨切片审计                  | `validate_slice.py --fix` → `batch_audit.py` | 致命项重生成 / 警告标记  | §4, §8   |
| ⑤ 终检 | 检索可达性评估（R@1/R@5/MRR）              | `evaluate_retrieval.py --fix`                | 检索报告 + 死片/弱片修复 | §8       |

## Init-Step-Poll 渐进式防卡死协议

长文档切片、批量知识库导入、检索评估和断点续传必须采用 Init → Step → Poll。该协议不替代标准工作流，而是把标准工作流拆成可恢复、可验证的小步，防止 Token 溢出、长时间生成中断或部分切片失败后无法归因。

| 阶段 | 动作                                                                                | 输出                                                            | 失败回退                                                         |
| :--- | :---------------------------------------------------------------------------------- | :-------------------------------------------------------------- | :--------------------------------------------------------------- |
| Init | 完成结构扫描、Token 预算、熔断点预演、切片计划生成和输出目录确认                    | `task_id`、切片计划、总切片数、批次大小、进度 `0/N`             | 源文件不可读、计划为空或输出目录冲突时停止，等待用户确认         |
| Step | 每次只生成 1-2 个切片，立即写入、运行 `validate_slice.py --fix` 并记录 `SELF_CHECK` | 已生成切片 ID、校验结果、下一批切片范围                         | 单片校验失败时重生成该片；仍失败则暂停并输出失败字段             |
| Poll | 汇总已生成/已验证/失败/待生成数量，必要时运行 `batch_audit.py` 或检索抽检           | `running/success/failed/paused`、进度百分比、失败清单、续跑入口 | 中断后从最后一个 `SELF_CHECK` 通过的切片继续，不得覆盖已验证切片 |

执行约束：

- Init 阶段只生成计划和任务状态，不直接批量写切片正文。
- Step 阶段不得一次性生成全部切片；超大文档每批最多 2 片，普通文档每批最多 5 片。
- Poll 阶段完成度只能按“已通过校验切片数 / 计划切片数”计算，未校验切片不得计入完成。
- 出现 Token 溢出、索引跳号、死链或检索死片时，必须先 Poll 当前状态，再回到对应 Step 修复。
- 最终交付前必须执行 `batch_audit.py` 和 `evaluate_retrieval.py`，并输出审计/检索验证证据。

## Goal

作为 RAG/知识库管线的核心 ETL 引擎，将长文档拆解为**语义完整、物理隔离、自包含、检索就绪**的 L0 级原子切片。核心使命：在严格执行目录隔离的同时，确保切片既是独立的检索单元，又是原文语义的无损投影。

## ⛔ 核心红线 (Critical Constraints)

1. **目录绝对隔离**：每个源文件必须生成同名专属文件夹。严禁任何切片文件直接存在于输出根目录。
2. **语义完整性优先于长度限制**：当字数熔断点位于代码块、表格、步骤列表或逻辑论证中间时，**必须**延后至当前语义单元结束处切分，禁止截断原子逻辑。
3. **禁止破坏性改写**：自包含转换仅允许"补充"和"显式化"，严禁删除原文技术参数、修改代码逻辑或替换专业术语。
4. **零指代原则**：切片内不得出现未定义的 `this`, `it`, `上述`, `如下` 等依赖外部上下文的指代词。
5. **废弃内容显式标记**：检测到废弃声明后必须在正文首段前强制插入 `> [!CAUTION]` 警告，并在 YAML 中标记 `status: deprecated`。若版本号未知，写"某个版本"并标记 `human_review_required: true`。
6. **禁止脚本模板生成元数据**：`embedding_hint`、`qa_pairs`、`hybrid_keywords`、`tags` 等语义元数据必须由 Agent 自身推理生成。
7. **禁止外部 LLM API**：禁止调用任何外部 LLM API 来生成切片内容、摘要或元数据。
8. **网络驱动器路径约束**：涉及网络驱动器（如 X: 盘、E:\Marvis_Data）上的文件读写时，优先使用 `python_executor`。
9. **语义保真原则**：关键数值/参数名/错误码/配置项等事实数据必须原样出现在切片正文中；隐含假设须显式化；因果链须标注前置条件。
10. **系统破坏操作禁令 (HARB)**：对工作目录外任意路径的批量写操作、`rm -rf`/`del /f /s` 等递归强制删除、git destructive 操作、数据库 DROP/TRUNCATE 等命令，**必须输出完整命令预览并等待用户显式确认**。

## Prerequisites (环境依赖)

| 依赖项                | 最低版本 | 安装命令                            | 说明                                                                      |
| :-------------------- | :------- | :---------------------------------- | :------------------------------------------------------------------------ |
| Python                | 3.8      | —                                   | 全平台通用                                                                |
| sentence-transformers | —        | `pip install sentence-transformers` | 检索评估（§8），首次运行自动下载 ~90MB；缺失时自动降级 PurePythonEmbedder |
| numpy                 | —        | `pip install numpy`                 | 数值计算支撑                                                              |
| pyyaml                | —        | `pip install pyyaml`                | YAML 切片元数据解析                                                       |

一键安装：`pip install sentence-transformers numpy pyyaml`

---

## 0. 预处理思维链 (Pre-computation CoT) [强制执行]

在生成任何切片文件之前，**必须**先在 `<analysis>` XML标签内完成以下推理（此标签内容不计入最终切片输出）：

**核心四步（必须完成）**：

1. **结构扫描 + 原子单元标记**：列出所有 H1-H3 标题并识别不可分割的代码块/表格/Mermaid图/LaTeX公式，标记其起止位置。超长代码块（>50行）判断是否可按方法/分支再拆分。
2. **Token 预算 + 熔断预演**：中文 1 字≈1.5 Token、代码 1 字符≈0.3 Token、英文 1 词≈1.3 Token。按差异化软上限（纯文本 700 字/代码 500 字/混合加权计算）模拟切分，检查是否命中原子单元。
3. **目录决策 + 版本检测**：确定二级目录结构（api/config/guide/concepts/misc/multimodal）、降级策略应用结果及完整文件名列表。扫描 deprecated/version 关键词，规划警告标记位置。
4. **可用性预检 + 消歧预检**：逐切片预判独立检索场景下的可用性（上下文自足/示例可执行/边界情况），识别同名冲突、语义重叠、泛化词陷阱并规划消歧前缀。

> ⚠️ 未完成 `<analysis>` 直接输出切片视为严重违规。
>
> 🛑 **STOP CHECKPOINT**：`<analysis>` 完成后**必须暂停**，将分析结果摘要输出给用户确认。确认后开始生成切片文件。禁止跳过确认直接进入批量生成。

> 💡 完整 10 步 CoT 细节（含命令有效性预检、Token 估算公式等）见 [REFERENCE.md §1-§2](REFERENCE.md)。

---

## 1. 动态目录与路由契约 (Directory & Routing Contract)

### 1.1 源文件指纹与元数据继承

- **Source-ID**：取文件名（不含后缀）作为唯一标识。
- **强制动作**：创建 `{Source-ID}/` 文件夹作为所有输出的根容器。
- **元数据继承**：从源文件 Frontmatter 提取 `version`, `author`, `last_updated`，注入到每个切片的 YAML 头中。

### 1.2 二级目录智能分类

在 `{Source-ID}/` 内部按内容属性路由：`api/`（函数签名/类定义）、`config/`（配置项说明，与 api/ 互斥）、`guide/`（操作指南/故障排查）、`concepts/`（架构原理/术语定义）、`misc/`（Changelog/FAQ/附录）、`multimodal/`（多模态元素 ≥5 且占比 ≥25% 时创建）。

**动态降级规则**：某分类下切片数 ≤2 且总字数 <1500 字 → 合并至根目录，文件名保留分类前缀；缓冲区 1300~1700 字按切片数和各片字数综合判断。单切片不建子目录。完整降级策略（含防抖动、结构稳定性）见 [REFERENCE.md §1](REFERENCE.md)。

### 1.3 版本演进处理

- 同一 Source-ID 下存在多版本时，自动生成 `version_matrix.md` 索引切片（含版本差异摘要表）。
- 源文件缺失或不可读 → 立即终止，明确报错，禁止凭空生成。

---

## 2. 智能拆分触发器 (Smart Splitting Triggers)

采用 **"语义优先，字数兜底，回溯保护"** 三重判定机制。

| 优先级 | 触发条件             | 执行动作                                                                                                       |
| :----- | :------------------- | :------------------------------------------------------------------------------------------------------------- |
| P0     | 独立 API/类/配置项   | 立即切分为独立文件。超长示例代码（>40行）独立为 `{prefix}_examples.md` 切片                                    |
| P1     | 并列结构 (Step/List) | 每个顶层步骤/条目独立成切片                                                                                    |
| P2     | 语义段落自然边界     | 在段落/章节交界处切分，禁止在句子中间切分                                                                      |
| P3     | 字数软上限触发       | 寻找最近语义边界切分。**回溯保护**：若边界在代码块/表格内，回溯至该块起始处；回溯后 >1500 字则整体移至下一切片 |

字数软上限：纯文本 700 字 / 代码 500 字+40 行 / 混合内容加权 `(文本字数×1.0 + 代码字数×1.4 + 表格单元格数×0.3) / 内容段数`。

> 多模态切分策略、自包含转换 8 条完整规则见 [REFERENCE.md §1-§2](REFERENCE.md)。

---

## 3. 标准化输出格式 (Standard Output Format)

每个切片必须严格遵循 YAML + Markdown 模板。14 个必填字段：`title`, `source_id`, `category`, `index`, `version`, `status`, `tags`, `embedding_hint`, `structural_context`, `hybrid_keywords`, `cross_refs`, `qa_pairs`, `multimodal_refs`, `human_review_required`。正文必须覆盖：概念定义 → 使用场景 → 核心内容 → 注意事项 → 相关链接。

> 完整模板及元数据字段精化规则（含 embedding_hint ≤200字 + 实体名、structural_context 格式、hybrid_keywords 泛化词排除、qa_pairs 唯一性验证）见 [REFERENCE.md §3](REFERENCE.md)。

---

## 4. 输出后自检协议 (Post-Generation Self-Correction)

每个切片写入磁盘后，**必须立即**执行校验。命中任何 🔴 致命项 → **立即重生成该切片**直至通过。

### 4.1 YAML Frontmatter 完整性强制校验 [最高优先级]

重新读取刚写入的文件，检查 14 个必填字段是否全部存在。若检测到外部水印覆盖标准 YAML 块，**必须强制追加第二个 `---` 块**写入完整标准元数据。校验失败立即重写，禁止延迟处理。

### 4.2 阻断式分级自检

| 级别    | 含义                   | 处理方式                                     |
| :------ | :--------------------- | :------------------------------------------- |
| 🔴 致命 | 检索或语义完整性被破坏 | **立即停止当前批次，重生成该切片**           |
| 🟡 警告 | 影响质量但不致命       | 标记 `human_review_required: true`，允许继续 |
| 🟢 建议 | 规范合规性             | 记录但允许通过                               |

**🔴 致命项（必须重生成）**：目录隔离违规 | 语义完整被破坏（代码块/表格/列表截断） | 零指代违规 | 事实保真失败（原文关键数值/参数/错误码未被保留） | YAML 14 字段缺失。

> 完整检查项（含 🟡 警告 10 项、🟢 建议 3 项、自检注释格式）见 [REFERENCE.md §4](REFERENCE.md)。

---

## 5. 分批处理协议 (Token Budget Awareness)

当预估切片数 > 10 或总 Token 接近上下文窗口 70% 时触发分批。

> 🔴 **CHECKPOINT**：触发分批后**必须暂停**，先输出 `<analysis>` 摘要和切片清单预览，等待用户确认后再逐批（每批 3~5 个切片）生成。

每批结束后强制追加批审计注释（索引连续性/孤立切片/大小异常），全部批次完成后强制执行终结校验（索引连续性/cross_refs 有效性/废弃命令检测）。终结校验不通过不得声称任务完成。

---

## 6. 切片完整可用性契约 (Slice Completeness & Usability Contract)

每片生成后逐项自检五维（必须全部 ✅）：**上下文自足性**（模块锚定/术语本地定义）、**示例可执行性**（import 补全/I/O 标注/环境声明）、**知识层级**（五段完整）、**负面信息**（限制/边界/陷阱显式化）、**检索信号**（qa_pairs 覆盖 how-to/what-is/troubleshooting 三类意图）。

> 完整检查表格（含合规/违规示例）见 [REFERENCE.md §4](REFERENCE.md)。

---

## 7. 语义保真与检索精度契约 (Semantic Fidelity & Retrieval Precision)

| 维度     | 检查项                                                         | 失败特征                                      |
| :------- | :------------------------------------------------------------- | :-------------------------------------------- |
| 事实保留 | 数值/参数/错误码/路径/命令逐项与源文件核对                     | YAML `fact_violations` 非空                   |
| 隐含假设 | 环境/权限/前置依赖/默认值/互斥关系五类均已显式声明             | 正文含模糊限定词（"一般"/"通常"）但未标注条件 |
| 因果链   | 标注规则：起点（触发条件）→ 中间（转换逻辑）→ 终点（预期结果） | 缺少任一环节导致步骤断链                      |
| 检索消歧 | 命名空间前缀/参数特征/版本标记/场景限定均已嵌入                | 同名概念跨切片 keyword 相同                   |
| 精度自检 | `batch_audit.py` 报告同名冲突=0、语义重叠=0、泛化词残留=0      | 冲突数 > 0 或泛化词未净化                     |

> 完整验证规则（含事实类型检测方法、隐含假设标注格式、消歧策略表）见 [REFERENCE.md §5](REFERENCE.md)。

---

## 8. 复用脚本体系 (Reusable Scripts)

> 🩺 **症状→脚本速查**（无需通读全文）：

| 症状                            | 运行                                                           | 章节 |
| :------------------------------ | :------------------------------------------------------------- | :--: |
| 单切片 YAML 缺字段 / 内容不规范 | `python scripts/validate_slice.py <file> --fix --json`         | §8.2 |
| 批量切片索引不连续 / 死链       | `python scripts/batch_audit.py <dir> --fix`                    | §8.3 |
| 需要生成切片计划 JSON / stub    | `python scripts/slice_generator.py <src> <out> --stubs --json` | §8.4 |
| 检索评估：想知道 R@1/R@5/MRR    | `python scripts/evaluate_retrieval.py <dir> --fix`             | §8.5 |
| 全局索引重排（插入/删除切片后） | `python scripts/slice_generator.py --renumber <dir>`           | §8.4 |
| 不确定当前环境平台/模型能力     | `python scripts/platform_detect.py`                            | §8.6 |
| 计划中断，需要恢复              | `python scripts/slice_generator.py <src> <out> --resume`       | §8.4 |

标准工作流：`slice_generator.py --json` → Agent 逐切片填充 → `validate_slice.py --fix`（每片）→ `batch_audit.py`（每批/全部）→ `evaluate_retrieval.py --fix`（终检）。

> 各脚本详细用法（参数说明、调用时机、输出格式）见 [REFERENCE.md §6](REFERENCE.md)。

---

## 9. 使用指南与能力边界 (User Guide & Scope)

**用户操作**：说"把 xxx.md 切片"即可。Agent 分析 10~30s → 确认计划 → 逐批生成 → 验收结果。支持增量追加、先看计划、从 docx 开始等常见操作。

**能力边界**：支持 md/txt/yaml/json/PDF，不支持图片直接切片/数据库/动态网页。完整用户指南（含实操案例、FAQ）见 [REFERENCE.md §7](REFERENCE.md)；能力边界细节见 [REFERENCE.md §9](REFERENCE.md)；更新记录见 [REFERENCE.md §11](REFERENCE.md)。

---

## 10. 异常处理 (Error Handling)

所有错误信息必须遵循三段式：`[类型]: 发生了什么 → 原因: 为什么发生 → 操作: 如何修复`。

| 错误类型     | 症状                             | 处理方式                                                |
| :----------- | :------------------------------- | :------------------------------------------------------ |
| 源文件不可读 | 切片计划为空、文件不存在或无权限 | 验证路径可达性；网络驱动器走 `python_executor`          |
| 切片数异常   | 实际生成数与计划不符             | `slice_generator.py --json` 获取修正计划                |
| 校验致命失败 | `validate_slice.py` 返回 FAIL    | `--fix --json` 输出完整日志，定位后重生成               |
| 索引不连续   | `batch_audit.py` 报告 index 跳跃 | 列出缺失索引，`--resume` 断点续传                       |
| 死链         | `cross_refs` 引用不存在          | `batch_audit.py --fix` 自动检测并修复                   |
| 依赖缺失     | `ModuleNotFoundError`            | `pip install` 后重试；评估层自动降级 PurePythonEmbedder |
| 检索死片     | R@1=0                            | 重写 `embedding_hint` 加入正文精确关键词                |
| Token 溢出   | 生成中断、上下文超限             | 分批模式，每批 ≤2 片                                    |
| 编码错误     | 切片中文变乱码                   | 检测实际编码后重新以 UTF-8 另存                         |

> 完整异常处理（含静默失败防护清单、通俗错误速查指南、精确处理命令）见 [REFERENCE.md §12](REFERENCE.md)。

---

## 11. 反模式 (Anti-Patterns)

| #   | 禁止行为                                         | 正确做法                                                          |
| :-- | :----------------------------------------------- | :---------------------------------------------------------------- |
| 1   | 跳过 `<analysis>` 直接生成切片                   | 必须完成 §0 CoT 并通过 CHECKPOINT 确认                            |
| 2   | 用脚本模板生成元数据（语义空洞导致检索命中率低） | Agent 逐片手工填充 embedding_hint/qa_pairs/hybrid_keywords        |
| 3   | 未完成 analysis 就写文件 / 单切片塞全部内容      | 分批协议 + §2 拆分阈值                                            |
| 4   | 忘记校验（低质量切片流入下游）                   | 每片写入后立即 `validate_slice.py --fix`，每批后 `batch_audit.py` |
| 5   | 假设阅读顺序（切片被跨顺序检索时上下文断裂）     | 每片自包含：术语定义 + 前置依赖声明 + 版本号                      |

> 完整 13 条反模式清单（含症状诊断、危害说明）及 FAQ 见 [REFERENCE.md §8](REFERENCE.md)。

---

## 12. 平台兼容性 (Platform Compatibility)

Windows / Linux / macOS 三平台完全支持。关键约束：网络驱动器用 `python_executor`；`evaluate_retrieval.py` 三层嵌入降级路由（SBERT → 自动安装 → PurePythonEmbedder）；所有脚本使用 `pathlib.Path`。详见 [REFERENCE.md §10](REFERENCE.md)。
