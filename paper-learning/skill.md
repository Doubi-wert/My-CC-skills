# Paper Learning Skill

## 概述

本skill基于输入的文献MD文件生成教学文档，结合迭代批判确保内容准确性和引用一致性。采用**章节级 + 整体级**双层批判迭代机制，覆盖前向引用一致性检查。

## 角色定义

- **G (Generator)**: 内容生成者，基于参考文档生成高质量内容
- **B (Critic)**: 批判性审查者，从批判角度审查生成内容是否符合参考文档
- **H (Harmonizer)**: 协调者，负责版本追踪、状态管理、收敛判定、人工介入触发

## 双层批判迭代架构

### 层级定义

| 层级 | 标签 | 范围 | 迭代单位 |
|------|------|------|---------|
| Macro | `macro` | 整体文档级别 | 全局一致性、前向引用、术语统一 |
| Meso | `meso` | 章节/模块级别 | 单章内容质量、章节内部逻辑 |
| Micro | `micro` | 段落/局部级别 | 局部表述、公式引用、细节准确 |

### 层级边界条件

#### 章节级迭代 (Meso层)

**触发条件**:
- 任意章节首次进入处理流程
- 前一轮整体级迭代标记需要章节级重处理

**退出条件** (满足任一即可):
1. 反馈点连续2轮 < threshold，且B确认章节完整性
2. 迭代次数达到 max_chapter_iterations (默认3次)
3. 人工介入触发并选择重置章节

**嵌套约束**:
- 章节级迭代不允许嵌套其他章节级迭代
- Micro级问题在章节级迭代内处理，不单独建层

#### 整体级迭代 (Macro层)

**触发条件**:
- 所有章节级迭代均已收敛（或达到最大章节迭代次数）
- 显式请求整体一致性检查

**退出条件** (满足任一即可):
1. 反馈点 = 0 且 B确认整体完整性
2. 迭代次数达到 max_global_iterations (默认2次)
3. 跨章节反馈连续2轮 < threshold

**嵌套约束**:
- 整体级迭代期间可触发章节级重处理（标记后在下轮执行）
- 不允许嵌套其他整体级迭代

### 跨Scope反馈聚合规则

| 反馈Scope | 聚合优先级 | 处理时机 |
|-----------|-----------|---------|
| Global | 最高 | 整体级迭代优先处理 |
| Chapter | 中 | 章节级迭代处理，宏观问题标记上浮 |
| Local | 最低 | 章节级迭代内处理，汇聚为章节级反馈 |

**上浮规则**: 当Local级反馈在同一章节累计 >= 3个时，自动汇聚为Chapter级反馈；当Chapter级反馈跨章节问题一致时，上浮为Global级。

## 角色接口协议

### H协调者接口定义

#### 章节级迭代接口

**输入**:
```json
{
  "chapter_id": "string",
  "chapter_content": "string",
  "chapter_version": "vN",
  "previous_feedback": [],
  "deferred_from_previous": []
}
```

**输出**:
```json
{
  "chapter_id": "string",
  "converged": true|false,
  "version": "vN",
  "feedback_summary": {},
  "next_action": "continue|escalate_to_global|request_human"
}
```

#### 整体级迭代接口

**输入**:
```json
{
  "all_chapters": [{"chapter_id": "string", "version": "vN", "converged": true}],
  "cross_chapter_feedback": [],
  "version_history": []
}
```

**输出**:
```json
{
  "converged": true|false,
  "global_version": "vN",
  "requires_chapter_revisit": ["chapter_id", ...],
  "next_action": "finalize|request_human"
}
```

### 状态同步机制

H维护以下状态变量，在角色间传递时保持同步:

| 状态变量 | 类型 | 说明 |
|---------|------|------|
| `chapter_states` | Map | 各章节当前迭代状态 |
| `global_iteration` | int | 整体迭代计数 |
| `pending_cross_scope` | List | 待处理的跨Scope反馈 |
| `deferred_registry` | Map | 所有deferred问题的追踪 |

状态变更时，H通过中间传递格式同步给G和B。

## 反馈点结构

```json
{
  "feedback_points": [
    {
      "id": 1,
      "location": "具体位置",
      "issue": "问题描述",
      "suggestion": "修改建议",
      "severity": "critical|high|medium|low",
      "scope": "global|chapter|local",
      "layer": "macro|meso|micro"
    }
  ],
  "count": N,
  "summary": "总体评价",
  "has_critical_error": true|false
}
```

### Severity分级定义

