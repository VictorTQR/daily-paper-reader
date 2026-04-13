# SiliconFlow LLM Provider 实施计划

&gt; **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 完善项目对硅基流动 (SiliconFlow) LLM 提供商的支持，包括前端配置预设和后端验证。

**Architecture:** 在保持现有架构不变的前提下，向前端添加 SiliconFlow 预设配置，同时验证后端支持的完整性。后端已具备基本支持，只需完善前端集成。

**Tech Stack:** 原生 JavaScript (前端), Python 3.11 (后端), OpenAI 兼容 API

---

## 文件结构映射

### 现有文件分析

| 文件 | 路径 | 责任 | 需要修改 |
|------|------|------|----------|
| 前端 LLM 配置工具 | `app/llm-config-utils.js` | LLM 提供商预设、类型检测、API 配置 | ✅ 需要修改 |
| 后端 LLM 客户端 | `src/llm.py` | LLM 提供商客户端实现 | ⚠️ 需验证 |
| 测试文件 | `tests/test_llm_base_url.py` | 基础 URL 测试 | ❌ 无需修改 |

### 修改文件清单

1. **修改**: `app/llm-config-utils.js` - 添加 SiliconFlow 预设配置
2. **验证**: `src/llm.py` - 确认后端支持完整性

---

## Task 1: 添加 SiliconFlow 前端预设配置

**Files:**
- Modify: `app/llm-config-utils.js:17-48`

**目标:** 在 `OPENAI_COMPATIBLE_PRESETS` 对象中添加 SiliconFlow 预设

- [ ] **Step 1: 在 OPENAI_COMPATIBLE_PRESETS 中添加 siliconflow 配置**

在 `app/llm-config-utils.js` 的 `OPENAI_COMPATIBLE_PRESETS` 对象（第 17 行开始）中，在 `openai` 配置之后添加以下内容：

```javascript
    siliconflow: Object.freeze({
      key: 'siliconflow',
      label: '硅基流动 SiliconFlow',
      baseUrl: 'https://api.siliconflow.cn/v1',
      models: Object.freeze([
        'Qwen/Qwen3-8B',
        'Qwen/Qwen3-72B',
        'deepseek-ai/DeepSeek-V3',
        'meta-llama/Llama-3.1-8B-Instruct',
        'meta-llama/Llama-3.1-70B-Instruct',
      ]),
    }),
```

- [ ] **Step 2: 验证修改位置正确**

确认代码插入在 `openai` 配置之后，整个 `OPENAI_COMPATIBLE_PRESETS` 对象结束之前。最终结构应该是：

```javascript
const OPENAI_COMPATIBLE_PRESETS = Object.freeze({
  deepseek: { ... },
  glm: { ... },
  minimax: { ... },
  kimi: { ... },
  openai: { ... },
  siliconflow: { ... },  // 新增
});
```

---

## Task 2: 更新 inferProviderType 函数

**Files:**
- Modify: `app/llm-config-utils.js:137-150`

**目标:** 在提供商类型检测函数中添加 SiliconFlow URL 检测

- [ ] **Step 1: 修改 inferProviderType 函数**

找到 `inferProviderType` 函数（第 137 行），在 `resolveSummaryLLM` 检查之后添加 SiliconFlow URL 检测：

```javascript
  const inferProviderType = (secret) => {
    const safeSecret = secret && typeof secret === 'object' ? secret : {};
    const llmProvider = safeSecret.llmProvider || {};
    const explicit = normalizeText(llmProvider.type || llmProvider.provider || '').toLowerCase();
    if (explicit === 'plato' || explicit === 'openai-compatible') {
      return explicit;
    }
    const summary = resolveSummaryLLM(safeSecret);
    if (!summary) return 'plato';
    if (/bltcy\.ai|gptbest\.vip/i.test(summary.baseUrl)) {
      return 'plato';
    }
    if (/siliconflow\.cn/i.test(summary.baseUrl)) {
      return 'openai-compatible';
    }
    return 'openai-compatible';
  };
```

---

## Task 3: 更新 inferChatApiProfile 函数

**Files:**
- Modify: `app/llm-config-utils.js:164-177`

**目标:** 在 API profile 检测函数中添加 SiliconFlow 识别

- [ ] **Step 1: 修改 inferChatApiProfile 函数**

找到 `inferChatApiProfile` 函数（第 164 行），在现有检测之后添加 SiliconFlow 检测：

```javascript
  const inferChatApiProfile = (baseUrl, model) => {
    const normalizedBaseUrl = normalizeBaseUrlForStorage(baseUrl || '').toLowerCase();
    const normalizedModel = normalizeText(model || '').toLowerCase();
    if (
      /(^|\/\/)(api\.)?deepseek\.com(?:$|\/)/i.test(normalizedBaseUrl)
      || normalizedModel.startsWith('deepseek-')
    ) {
      return 'deepseek';
    }
    if (/bltcy\.ai|gptbest\.vip/i.test(normalizedBaseUrl)) {
      return 'plato';
    }
    if (/siliconflow\.cn/i.test(normalizedBaseUrl)) {
      return 'generic-openai';
    }
    return 'generic-openai';
  };
```

---

## Task 4: 更新 buildConnectivityTestPayload 函数

**Files:**
- Modify: `app/llm-config-utils.js:207-240`

**目标:** 在连接测试 payload 构建函数中添加 SiliconFlow 模型检测

- [ ] **Step 1: 修改 wantsMaxCompletionTokens 检测条件**

