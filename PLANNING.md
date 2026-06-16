# 智能故障转移中转路由系统 - 完整实施计划

## 项目目标

将 Claude-Cloak 升级为**智能故障转移中转路由系统**，实现：
1. 多上游中转站管理（带权重）
2. 自动故障切换 + 内部重试
3. 错误内部消化，对用户透明
4. 心跳保持防止客户端超时
5. 多 API 格式兼容（OpenAI/Anthropic）
6. 思考强度控制与覆盖
7. Token 统计
8. 系统提示词 CLAUDE.md 格式处理

---

## 核心需求确认

1. ❌ **完全删除旧 credentials 系统** - 用新的 upstream-pool 替代
2. ✅ **API Key 绑定** - 允许为不同 API Key 指定不同的上游池
3. ✅ **敏感词混淆** - 按节点绑定敏感词集
4. ✅ **健康检查** - 仅被动检查（实际请求失败时更新，节省成本）
5. ✅ **权重** - 无上限限制，用户自由设置
6. ✅ **多接口兼容** - 支持 `/v1/chat/completions`, `/v1/messages`, `/v1/responses`
7. ✅ **思考强度** - 提取、覆盖、转换
8. ✅ **Token 统计** - 完整统计并返回
9. ✅ **系统提示词** - 严格模式关闭时以 CLAUDE.md 格式附加

---

## 系统架构

```
客户端 (多种 API 格式)
    ↓
[格式转换层] OpenAI ↔ Anthropic
    ↓
[认证层] API Key → 上游池映射
    ↓
[增强层] 思考强度、系统提示词、cch 生成
    ↓
[路由层] 加权选择节点
    ↓
[重试层] 故障切换 + 心跳 + 指数退避
    ↓
[伪装层] CC 请求伪装
    ↓
上游中转站池 (N 个节点)
    ↓
Anthropic API
```

---

## 从 Claude Code 提取的关键实现

### 1. Thinking (思考强度) 配置

```typescript
export type ThinkingConfig =
  | { type: 'adaptive' }  // 自适应思考
  | { type: 'enabled'; budgetTokens: number }  // 固定 token 预算
  | { type: 'disabled' }  // 禁用思考
```

### 2. CLAUDE.md 格式附加系统提示词

当严格模式关闭时，用户系统提示词以此格式附加：

```typescript
const MEMORY_INSTRUCTION_PROMPT = 
  'Codebase and user instructions are shown below. Be sure to adhere to these instructions. IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.'

function formatUserSystemPromptAsClaudeMd(userSystemPrompt: string): string {
  return `<user_instructions priority="high">
${MEMORY_INSTRUCTION_PROMPT}

${userSystemPrompt}
</user_instructions>`
}
```

### 3. Token 统计

```typescript
interface Usage {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens?: number
  cache_read_input_tokens?: number
}

function calculateTotalTokens(usage: Usage): number {
  return (
    usage.input_tokens +
    (usage.cache_creation_input_tokens ?? 0) +
    (usage.cache_read_input_tokens ?? 0) +
    usage.output_tokens
  )
}
```

### 4. cch 随机生成

```typescript
function generateRandomCch(): string {
  const chars = '0123456789abcdef'
  let result = ''
  for (let i = 0; i < 5; i++) {
    result += chars[Math.floor(Math.random() * chars.length)]
  }
  return result
}
```

---

## 数据模型设计

### 1. 上游节点模型

**文件**: `src/upstream-pool/types.ts`

```typescript
export interface UpstreamNode {
  id: string
  name: string
  targetUrl: string
  apiKey: string
  proxyUrl: string | null
  weight: number // 无上限，用户自由设置
  enabled: boolean
  
  // 健康状态（仅被动更新）
  healthy: boolean
  lastFailureTime: number | null
  lastSuccessTime: number | null
  lastLatency: number | null
  consecutiveFailures: number
  totalRequests: number
  totalFailures: number
  totalSuccesses: number
  
  // 敏感词绑定
  wordSetIds: string[]
  
  createdAt: string
  updatedAt: string
}

export interface UpstreamPool {
  id: string
  name: string
  nodeIds: string[] // 关联的节点 ID 列表
  createdAt: string
  updatedAt: string
}

export interface UpstreamPoolStore {
  nodes: UpstreamNode[]
  pools: UpstreamPool[]
}
```

### 2. API Key 绑定上游池

**文件**: `src/apikeys/types.ts` (修改)

```typescript
export interface ApiKey {
  id: string
  name: string
  key: string
  poolId: string | null // 绑定到上游池（null = 使用默认池）
  createdAt: string
  updatedAt: string
}
```

