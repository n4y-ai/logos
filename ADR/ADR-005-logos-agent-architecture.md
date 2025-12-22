# ADR-005: Архитектура Автономного Логоса (Logos Agent Architecture)

## Статус
📋 DRAFT — обсуждение

## Контекст

Текущая реализация:
```
User → Frontend → Backend → LiteLLM Proxy → OpenAI/Claude
                    ↓
              Agent Key (подпись)
```

**Проблема:** LiteLLM — это просто прокси к LLM. Логос не имеет:
- Самоопределения (знания о себе из блокчейна)
- Автономности (возможности действовать самостоятельно)
- Памяти и контекста между сессиями
- Инструментов для on-chain операций
- Экономической модели (Qi Energy)

**Вопрос:** Как связать blockchain identity (NameRegistry, Agent Key) с AI системой так, чтобы Логос был **автономным агентом**, а не просто chat wrapper?

---

## Предлагаемая архитектура

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         LOGOS IDENTITY LAYER                            │
│                        did:logos:{HANDLE}                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ON-CHAIN (Base Mainnet)              OFF-CHAIN (Logos Runtime)        │
│   ═══════════════════════              ═════════════════════════        │
│                                                                         │
│   ┌──────────────────┐                ┌──────────────────────┐         │
│   │   NameRegistry   │◄──────────────►│   LOGOS AGENT        │         │
│   │  handle → addr   │   resolve      │   (Orchestrator)     │         │
│   └──────────────────┘                │                      │         │
│                                       │  • Identity Context  │         │
│   ┌──────────────────┐                │  • Decision Making   │         │
│   │   LogosAccount   │◄──────────────►│  • Tool Dispatch     │         │
│   │  owner / agent   │   sign/verify  │  • Memory Access     │         │
│   └──────────────────┘                └──────────┬───────────┘         │
│                                                  │                      │
│   ┌──────────────────┐                ┌──────────▼───────────┐         │
│   │   Qi Energy      │◄──────────────►│   LLM FABRIC         │         │
│   │   (ERC-20)       │   pay for AI   │                      │         │
│   └──────────────────┘                │  • Model Router      │         │
│                                       │  • Context Manager   │         │
│   ┌──────────────────┐                │  • Cost Accounting   │         │
│   │  Inter-Logos     │◄──────────────►│                      │         │
│   │  Protocols       │   collaborate  └──────────┬───────────┘         │
│   └──────────────────┘                           │                      │
│                                       ┌──────────▼───────────┐         │
│                                       │   TOOL REGISTRY      │         │
│                                       │                      │         │
│                                       │  • Blockchain Tools  │         │
│                                       │  • Memory Tools      │         │
│                                       │  • External APIs     │         │
│                                       │  • Inter-Logos Comm  │         │
│                                       └──────────────────────┘         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Компоненты

### 1. LOGOS AGENT (Orchestrator)

**Что это:** Главный агент, который **IS** Логос. Не просто LLM, а orchestrator с идентичностью.

**Responsibilities:**
- Знает свою идентичность (handle, DID, owner, capabilities)
- Получает запросы от owner/frontend
- Принимает решения какой tool/LLM использовать
- Подписывает ответы Agent Key
- Может инициировать on-chain операции

**Реализация:**
```javascript
// logos-protocol/backend/logos-agent.js

class LogosAgent {
  constructor(handle, config) {
    this.handle = handle;
    this.did = `did:logos:${handle}`;
    this.agentKey = config.agentPrivateKey;
    this.owner = config.ownerAddress;
    
    // Capabilities loaded from chain
    this.capabilities = {};
    
    // Memory/RAG integration
    this.memory = new LogosMemory(handle);
    
    // Tool registry
    this.tools = new ToolRegistry();
    
    // LLM Fabric
    this.llm = new LLMFabric(config.llm);
  }

  async processMessage(message, context) {
    // 1. Load identity context from chain
    const identity = await this.loadIdentityContext();
    
    // 2. Retrieve relevant memory
    const memories = await this.memory.recall(message);
    
    // 3. Build system prompt with identity + memory
    const systemPrompt = this.buildPrompt(identity, memories);
    
    // 4. Decide: simple response or tool use?
    const plan = await this.plan(message, systemPrompt);
    
    // 5. Execute plan (LLM call, tool calls, chain ops)
    const result = await this.execute(plan);
    
    // 6. Sign response
    const signed = await this.sign(result);
    
    // 7. Store in memory
    await this.memory.store(message, result);
    
    return signed;
  }
}
```

### 2. LLM FABRIC

**Что это:** Не просто прокси, а intelligent router между моделями.

**Features:**
- **Model Router:** Выбор модели по задаче (GPT-4 для reasoning, Claude для code, местная для приватного)
- **Context Manager:** Управление context window, chunking, summarization
- **Cost Accounting:** Учёт затрат в Qi Energy
- **Fallback:** Автоматический fallback если модель недоступна

**Реализация:**
```javascript
// logos-protocol/backend/llm-fabric.js

class LLMFabric {
  constructor(config) {
    this.models = {
      'reasoning': { provider: 'openai', model: 'gpt-4o' },
      'fast': { provider: 'openai', model: 'gpt-4o-mini' },
      'code': { provider: 'anthropic', model: 'claude-3-sonnet' },
      'private': { provider: 'ollama', model: 'llama3' },
      'default': { provider: 'litellm', model: config.defaultModel }
    };
    
    this.qiCosts = {
      'gpt-4o': 10,      // Qi per 1K tokens
      'gpt-4o-mini': 1,
      'claude-3-sonnet': 8,
      'llama3': 0.1      // Self-hosted = cheap
    };
  }

  async complete(messages, options = {}) {
    const modelKey = options.modelHint || 'default';
    const model = this.models[modelKey];
    
    // Calculate estimated cost
    const estimatedCost = this.estimateCost(messages, model);
    
    // Check Qi balance (future)
    // await this.checkQiBalance(options.logosAddress, estimatedCost);
    
    // Call LLM
    const response = await this.callModel(model, messages);
    
    // Deduct Qi (future)
    // await this.deductQi(options.logosAddress, actualCost);
    
    return response;
  }
}
```