| 等级 | 标签 | 定义 | 处理策略 |
|------|------|------|---------|
| Critical | `critical` | 关键需求违背、安全问题、格式完全错误 | 立即处理，阻塞式 |
| High | `high` | 严重影响功能的问题 | 本轮优先处理 |
| Medium | `medium` | 功能正常但有改进空间 | 本轮处理，延后也可 |
| Low | `low` | 小的优化建议 | 标记为deferred，后续轮次处理 |

### Scope分级定义

| 等级 | 标签 | 定义 |
|------|------|------|
| Global | `global` | 影响整个文档的问题 |
| Chapter | `chapter` | 影响单个章节的问题 |
| Local | `local` | 仅影响局部内容的问题 |

## 收敛标准

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 最大迭代次数 | 5次 | 防止无限循环 |
| 最大章节迭代次数 | 3次 | 单章最大迭代 |
| 最大整体迭代次数 | 2次 | 整体级最大迭代 |
| 反馈点阈值 | 3个/轮 | 单轮超过此值则继续迭代 |
| 连续低阈值轮次 | 2轮 | 连续2轮反馈点<阈值则退出 |
| 单轮最大处理反馈数 | 5个 | 超出部分自动deferred到下一轮 |

### 收敛判定逻辑

```
// 章节级收敛判定
if chapter_iteration >= max_chapter_iterations:
    退出章节迭代，标记为"max_iterations_reached"
elif feedback_points == 0:
    B确认审查完整性
    if 完整:
        下一轮仍为0则退出
    else:
        要求重新审查，继续迭代
elif feedback_points < 阈值:
    低阈值轮次 += 1
    if 低阈值轮次 >= 2:
        退出
else:
    低阈值轮次 = 0

// 整体级收敛判定
if global_iteration >= max_global_iterations:
    退出整体迭代
elif cross_chapter_feedback_points == 0:
    B确认整体完整性
    if 完整:
        下一轮仍为0则退出
elif cross_chapter_feedback_points < 阈值:
    if 低阈值轮次 >= 2:
        退出
else:
    低阈值轮次 = 0
```

### 快速收敛路径 (Feedback = 0)

当反馈点为0时:
1. **B审查完整性确认**: B需确认审查是否完整
   - 如果审查完整: 进入下一轮
   - 如果审查不完整: B要求重新审查，确认后方可进入下一轮
2. **下一轮仍为0**: 退出迭代

### 标准收敛路径 (连续2轮低于阈值)

当反馈点连续2轮低于阈值时:
1. H评估是否已达到可接受质量标准
2. 如果达标: 结束迭代
3. 如果未达标: 继续迭代

## 关键机制

### 1. 版本追踪

版本号格式: `v{iteration}`

- 例: v1, v2, v3
- 每个版本必须带版本号标识
- 最终输出包含版本演进记录表

### 2. Deferred机制

#### 定义

Deferred是指暂时搁置、需要后续处理的问题。

#### 传递机制

- 在中间传递格式中增加 `deferred_points` 字段
- 该字段记录本轮deferred的问题列表
- 每轮迭代时传递 `previous_deferred_count`

#### 强制升级与降级规则

**强制升级 (Forced Upgrade)**:
- 触发条件: 连续deferred次数达到 max_consecutive_deferred (默认3次)
- 效果: 该问题必须在本轮处理，不再允许deferred

**降级回退 (Degradation Fallback)**:
- 触发条件: 强制升级的问题在本轮处理失败（无法解决/引入新错误）
- 规则: 最多重试 N 次（max_upgrade_retries = 2）后降级为"静默接受"
- 降级后处理: 问题标记为 `degraded_accepted`，记录但不阻塞迭代

```
if issue is deferred:
    consecutive_deferred_count++
    if consecutive_deferred_count >= max_consecutive_deferred:
        // 强制升级
        attempt_process(issue)
        if 处理失败:
            upgrade_retry_count++
            if upgrade_retry_count >= max_upgrade_retries:
                // 降级为静默接受
                mark_as_degraded_accepted(issue)
            else:
                // 重试
                retry_process(issue)
        else:
            // 处理成功
            resolve_issue(issue)
    else:
        // 允许deferred
        add_to_deferred_list(issue)
```

#### Deferred状态机

```
                    ┌─────────────────────────────────────┐
                    │                                     │
                    ▼                                     │
[New] ──► [Deferred] ──► [Forced_Upgrade] ──► [Resolved] │
                    │                    │               │
                    │                    ▼               │
                    │              [Retry]               │
                    │                    │               │
                    │                    ▼               │
                    │             [Failed_After_Retries]  │
                    │                    │               │
                    │                    ▼               │
                    └────── [Degraded_Accepted] ◄────────┘
```

