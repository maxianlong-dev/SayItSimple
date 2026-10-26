# Agent 自进化（Self-Evolving Agent）
#
Agent 自进化指 AI Agent 在运行过程中**自主积累经验、优化策略、提升能力**，而非依赖人工迭代 prompt 或代码。
## 核心思路
```plain text
传统 Agent：固定 prompt + 固定工具 → 每次从零开始
自进化 Agent：经验积累 + 策略调整 → 越用越强
```
## 自进化的三个层次
### 1. 记忆进化（Memory Evolution）
从交互中提取可复用知识，下次直接利用。
```plain text
┌──────────┐     ┌──────────┐     ┌──────────┐
│ 短期记忆  │ ──→ │ 经验提炼  │ ──→ │ 长期记忆  │
│ (对话上下文)│    │ (反思总结) │     │ (持久化)  │
└──────────┘     └──────────┘     └──────────┘
```
- 记住用户偏好、项目约定
- 记住哪些方案有效/失败
- 记住外部系统的行为模式
### 2. 策略进化（Strategy Evolution）
根据历史成败调整决策路径。
```python
# 伪代码：策略选择 + 反馈循环
class EvolvingAgent:
    def select_strategy(self, task):
        # 查历史：类似任务用什么策略成功过？
        past = self.memory.recall(similar_to=task)
        if past.success_rate("strategy_A") > 0.8:
            return "strategy_A"
        else:
            return self.explore_new_strategy(task)

    def learn_from_outcome(self, task, strategy, result):
        # 反馈：记录本次结果，更新策略偏好
        self.memory.store(task, strategy, result.success)
```
典型机制：
- **Reflexion**：执行后自我反思，生成改进建议存入记忆
- **Trial & Error**：多路径尝试，保留最优解
- **Reward Signal**：根据用户反馈/任务指标强化好策略
### 3. 能力进化（Capability Evolution）
Agent 动态扩展自身能力边界。
| 方式 | 说明 |
| --- | --- |
| 工具学习 | 遇到新 API 时自动学习用法，下次直接调用 |
| 代码生成 | 发现缺少某工具时，自己写一个并注册 |
| Prompt 自优化 | 根据效果自动调整 system prompt |
| 技能组合 | 将多步操作封装为可复用的"技能" |
---
## 能力进化深入展开
能力进化是自进化中最具"涌现性"的层次——Agent 不仅变得更聪明，还变得**能做更多事**。
### 3.1 工具学习（Tool Learning）
Agent 在运行时遇到未知工具/API，通过阅读文档或试探性调用来学会使用。
```plain text
发现需求 → 搜索可用工具 → 阅读文档/schema → 试调用 → 验证结果 → 存入技能库
```
```python
class ToolLearner:
    def encounter_new_tool(self, tool_schema):
        # 1. 理解参数和返回值
        understanding = self.llm.analyze(tool_schema)
        # 2. 构造测试用例试跑
        test_result = self.try_call(tool_schema, sample_params)
        # 3. 验证通过后注册
        if test_result.success:
            self.tool_registry.register(tool_schema, understanding)
```
**现实案例**：
- Agent 首次接触一个新数据库，读 schema 后自动学会查询方式
- 遇到新 REST API，解析 OpenAPI spec 后直接调用
- 发现 MCP server 提供的工具，按 schema 自动适配
### 3.2 代码生成与自注册（Self-Tooling）
当现有工具无法满足需求时，Agent 自己编写工具代码并注册为可用能力。
```plain text
┌─────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ 识别能力 │ ──→ │ 生成工具  │ ──→ │ 测试验证  │ ──→ │ 注册入库  │
│ 缺口    │     │ 代码     │     │          │     │          │
└─────────┘     └──────────┘     └──────────┘     └──────────┘
```
**Voyager（Minecraft Agent）的做法**：
```javascript
// Agent 自动生成的技能函数
async function mineWoodLog(bot) {
  const log = bot.findBlock({ matching: mcData.blocksByName.oak_log.id });
  if (!log) {
    await exploreUntil(bot, 60, () =>
      bot.findBlock({ matching: mcData.blocksByName.oak_log.id })
    );
  }
  await bot.dig(log);
}
// 技能存入向量数据库，未来遇到类似需求直接检索复用
```
**在软件工程 Agent 中的体现**：
```python
# Agent 发现需要批量重命名文件，但没有现成工具
# 于是自己写一个：
def batch_rename(directory, pattern, replacement):
    """批量重命名文件"""
    import os, re
    for f in os.listdir(directory):
        new_name = re.sub(pattern, replacement, f)
        if new_name != f:
            os.rename(os.path.join(directory, f),
                      os.path.join(directory, new_name))
    return f"Renamed files in {directory}"

# 注册为可调用工具
agent.register_tool("batch_rename", batch_rename)
```
### 3.3 Prompt 自优化（Self-Prompt Optimization）
Agent 根据任务效果自动调整自身的系统提示词。
```plain text
Round 1: 原始 prompt → 执行 → 效果评分 60%
Round 2: 修改 prompt（加入约束） → 执行 → 效果评分 75%
Round 3: 再次调整（加入示例） → 执行 → 效果评分 90% ✓ 保留
```
实现方式：
| 方法 | 机制 |
| --- | --- |
| APE (Automatic Prompt Engineering) | 生成多个候选 prompt，评估选最优 |
| OPRO | 用 LLM 本身作为优化器，迭代改进 prompt |
| DSPy | 将 prompt 视为可优化参数，用编译器自动调参 |
| PromptBreeder | 进化算法，prompt 之间交叉变异 |
```python
class PromptEvolver:
    def optimize(self, task_set, initial_prompt):
        best_prompt = initial_prompt
        best_score = self.evaluate(best_prompt, task_set)

        for _ in range(max_iterations):
            # LLM 生成变体
            variants = self.llm.generate_variants(best_prompt, n=5)
            for v in variants:
                score = self.evaluate(v, task_set)
                if score > best_score:
                    best_prompt, best_score = v, score

        return best_prompt
```
### 3.4 技能组合与抽象（Skill Composition）
将多个原子操作组合为高层技能，形成层次化能力体系。
```plain text
层次 3：部署应用 = 构建 + 测试 + 推送 + 发布
层次 2：构建 = 安装依赖 + 编译 + 打包
层次 1：安装依赖 = 读配置 + 执行 pip install
层次 0：原子工具（读文件、执行命令、网络请求）
```
```python
class SkillLibrary:
    def __init__(self):
        self.skills = {}  # name → Skill

    def compose(self, name, steps, preconditions=None):
        """将多个已有技能组合为新技能"""
        async def composed_skill(context):
            results = []
            for step in steps:
                r = await self.skills[step].execute(context)
                results.append(r)
                context.update(r)
            return results
        self.skills[name] = Skill(name, composed_skill, preconditions)

    def decompose_and_learn(self, complex_task, execution_trace):
        """从一次成功的复杂执行中拆解出可复用的子技能"""
        segments = self.identify_reusable_segments(execution_trace)
        for seg in segments:
            if seg.reuse_potential > threshold:
                self.register(seg.as_skill())
```
### 3.5 跨任务能力迁移（Skill Transfer）
在 A 任务中学到的能力，自动应用到 B 任务。
```plain text
任务 A（数据清洗）→ 学会了"正则提取 + 异常处理"模式
任务 B（日志分析）→ 检索到相似模式 → 直接复用，无需重新探索
```
关键在于**技能的泛化表示**：
- 不是存"清洗 CSV 第三列"这样的具体操作
- 而是存"对结构化文本的列做正则提取并处理异常值"这样的抽象模式
### 3.6 能力进化的安全边界
自进化必须有约束，否则 Agent 可能：
| 风险 | 应对 |
| --- | --- |
| 生成恶意工具 | 沙箱执行 + 权限白名单 |
| Prompt 注入自身 | 优化后的 prompt 需人工审核 |
| 无限自我扩展 | 技能数量上限 + 定期清理 |
| 能力退化 | 版本管理，可回滚到上一个有效版本 |
```python
class SafeEvolution:
    def register_new_skill(self, skill):
        # 1. 沙箱测试
        if not self.sandbox.test(skill):
            return False
        # 2. 权限检查
        if skill.requires_permissions not in self.allowed:
            return False
        # 3. 人工确认（高风险操作）
        if skill.risk_level > THRESHOLD:
            return self.request_human_approval(skill)
        # 4. 注册并记录版本
        self.library.register(skill, version=self.next_version())
        return True
```
---
## 经典架构
```plain text
┌─────────────────────────────────────────┐
│              Self-Evolving Agent          │
├─────────────────────────────────────────┤
│                                          │
│  ┌─────────┐  ┌──────────┐  ┌────────┐ │
│  │ Planner │→ │ Executor │→ │ Critic │ │
│  │ (规划)   │  │ (执行)    │  │ (评估) │ │
│  └─────────┘  └──────────┘  └────────┘ │
│       ↑                          │       │
│       └──── Experience Pool ←────┘       │
│             (经验池)                      │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │        Evolution Engine           │   │
│  │  - 经验提炼 (Distillation)        │   │
│  │  - 策略更新 (Policy Update)       │   │
│  │  - 能力扩展 (Skill Acquisition)   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```
## 关键论文/项目
| 名称 | 核心思想 |
| --- | --- |
| **Reflexion** | 执行 → 失败 → 生成语言反馈 → 存入记忆 → 重试 |
| **Voyager** (Minecraft Agent) | 自动编写技能代码 + 技能库积累 → 能力不断扩展 |
| **LATS** | 将 MCTS 搜索引入 Agent 决策，试错中优化路径 |
| **ExpeL** | 从成功/失败经验中提取通用规则，跨任务迁移 |
| **Self-Refine** | 生成 → 自评 → 修改，迭代直到满意 |
| **DEPS** | 描述-解释-规划-选择，任务失败后自动诊断并调整计划 |
| **ToolLLM** | 大规模工具学习，Agent 自主检索并学会使用 16000+ API |
## 实现要点
```plain text
1. 经验表示：如何结构化存储经验（向量/KV/规则）
2. 检索机制：如何在新任务中找到相关经验
3. 反思质量：Agent 能否准确归因成败原因
4. 遗忘策略：过时/错误经验需要淘汰
5. 安全护栏：自进化不能突破安全边界
6. 能力版本化：技能需要版本管理和回滚机制
7. 组合爆炸控制：技能组合数量需要剪枝
```
## 与"微调"的区别
|  | 自进化 | 微调 |
| --- | --- | --- |
| 时机 | 运行时持续 | 离线批量 |
| 改变的是 | 记忆/策略/工具 | 模型权重 |
| 可解释性 | 高（经验可读） | 低 |
| 回滚 | 删除记忆即可 | 需要重新训练 |
| 成本 | 低（推理即可） | 高（GPU 训练） |
| 能力上限 | 受限于基座模型能力 | 可突破原有能力边界 |
## 一句话总结
> Agent 自进化 = **在不改模型权重的前提下，通过记忆、反思、技能积累，让 Agent 的表现随使用而持续提升。**