### 3. 思考强度配置

**文件**: `src/types.ts` (新增)

```typescript
export type ThinkingConfig =
  | { type: 'adaptive' }
  | { type: 'enabled'; budgetTokens: number }
  | { type: 'disabled' }

export interface ClaudeRequest {
  model: string
  max_tokens: number
  messages: ClaudeMessage[]
  system?: string | ClaudeSystemBlock[]
  thinking?: ThinkingConfig
  temperature?: number
  top_p?: number
  top_k?: number
  stream?: boolean
  metadata?: {
    user_id?: string
  }
}
```

### 4. 重试配置

**文件**: `src/services/retry-config.ts`

```typescript
export interface RetryConfig {
  maxRetries: number // 默认 3
  initialDelay: number // 初始延迟 ms，默认 1000
  maxDelay: number // 最大延迟 ms，默认 30000
  backoffMultiplier: number // 退避倍数，默认 2
  retryableStatusCodes: number[] // 可重试状态码
  heartbeatInterval: number // 心跳间隔 ms，默认 15000
}

export const DEFAULT_RETRY_CONFIG: RetryConfig = {
  maxRetries: 3,
  initialDelay: 1000,
  maxDelay: 30000,
  backoffMultiplier: 2,
  retryableStatusCodes: [408, 429, 500, 502, 503, 504],
  heartbeatInterval: 15000,
}
```

---

## 核心模块实现

### 模块 1: 上游池管理器

**文件**: `src/upstream-pool/manager.ts`

```typescript
export class UpstreamPoolManager {
  // 节点 CRUD
  async createNode(input: CreateNodeInput): Promise<UpstreamNode>
  async updateNode(id: string, input: UpdateNodeInput): Promise<UpstreamNode>
  async deleteNode(id: string): Promise<void>
  async setNodeEnabled(id: string, enabled: boolean): Promise<UpstreamNode>
  async setNodeWordSetIds(id: string, wordSetIds: string[]): Promise<UpstreamNode>
  
  // 池 CRUD
  async createPool(input: CreatePoolInput): Promise<UpstreamPool>
  async updatePool(id: string, input: UpdatePoolInput): Promise<UpstreamPool>
  async deletePool(id: string): Promise<void>
  async addNodeToPool(poolId: string, nodeId: string): Promise<void>
  async removeNodeFromPool(poolId: string, nodeId: string): Promise<void>
  
  // 节点选择（加权随机）
  selectHealthyNode(poolId: string): UpstreamNode | null
  
  // 健康管理（被动更新）
  recordSuccess(nodeId: string, latency: number): Promise<void>
  recordFailure(nodeId: string): Promise<void>
  
  // 查询
  getPoolForApiKey(apiKeyId: string): Promise<UpstreamPool>
  getNodesInPool(poolId: string): UpstreamNode[]
  getAllNodes(): UpstreamNode[]
  getAllPools(): UpstreamPool[]
}
```

### 模块 2: 加权选择算法

**文件**: `src/upstream-pool/selector.ts`

```typescript
export function selectByWeight(nodes: UpstreamNode[]): UpstreamNode | null {
  const healthyNodes = nodes.filter(n => n.enabled && n.healthy)
  if (healthyNodes.length === 0) return null
  
  const totalWeight = healthyNodes.reduce((sum, n) => sum + n.weight, 0)
  let random = Math.random() * totalWeight
  
  for (const node of healthyNodes) {
    random -= node.weight
    if (random <= 0) return node
  }
  
  return healthyNodes[0]
}
```

### 模块 3: API 格式转换

**文件**: `src/services/api-converter.ts`

```typescript
// OpenAI /v1/chat/completions → Anthropic /v1/messages
export function convertOpenAIToAnthropic(openaiReq: any): ClaudeRequest

// Anthropic → OpenAI 响应转换
export function convertAnthropicToOpenAI(anthropicResp: any): any

// /v1/responses → /v1/messages
export function convertResponsesToMessages(responsesReq: any): ClaudeRequest
```

### 模块 4: 思考强度处理

**文件**: `src/services/thinking-handler.ts`

```typescript
export interface ThinkingOverride {
  enabled: boolean
  type?: 'adaptive' | 'enabled' | 'disabled'
  budgetTokens?: number
}

export function applyThinkingOverride(
  request: ClaudeRequest,
  override: ThinkingOverride | null
): ClaudeRequest

export function extractThinkingConfig(
  request: ClaudeRequest
): ThinkingConfig | null
```

### 模块 5: 系统提示词处理

**文件**: `src/services/system-prompt-handler.ts`