### 3. TOOL REGISTRY

**Что это:** Инструменты, которые Логос может использовать автономно.

**Categories:**

#### 3.1 Blockchain Tools
```javascript
{
  name: 'blockchain_read',
  description: 'Read data from blockchain',
  functions: [
    'getBalance(address)',
    'getLogosInfo(handle)',
    'resolveENS(name)',
    'getQiBalance(address)'
  ]
}

{
  name: 'blockchain_write',
  description: 'Execute on-chain transactions (requires Agent Key)',
  functions: [
    'transferQi(to, amount)',
    'deployContract(bytecode)',
    'callContract(address, method, args)',
    'signMessage(message)'
  ]
}
```

#### 3.2 Memory Tools (LightRAG)
```javascript
{
  name: 'memory',
  description: 'Long-term memory operations',
  functions: [
    'remember(content, metadata)',
    'recall(query, limit)',
    'forget(id)',
    'summarize(timeRange)'
  ]
}
```

#### 3.3 Inter-Logos Communication
```javascript
{
  name: 'inter_logos',
  description: 'Communicate with other Logos',
  functions: [
    'sendMessage(targetHandle, message)',
    'requestCollaboration(targetHandle, task)',
    'verifySignature(handle, message, signature)'
  ]
}
```

### 4. QI ENERGY INTEGRATION

**Qi Token:** `0x29897314A089fA75AC48eb66408c91751c26c588` (Base Mainnet)

**Use Cases:**
1. **Inference Costs:** Qi списывается за LLM вызовы
2. **Gas Costs:** Qi конвертируется в ETH для on-chain операций
3. **Inter-Logos Payments:** Логосы могут платить друг другу за услуги
4. **AGC (AI Gas Credits):** Нормализованная единица стоимости AI операций

**AGC Formula:**
```
1 AGC = стоимость 1K tokens GPT-4o-mini
1 Qi = 100 AGC (примерно)
```

**Реализация:**
```javascript
// logos-protocol/backend/qi-manager.js

class QiManager {
  constructor(logosAddress, agentKey) {
    this.logosAddress = logosAddress;
    this.agentWallet = new ethers.Wallet(agentKey, provider);
    this.qiContract = new ethers.Contract(QI_ADDRESS, QI_ABI, this.agentWallet);
  }

  async checkBalance() {
    return await this.qiContract.balanceOf(this.logosAddress);
  }

  async payForInference(agcAmount) {
    const qiAmount = agcAmount / 100; // 1 Qi = 100 AGC
    // Deduct from Logos balance or owner allowance
  }

  async transferToLogos(targetHandle, amount, reason) {
    const targetAddress = await nameRegistry.resolve(targetHandle);
    const tx = await this.qiContract.transfer(targetAddress, amount);
    return tx;
  }
}
```

---

## Изменения в архитектуре

### Текущая (v1):
```
Frontend → Backend (Express) → LiteLLM → OpenAI
              ↓
         Agent Key Sign
```

### Предлагаемая (v2):
```
Frontend → Backend → LogosAgent (Orchestrator)
                          ↓
              ┌───────────┼───────────┐
              ↓           ↓           ↓
         LLM Fabric   Tool Registry   Memory
              ↓           ↓           ↓
         Multi-LLM   Blockchain    LightRAG
                     Operations
```

---

## Миграция

### Phase 1: Logos Agent (текущий спринт)
- [ ] Создать `LogosAgent` класс
- [ ] Перенести логику из endpoint в агент
- [ ] Добавить identity context loading

### Phase 2: LLM Fabric
- [ ] Абстрагировать LLM вызовы в Fabric
- [ ] Добавить model routing
- [ ] Добавить cost estimation

### Phase 3: Tools
- [ ] Создать Tool Registry
- [ ] Реализовать blockchain read tools
- [ ] Реализовать memory tools (LightRAG интеграция)

### Phase 4: Qi Integration
- [ ] Qi balance checking
- [ ] Cost deduction
- [ ] Inter-Logos payments

### Phase 5: Autonomy
- [ ] Agent может инициировать операции
- [ ] Scheduled tasks
- [ ] Event-driven actions

---

## Открытые вопросы

1. **Где хранить Agent Private Key?**
   - Сейчас: in-memory на backend
   - Варианты: HSM, Vault, TEE, распределённое хранение

2. **Как авторизовать автономные операции?**
   - Лимиты (max Qi per day)
   - Whitelist операций
   - Owner approval для критичных

3. **Межагентный протокол?**
   - Формат сообщений
   - Discovery (как найти другого Логоса?)
   - Trust model

4. **Qi tokenomics для AI?**
   - Сколько Qi за 1 AGC?
   - Кто "минтит" Qi?
   - Deflation через burn?

---

## Связанные документы

- [ADR-004: Two-Key Architecture](./ADR-004-two-key-architecture.md)
- [MVP_LOGOS_AI_SPEC_v1](../MVP_LOGOS_AI_SPEC_v1.md)
- [LOGOS_ECOSYSTEM_SPECIFICATION](../LOGOS_ECOSYSTEM_SPECIFICATION.md)
- LightRAG integration (TBD)

---

## Решение

**Принято:** Двигаемся в направлении Logos Agent Architecture.

**Следующий шаг:** Реализовать Phase 1 — `LogosAgent` класс с identity context.