### 3. G拒绝反馈的合理性判定

#### 拒绝合理性判定流程

当G拒绝某条反馈时:

1. **G提供拒绝理由**: 必须说明拒绝的具体原因
2. **B二次确认**:
   - B评估G的拒绝理由是否充分
   - 如果理由充分: 标记为"有效拒绝"，反馈关闭
   - 如果理由不充分: 标记为"无效拒绝"，要求G重新处理

**拒绝理由合理性判定原则**:
- 参考文档中无相关依据
- 建议与任务目标不符
- 建议会导致内容逻辑矛盾

G不应因"实现困难"或"个人偏好"拒绝反馈。

#### 拒绝判定与Deferred状态联动

| 场景 | Deferred状态 | 后续动作 |
|------|-------------|---------|
| 拒绝合理 + 原反馈为deferred | 清除deferred标记 | 反馈关闭，不计入迭代 |
| 拒绝合理 + 原反馈非deferred | 不适用 | 反馈关闭 |
| 拒绝不合理 + 原反馈为deferred | 保持deferred | 本轮强制升级处理 |
| 拒绝不合理 + 原反馈非deferred | 不适用 | 要求G重新处理，连续3次无效触发人工介入 |

### 4. 同一问题判定标准

判断是否为"同一问题"的依据:

- **语义相似度**: 基于 (issue描述 + location) 的语义进行判定
- **判定因素**:
  - 问题描述的核心内容是否相同
  - 问题发生的位置是否相同
- **判定方式**: 非精确字符串匹配，而是语义相似度匹配
- **合并处理**: 同一问题的多个反馈仅计为1个反馈点

### 5. 无参考文档模式

当用户未提供参考文档时，系统基于初始输入的任务目标进行批判迭代。此时:
- G的生成内容需符合任务目标
- B的评判标准基于任务目标本身（而非外部文档）
- 迭代收敛后输出符合任务目标的高质量内容
- **系统会明确告知用户当前处于无参考文档模式**

#### 无参考文档模式的质量基准建立

在无外部参照时，质量基准采用**自举基准 (Bootstrap Baseline)** 方法:

| 阶段 | 方法 | 说明 |
|------|------|------|
| 初始基准 | 任务目标分解 | 从任务目标提取核心质量维度，建立初始期望 |
| 过程基准 | 历史均值 | 记录前3轮迭代的质量分数，均值作为基准 |
| 动态调整 | 移动窗口 | 每轮迭代后更新基准（最近3轮均值） |
| 终止判断 | 基准收敛 | 连续2轮质量分数波动 < threshold * 0.5 |

### 6. 人工介入机制

#### 触发条件形式化定义

人工介入触发条件采用**规则型 + 阈值型**混合定义:

| 触发条件 | 类型 | 形式化定义 | 阈值 |
|---------|------|-----------|------|
| 重复反馈 | 阈值型 | `count(same_feedback_id) >= threshold` | >= 3次 |
| Deferred超限 | 阈值型 | `consecutive_deferred_rounds > threshold` | > 2轮 |
| 无效拒绝 | 阈值型 | `count(invalid_rejection) >= threshold` | >= 3次 |
| 质量下降 | 阈值型 | `quality_score_drop > threshold` | 下降 > 0.15 |
| 配置冲突 | 规则型 | `static_config modified in iteration` | 即时触发 |
| 循环依赖 | 规则型 | `chapter_A waits chapter_B AND chapter_B waits chapter_A` | 即时触发 |

#### 触发条件详解

| 触发条件 | 阈值 | 说明 |
|---------|------|------|
| 重复反馈 | >= 3次 | 同一反馈点被重复提出 |
| Deferred超限 | > 2轮 | 同一问题连续deferred轮次 |
| 无效拒绝 | >= 3次 | G拒绝反馈但理由不充分 |
| 质量下降 | 超过阈值 | 质量分数下降幅度超过阈值 |

#### 处理流程

1. 暂停迭代
2. 向用户说明当前状态和待决策事项
3. 等待用户指示
4. 根据指示继续迭代或终止

### 7. 参考文档自动识别

系统自动识别用户提供的文件路径，识别后**必须经过用户确认**才可使用:

1. 当前工作目录下的相对路径
2. 用户明确指定的绝对路径
3. 常见的参考文档格式: .md, .txt, .pdf, .doc, .docx

### 8. 质量追踪

通过 `user_requirements.quality_tracking_mode` 配置:

| 模式 | 说明 |
|------|------|
| disabled | 不启用质量追踪 |
| relaxed | 仅当整体分数下降 > threshold * 1.5 时暂停 |
| strict | 整体或任一维度分数下降 > threshold 时暂停 |