```typescript
export function handleSystemPromptInNonStrictMode(
  request: ClaudeRequest,
  strictMode: boolean
): ClaudeRequest {
  if (strictMode) {
    // 覆盖用户系统提示词
    return {
      ...request,
      system: [CLAUDE_CODE_SYSTEM_PROMPT]
    }
  }
  
  // 将用户系统提示词以 CLAUDE.md 格式插入第一条 user 消息
  const userSystemPrompt = extractSystemPrompt(request.system)
  if (!userSystemPrompt) {
    return { ...request, system: [CLAUDE_CODE_SYSTEM_PROMPT] }
  }
  
  const claudeMdFormatted = formatAsClaudeMd(userSystemPrompt)
  return {
    ...request,
    system: [CLAUDE_CODE_SYSTEM_PROMPT],
    messages: [
      { role: 'user', content: claudeMdFormatted },
      ...request.messages
    ]
  }
}
```

### 模块 6: Token 统计

**文件**: `src/services/token-tracker.ts`

```typescript
export interface TokenUsage {
  input_tokens: number
  output_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
  total_tokens: number
}

export function extractTokenUsage(response: any): TokenUsage
export function attachTokenUsage(response: any, usage: TokenUsage): any
```

### 模块 7: 智能重试引擎

**文件**: `src/services/retry-engine.ts`

```typescript
export async function retryWithFailover<T>(
  executor: (node: UpstreamNode) => Promise<T>,
  options: {
    poolManager: UpstreamPoolManager
    pool: UpstreamPool
    config: RetryConfig
    reply?: FastifyReply
    isStream?: boolean
    logger?: FastifyBaseLogger
  }
): Promise<T> {
  // 1. 启动心跳
  // 2. 循环重试
  // 3. 选择健康节点
  // 4. 执行请求
  // 5. 记录成功/失败
  // 6. 指数退避
  // 7. 故障切换
}
```

### 模块 8: 心跳保持

**文件**: `src/services/heartbeat.ts`

```typescript
export function startHeartbeat(
  reply: FastifyReply,
  isStream: boolean,
  interval: number
): NodeJS.Timeout {
  return setInterval(() => {
    try {
      if (isStream) {
        reply.raw.write(': ping\n\n') // SSE 格式
      } else {
        reply.raw.write('') // 空 chunk
      }
    } catch (err) {
      // 客户端已断开
    }
  }, interval)
}
```

---

## 路由层实现

**文件**: `src/routes/proxy.ts` (完全重写)

```typescript
export async function proxyRoutes(fastify: FastifyInstance, config: Config) {
  // 1. Anthropic 原生格式
  fastify.post('/v1/messages', handleAnthropicMessages)
  
  // 2. OpenAI 格式
  fastify.post('/v1/chat/completions', handleOpenAIChatCompletions)
  
  // 3. Anthropic responses 格式
  fastify.post('/v1/responses', handleAnthropicResponses)
}

async function handleAnthropicMessages(req, reply) {
  const anthropicRequest = req.body as ClaudeRequest
  return await processRequest(anthropicRequest, 'anthropic', req, reply)
}

async function handleOpenAIChatCompletions(req, reply) {
  const openaiRequest = req.body
  const anthropicRequest = convertOpenAIToAnthropic(openaiRequest)
  const result = await processRequest(anthropicRequest, 'openai', req, reply)
  
  if (!anthropicRequest.stream) {
    return convertAnthropicToOpenAI(result)
  }
  return result
}

async function processRequest(anthropicRequest, format, req, reply) {
  // 1. 获取上游池
  const pool = await upstreamPoolManager.getPoolForApiKey(req.apiKeyEntity.id)
  
  // 2. 系统提示词处理
  const strictMode = settingsManager.isStrictMode()
  let enhanced = handleSystemPromptInNonStrictMode(anthropicRequest, strictMode)
  
  // 3. 思考强度覆盖
  const thinkingOverride = settingsManager.getThinkingOverride()
  enhanced = applyThinkingOverride(enhanced, thinkingOverride)
  
  // 4. 智能重试 + 故障转移
  const result = await retryWithFailover(
    async (node) => {
      const final = await enhanceAnthropicRequest(enhanced, req.log, node)
      return await executeProxyRequest(node, final, req, reply)
    },
    { poolManager, pool, config: DEFAULT_RETRY_CONFIG, reply, isStream: enhanced.stream, logger: req.log }
  )
  
  // 5. 附加 token 统计
  if (!enhanced.stream && result.usage) {
    const tokenUsage = extractTokenUsage(result)
    return attachTokenUsage(result, tokenUsage)
  }
  
  return result
}
```

---