找到 `buildConnectivityTestPayload` 函数中的 `wantsMaxCompletionTokens` 定义（第 210 行），添加 SiliconFlow 相关模型检测：

```javascript
    const wantsMaxCompletionTokens =
      /^glm-/i.test(normalizedModel)
      || /open\.bigmodel\.cn/.test(normalizedBaseUrl)
      || /thinking/i.test(normalizedModel)
      || /^kimi-/i.test(normalizedModel)
      || /^minimax-/i.test(normalizedModel)
      || normalizedModel.toLowerCase() === 'deepseek-reasoner'
      || /siliconflow\.cn/i.test(normalizedBaseUrl);
```

---

## Task 5: 验证后端支持

**Files:**
- Verify: `src/llm.py`

**目标:** 确认后端对 SiliconFlow 的支持已完整

- [ ] **Step 1: 检查后端 SiliconFlowClient 类**

查看 `src/llm.py`，确认以下内容存在：

1. 第 13 行注释包含 "siliconflow"
2. 第 139-140 行 `_provider_name` 方法中有 siliconflow 检测
3. 第 583-585 行有 `SiliconflowClient` 类定义
4. 第 715 行示例包含 "SiliconFlow/Qwen/Qwen3-8B"
5. 第 747-748 行 `ClientFactory.from_env()` 中有 siliconflow 处理

如果以上内容都存在，则无需修改后端代码。

- [ ] **Step 2: 运行现有测试**

```bash
python -m pytest tests/test_llm_base_url.py -v
```

预期结果：所有测试通过

---

## Task 6: 集成测试

**Files:**
- Test: 手动测试流程

**目标:** 验证前后端集成正常工作

- [ ] **Step 1: 验证前端预设可访问**

检查以下代码能否正常获取 SiliconFlow 预设：

```javascript
// 在浏览器控制台或 Node.js 中测试
const preset = DPRLLMConfigUtils.getOpenAICompatiblePreset('siliconflow');
console.log('SiliconFlow preset:', preset);
```

预期输出：
```javascript
{
  key: 'siliconflow',
  label: '硅基流动 SiliconFlow',
  baseUrl: 'https://api.siliconflow.cn/v1',
  models: [
    'Qwen/Qwen3-8B',
    'Qwen/Qwen3-72B',
    'deepseek-ai/DeepSeek-V3',
    'meta-llama/Llama-3.1-8B-Instruct',
    'meta-llama/Llama-3.1-70B-Instruct',
  ]
}
```

- [ ] **Step 2: 验证提供商类型检测**

```javascript
const testSecret = {
  summarizedLLM: {
    baseUrl: 'https://api.siliconflow.cn/v1',
    apiKey: 'test-key',
    model: 'Qwen/Qwen3-8B'
  }
};
const providerType = DPRLLMConfigUtils.inferProviderType(testSecret);
console.log('Provider type:', providerType);
```

预期输出：`'openai-compatible'`

- [ ] **Step 3: 验证 API profile 检测**

```javascript
const profile = DPRLLMConfigUtils.inferChatApiProfile(
  'https://api.siliconflow.cn/v1',
  'Qwen/Qwen3-8B'
);
console.log('API profile:', profile);
```

预期输出：`'generic-openai'`

---

## Task 7: 提交更改

**Files:**
- Git: 版本控制

**目标:** 将更改提交到 Git

- [ ] **Step 1: 查看修改的文件**

```bash
git status
```

预期显示 `app/llm-config-utils.js` 被修改

- [ ] **Step 2: 查看具体修改内容**

```bash
git diff app/llm-config-utils.js
```

确认修改正确无误

- [ ] **Step 3: 提交更改**

```bash
git add app/llm-config-utils.js
git commit -m "feat: 添加 SiliconFlow (硅基流动) LLM 提供商支持

- 在 OPENAI_COMPATIBLE_PRESETS 中添加 siliconflow 预设
- 更新 inferProviderType 支持 siliconflow.cn 域名检测
- 更新 inferChatApiProfile 添加 siliconflow 识别
- 更新 buildConnectivityTestPayload 支持 siliconflow 模型"
```

---

## 自审清单

### 1. 规格覆盖检查
- ✅ 添加 SiliconFlow 前端预设 - Task 1 覆盖
- ✅ 更新提供商类型检测 - Task 2 覆盖
- ✅ 更新 API profile 检测 - Task 3 覆盖
- ✅ 更新连接测试 payload - Task 4 覆盖
- ✅ 验证后端支持 - Task 5 覆盖
- ✅ 集成测试 - Task 6 覆盖
- ✅ Git 提交 - Task 7 覆盖

### 2. 占位符检查
- ❌ 无 "TBD" 或 "TODO"
- ❌ 无模糊的 "添加适当错误处理"
- ❌ 无 "类似上述" 的引用
- ✅ 所有步骤都有完整代码

### 3. 类型一致性检查
- ✅ 所有函数名、属性名一致
- ✅ `siliconflow` 拼写一致（小写 key，首字母大写 label）
- ✅ API 模式使用 `'generic-openai'` 与现有模式一致

---

## 执行交接

计划已保存到 `docs/superpowers/plans/2026-04-13-siliconflow-llm-provider.md`。两种执行选项：

**1. Subagent-Driven (推荐)** - 我为每个任务分配一个独立的子代理，在任务之间进行审查，实现快速迭代

**2. Inline Execution** - 在当前会话中使用 executing-plans 执行任务，带检查点的批量执行以供审查

选择哪种方式？