阈值通过 `user_requirements.quality_threshold` 配置（默认0.1）。

## 中间传递格式

```json
{
  "current_version": "vN",
  "current_content": "待审查的内容",
  "layer": "macro|meso|micro",
  "feedback_points": [
    {
      "id": 1,
      "location": "具体位置",
      "issue": "问题描述",
      "suggestion": "修改建议",
      "severity": "critical|high|medium|low",
      "scope": "global|chapter|local"
    }
  ],
  "feedback_count": N,
  "deferred_points": [
    {
      "id": 6,
      "reason": "low_priority",
      "deferred_to": "round_3",
      "consecutive_count": 0,
      "upgrade_retry_count": 0,
      "status": "deferred|forced_upgrade|degraded_accepted"
    }
  ],
  "previous_deferred_count": 0,
  "consecutive_low_feedback": 0,
  "rejected_feedbacks": [
    {"id": 2, "count": 1, "reason": "...", "validated": false}
  ]
}
```

## 最终输出格式

```markdown
# [任务标题]

## 内容
[最终生成的文档]

## 修改记录
| 版本 | 迭代 | 主要修改点 | B反馈数 |
|------|------|-----------|--------|
| v1 | 1 | [描述] | N |
| v2 | 2 | [描述] | N |
| ... | ... | ... | ... |

## 最终评估
[是否符合参考文档，总体评价]

## 统计信息
- 总迭代次数: N
- 最终版本: vN
- 最终反馈点: N
- Deferred总数: N
- 降级接受数: N
```

## G代理Prompt模板

```markdown
# 角色
你是一个内容生成者(G)，基于参考文档生成高质量的内容。

# 输入
- 参考文档内容: [提取后的关键部分]；无文档时为任务目标
- 任务描述: [需要生成什么]
- 当前层级: [macro/meso/micro]
- 上一轮B的反馈: [如有]
- Deferred反馈: [从之前轮次推迟的反馈点，需在本轮处理]

# 输出要求
生成修订后的内容，并附上修改说明。

# 行为约束
1. 必须逐条回应B的反馈，说明接受或拒绝
2. 拒绝某条反馈时需给出理由（参考文档依据/任务目标不符/逻辑矛盾）
3. 不要因"实现困难"或"个人偏好"拒绝反馈
4. 修改后明确说明做了哪些改动
5. Deferred反馈必须在本轮处理，不能无限推迟
6. 每个版本必须带版本号标识（v1, v2, ...）
7. 强制升级的问题必须在本轮处理，不允许继续defer

# 输出格式
## 版本
[vN]

## 修订内容
[生成的内容]

## 修改说明
1. [反馈1] → 已接受/已拒绝（理由）
2. [反馈2] → 已接受/已拒绝（理由）
3. [Deferred反馈] → 已处理/强制升级处理/降级接受（理由）
```

## B代理Prompt模板

```markdown
# 角色
你是一个批判性审查者(B)，从批判角度审查生成内容是否符合参考文档。

# 输入
- 参考文档内容: [提取后的关键部分]；无文档时为任务目标本身
- G生成的内容: [待审查内容]
- 当前层级: [macro/meso/micro]
- 上一轮G对反馈的回应: [用于检查是否正确修订]
- 已拒绝反馈列表: [feedback_id, ...] - 这些反馈已被G拒绝，拒绝理由合理的不要重复提出

# 输出要求
给出具体的修改点列表，明确指出问题所在和位置。

# 判断原则
1. 基于参考文档的具体内容，不使用主观偏好
2. 无参考文档时，基于任务目标本身进行评判
3. 同一问题只反馈一次，基于(issue描述+location)的语义相似度判定
4. 每条反馈需附带修改建议
5. 反馈需具体可操作，非泛泛评价
6. 对于已被拒绝的反馈，如果拒绝理由合理，不应重复提出
7. **当反馈点为0时，需确认审查完整性**
8. 强制升级的问题若G已处理，验证处理是否有效

# 输出格式
```json
{
  "feedback_points": [
    {
      "id": 1,
      "location": "具体位置",
      "issue": "问题描述",
      "suggestion": "修改建议",
      "severity": "critical|high|medium|low",
      "scope": "global|chapter|local"
    }
  ],
  "count": N,
  "summary": "总体评价",
  "has_critical_error": true|false,
  "deferred_points": [
    {"id": 6, "reason": "low_priority", "deferred_to": "round_3"}
  ],
  "rejected_reasons_validated": [1, 3]
}
```
```

## 严重错误定义