## 管理面板更新

### 新增 API 端点

**文件**: `src/routes/admin.ts`

```typescript
// 上游节点
fastify.get('/admin/api/upstream-nodes', listNodes)
fastify.post('/admin/api/upstream-nodes', createNode)
fastify.put('/admin/api/upstream-nodes/:id', updateNode)
fastify.delete('/admin/api/upstream-nodes/:id', deleteNode)
fastify.post('/admin/api/upstream-nodes/:id/toggle', toggleNodeEnabled)
fastify.post('/admin/api/upstream-nodes/:id/test', testNodeHealth)
fastify.put('/admin/api/upstream-nodes/:id/word-sets', setNodeWordSets)

// 上游池
fastify.get('/admin/api/upstream-pools', listPools)
fastify.post('/admin/api/upstream-pools', createPool)
fastify.put('/admin/api/upstream-pools/:id', updatePool)
fastify.delete('/admin/api/upstream-pools/:id', deletePool)
fastify.post('/admin/api/upstream-pools/:id/nodes', addNodeToPool)
fastify.delete('/admin/api/upstream-pools/:id/nodes/:nodeId', removeNodeFromPool)

// 统计
fastify.get('/admin/api/upstream-stats', getUpstreamStats)

// 思考强度配置
fastify.get('/admin/api/thinking-override', getThinkingOverride)
fastify.put('/admin/api/thinking-override', updateThinkingOverride)
```

### UI 更新

**文件**: `public/app.js`

新增功能：
1. 上游节点管理页面
2. 上游池管理页面
3. 节点健康状态实时显示
4. 权重配置（无上限）
5. 思考强度全局覆盖配置
6. API Key 绑定池选择

---

## 完整模块清单

### Phase 1: 删除旧系统 + 数据层（Day 1）
1. ❌ **删除** `src/credentials/` 整个目录
2. ❌ **删除** `data/credentials.json`
3. ✅ 创建 `src/upstream-pool/types.ts`
4. ✅ 创建 `src/upstream-pool/storage.ts`
5. ✅ 创建 `src/upstream-pool/manager.ts`
6. ✅ 修改 `src/apikeys/types.ts` - 添加 poolId
7. ✅ 修改 `src/apikeys/manager.ts` - 支持池绑定

### Phase 2: 格式转换 + 增强服务（Day 2-3）
8. ✅ 创建 `src/services/api-converter.ts`
9. ✅ 创建 `src/services/thinking-handler.ts`
10. ✅ 创建 `src/services/system-prompt-handler.ts`
11. ✅ 创建 `src/services/token-tracker.ts`
12. ✅ 修改 `src/services/transform.ts` - 随机 cch
13. ✅ 创建 `src/upstream-pool/selector.ts`

### Phase 3: 重试引擎（Day 4）
14. ✅ 创建 `src/services/retry-config.ts`
15. ✅ 创建 `src/services/retry-engine.ts`
16. ✅ 创建 `src/services/heartbeat.ts`

### Phase 4: 路由集成（Day 5）
17. ✅ 重写 `src/routes/proxy.ts`
18. ✅ 修改 `src/server.ts`
19. ✅ 修改 `src/services/auth.ts` - 移除旧 credential 依赖

### Phase 5: 管理界面（Day 6-7）
20. ✅ 扩展 `src/routes/admin.ts`
21. ✅ 修改 `src/settings/manager.ts` - 思考强度配置
22. ✅ 重写 `public/app.js`
23. ✅ 修改 `public/index.html`

### Phase 6: 测试与文档（Day 8）
24. ✅ 端到端测试
25. ✅ 更新 `README.md`
26. ✅ 更新 `.env.example`

---

## 环境变量配置

**文件**: `.env.example` (更新)

```bash
# 管理员密钥
ADMIN_KEY=your-admin-secret-key

# 重试配置
RETRY_MAX_RETRIES=3
RETRY_INITIAL_DELAY=1000
RETRY_MAX_DELAY=30000
RETRY_BACKOFF_MULTIPLIER=2
RETRY_HEARTBEAT_INTERVAL=15000

# 思考强度全局覆盖（可选）
THINKING_OVERRIDE_ENABLED=false
THINKING_OVERRIDE_TYPE=adaptive  # adaptive | enabled | disabled
THINKING_OVERRIDE_BUDGET_TOKENS=10000

# 严格模式（是否覆盖用户系统提示词）
STRICT_MODE=false

# 存储路径
UPSTREAM_POOL_STORE_PATH=./data/upstream-pool.json
APIKEY_STORE_PATH=./data/apikeys.json
SENSITIVE_WORDS_PATH=./data/sensitive-words.json
SETTINGS_STORE_PATH=./data/settings.json

# CLI 版本
CLI_VERSION=2.1.167
SDK_VERSION=0.94.0
```