| 错误类型 | 触发条件 | 处理方式 | 响应动作 |
|---------|---------|---------|---------|
| CE-1: 格式完全错误 | 输出格式与要求完全不匹配 | 立即退出 | 记录错误，输出当前版本，终止迭代 |
| CE-2: 关键需求违背 | 标记为critical的需求未满足 | 立即退出 | 记录违背详情，输出当前版本，终止迭代 |
| CE-3: 安全漏洞 | 检测到注入攻击等安全问题 | 立即退出 | 记录安全日志，输出当前版本，终止迭代 |
| CE-4: 编译/解析错误 | 代码无法编译或文档无法解析 | 立即退出 | 记录错误位置，输出当前版本，终止迭代 |
| CE-5: 死循环/超时 | 单次操作超过timeout阈值 | 立即退出 | 记录超时位置，输出当前版本，重置该章节状态 |

## 配置参数

### 静态配置 (全局锁定)

静态配置在迭代过程中不可修改，必须在开始前确定:

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `max_iterations` | 5 | 全局最大迭代次数 |
| `max_chapter_iterations` | 3 | 单章最大迭代次数 |
| `max_global_iterations` | 2 | 整体级最大迭代次数 |
| `quality_tracking_mode` | relaxed | 质量追踪模式 |
| `scope_layer_def` | 见层级定义 | Scope维度层级体系 |

### 动态配置 (迭代内可变)

动态配置可在迭代过程中修改:

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `threshold` | 3 | 反馈点阈值 |
| `max_consecutive_deferred` | 3 | 连续deferred最大次数 |
| `max_upgrade_retries` | 2 | 强制升级最大重试次数 |
| `convergence_rounds` | 2 | 连续低于阈值的轮数 |
| `quality_threshold` | 0.1 | 质量分数下降阈值 |

### 配置冲突解决

当动态配置修改与静态配置冲突时:
- **规则**: 静态配置优先，动态配置不得突破静态配置的限制
- **示例**: 动态配置不能将 max_chapter_iterations 从3提升到5（静态配置）
- **检测**: H在应用配置修改前检测冲突，有冲突时拒绝修改并警告

## 关键原则

1. **章节级+整体级双层批判**: 先逐章迭代，再整体验证
2. **前向引用一致性**: 检查章节间的引用关系是否一致
3. **版本追踪**: 每个版本必须带版本号，便于追踪内容演进
4. **可追溯性**: G必须对每条反馈做出回应（接受/拒绝+理由）
5. **收敛保证**: 多层防护防止无限循环
6. **同一问题判定**: 基于语义相似度，非精确字符串匹配
7. **拒绝合理性判定**: G拒绝反馈后，B需判定拒绝理由是否充分
8. **Deferred降级机制**: 强制升级失败后最多重试2次，然后降级为静默接受
9. **配置层级隔离**: 静态配置全局锁定，动态配置迭代内可变

## 工作流程

### 阶段1: 论文解析

1. **检查参考文档**: 判断是否有参考文档输入
   - **有参考文档**: 验证文件存在，提取关键章节、公式、定理、引用
   - **无参考文档**: 基于任务目标/初始输入作为评判标准，**明确提示用户当前处于无参考文档模式**
2. **提取关键内容**:
   - 章节结构
   - 核心公式与定理
   - 关键引用关系
   - 算法步骤
3. **分解任务**: 将论文内容分解为可独立处理的章节/模块

### 阶段2: 章节级批判迭代 (每章独立G-B循环)

对每个章节独立执行迭代批判:

```
循环 while (chapter_iteration <= max_chapter_iterations) {
    1. [G] 生成/修订章节内容
    2. [B] 批判性对比参考文档，输出反馈点
    3. [H] 评估反馈，执行收敛检查:
       a. 如果 feedback_points = 0：
          - B 确认审查完整性
          - 确认后进入下一轮
       b. 如果 feedback_points < threshold（连续2轮）：
          - H 评估是否达到收敛标准
          - 达标则结束该章节迭代
       c. 否则：继续迭代
    4. [G] 根据反馈修订输出
}
```

### 阶段3: 整体批判迭代 (前向引用一致性检查)

在所有章节级迭代完成后，执行整体一致性检查:

1. **前向引用验证**: 检查章节间的引用关系是否一致
2. **术语一致性**: 检查关键术语在不同章节中的使用是否统一
3. **逻辑连贯性**: 检查章节间的逻辑衔接
4. **整体反馈**: 对整体文档进行批判，给出跨章节问题

### 阶段4: 输出生成

多文件拆分输出:
- 每个章节独立文件
- 整体文档汇总文件
- 修改记录文件