---

## 数据迁移脚本

**文件**: `src/upstream-pool/migration.ts`

```typescript
export async function migrateCredentialsToPool(): Promise<void> {
  const oldCredentialsPath = './data/credentials.json'
  
  // 检查旧文件是否存在
  if (!existsSync(oldCredentialsPath)) {
    console.log('No credentials.json found, skipping migration')
    return
  }
  
  const oldStore = JSON.parse(readFileSync(oldCredentialsPath, 'utf-8'))
  const newStore: UpstreamPoolStore = {
    nodes: [],
    pools: [{
      id: 'default',
      name: 'Default Pool',
      nodeIds: [],
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString()
    }]
  }
  
  for (const cred of oldStore.credentials) {
    const node: UpstreamNode = {
      id: cred.id,
      name: cred.name,
      targetUrl: cred.targetUrl,
      apiKey: cred.apiKey,
      proxyUrl: cred.proxyUrl || null,
      weight: 10,
      enabled: cred.enabled,
      healthy: true,
      lastFailureTime: null,
      lastSuccessTime: null,
      lastLatency: null,
      consecutiveFailures: 0,
      totalRequests: 0,
      totalFailures: 0,
      totalSuccesses: 0,
      wordSetIds: cred.wordSetIds || [],
      createdAt: cred.createdAt,
      updatedAt: cred.updatedAt
    }
    newStore.nodes.push(node)
    newStore.pools[0].nodeIds.push(node.id)
  }
  
  await writeUpstreamPoolStore(newStore)
  console.log(`Migrated ${newStore.nodes.length} credentials to upstream pool`)
  
  // 备份旧文件
  renameSync(oldCredentialsPath, oldCredentialsPath + '.backup')
}
```

---

## 测试计划

### 1. 单元测试
- [ ] 加权选择算法正确性
- [ ] API 格式转换准确性
- [ ] 思考强度提取/覆盖
- [ ] Token 统计计算
- [ ] 系统提示词 CLAUDE.md 格式化

### 2. 集成测试
- [ ] 故障切换流程
- [ ] 心跳发送验证
- [ ] 节点自动标记不健康/恢复
- [ ] 多格式 API 端到端

### 3. 压力测试
- [ ] 高并发性能
- [ ] 多节点轮询均匀性
- [ ] 内存泄漏检查

---

## 最终验收标准

✅ **功能完整性**
- [ ] 支持 3 种 API 格式互转（`/v1/messages`, `/v1/chat/completions`, `/v1/responses`）
- [ ] 思考强度可提取、可覆盖
- [ ] Token 统计完整准确
- [ ] 严格模式关闭时，系统提示词以 CLAUDE.md 格式附加
- [ ] cch 使用随机 5 位十六进制
- [ ] 旧 credentials 系统完全删除

✅ **故障转移**
- [ ] 单节点失败自动切换到下一个
- [ ] 重试期间心跳保持连接
- [ ] 错误内部消化，用户无感知（除非所有节点都失败）
- [ ] 被动健康检查工作正常

✅ **管理能力**
- [ ] 上游池 CRUD 完整
- [ ] 节点 CRUD 完整
- [ ] 节点权重无限制
- [ ] API Key 可绑定到不同池
- [ ] 实时健康状态显示
- [ ] 敏感词按节点绑定
- [ ] 思考强度全局覆盖配置

---

## 部署流程

```bash
# 1. 备份现有数据
cp data/credentials.json data/credentials.json.backup
cp data/apikeys.json data/apikeys.json.backup

# 2. 更新代码
cd /fandai_api
git pull

# 3. 安装依赖
bun install

# 4. 运行迁移脚本（可选，如果有旧数据）
bun run migrate

# 5. 配置环境变量
cp .env.example .env
vim .env

# 6. 构建并启动
docker compose up -d --build

# 7. 验证服务
curl http://localhost:4000/healthz

# 8. 访问管理面板配置节点
open http://localhost:4000/admin/
```

---

## 注意事项

1. **数据迁移**：首次部署时会自动将旧 `credentials.json` 转换为 `upstream-pool.json`
2. **向后兼容**：旧的 API Key 会自动绑定到 "Default Pool"
3. **健康检查**：仅被动检查，不产生额外 API 调用费用
4. **权重配置**：无上限，用户根据实际需求设置（建议 1-100）
5. **思考强度**：全局覆盖优先级高于请求中的配置

---

**计划完成，准备开始实施！**
