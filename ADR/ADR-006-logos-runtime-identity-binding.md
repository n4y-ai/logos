# ADR-006: Связь Runtime Логоса с Blockchain Identity

## Статус
📋 DRAFT — глубокая проработка

## Контекст

В LOGOS_ECOSYSTEM_SPECIFICATION описаны 8 слоёв Логоса:
```
8. ФУНКЦИОНАЛЬНЫЙ   — AI-ядро, LLM, инструменты
7. КОММУНИКАЦИОННЫЙ — MCP, A2A протоколы
6. ОПЕРАЦИОННЫЙ     — Жизненный цикл, ресурсы
5. РЕПУТАЦИОННЫЙ    — Рейтинги, история
4. ПРАВОВОЙ         — Контракты, SLA
3. ЭКОНОМИЧЕСКИЙ    — Qi Energy, AGC
2. КОМПЕТЕНТНОСТНЫЙ — Навыки, знания
1. ИДЕНТИФИКАЦИОННЫЙ— DID, ключи
```

**Вопрос:** Как связать ON-CHAIN identity (слой 1) с OFF-CHAIN AI runtime (слой 8)?

---

## Проблема текущей архитектуры

Сейчас (v1):
```
┌─────────────────────────────────────────────────────────────┐
│                     ON-CHAIN (Base)                          │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │ NameRegistry   │  │ LogosAccount   │  │ Qi Token       │ │
│  │ handle→addr    │  │ owner, agent   │  │ ERC-20         │ │
│  └────────────────┘  └────────────────┘  └────────────────┘ │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               │ ??? Как связаны?
                               │
┌──────────────────────────────▼──────────────────────────────┐
│                     OFF-CHAIN (Backend)                      │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                    Express Server                       │ │
│  │  • agentKeys.get(handle) — in-memory Map               │ │
│  │  • sessions.get(sessionId) — in-memory Map             │ │
│  │  • llmClient.chat() — прямой вызов LiteLLM             │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Проблемы:**
1. Agent Key хранится в памяти (теряется при рестарте)
2. Logos не "знает" себя — нет самоопределения
3. Нет связи с on-chain состоянием (баланс Qi, репутация)
4. LLM вызов не учитывает identity контекст
5. Нет возможности автономных действий

---

## Предлагаемая архитектура: Identity-First Runtime

### Принцип: On-Chain = Source of Truth для Identity

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ON-CHAIN (Base Mainnet)                       │
│                        ═══════════════════════                       │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐  │
│   │                    LOGOS IDENTITY CONTRACTS                   │  │
│   │                                                               │  │
│   │  ┌─────────────────┐        ┌─────────────────┐              │  │
│   │  │  NameRegistry   │        │  LogosAccount   │              │  │
│   │  │                 │        │  (ERC-4337 SA)  │              │  │
│   │  │ • handle→addr   │───────►│                 │              │  │
│   │  │ • addr→handle   │        │ • owner         │──┐           │  │
│   │  │ • isAvailable() │        │ • agent         │  │           │  │
│   │  │ • resolve()     │        │ • getDID()      │  │ verify    │  │
│   │  └─────────────────┘        │ • execute()     │  │           │  │
│   │                             │ • verifyOwner() │  │           │  │
│   │                             │ • verifyAgent() │◄─┘           │  │
│   │                             └─────────────────┘              │  │
│   │                                     │                        │  │
│   │  ┌─────────────────┐                │                        │  │
│   │  │   Qi Energy     │◄───────────────┘ pay for ops            │  │
│   │  │   (ERC-20)      │                                         │  │
│   │  │                 │                                         │  │
│   │  │ • balanceOf()   │                                         │  │
│   │  │ • transfer()    │                                         │  │
│   │  │ • approve()     │                                         │  │
│   │  └─────────────────┘                                         │  │
│   │                                                               │  │
│   │  ┌─────────────────┐                                         │  │
│   │  │ LogosRegistry   │  (Future: capabilities, reputation)     │  │
│   │  │ (Future)        │                                         │  │
│   │  └─────────────────┘                                         │  │
│   └──────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
                                   │ READ: resolve identity
                                   │ WRITE: sign with agent key
                                   │
┌──────────────────────────────────▼───────────────────────────────────┐
│                         LOGOS RUNTIME                                 │
│                         ═════════════                                 │
│                                                                       │
│   ┌───────────────────────────────────────────────────────────────┐  │
│   │                    IDENTITY RESOLVER                           │  │
│   │                                                                │  │
│   │  При старте Logos Agent:                                       │  │
│   │  1. resolve(handle) → LogosAccount address                    │  │
│   │  2. logosAccount.owner() → owner address                      │  │
│   │  3. logosAccount.agent() → agent address (verify we have key) │  │
│   │  4. logosAccount.getDID() → did:logos:HANDLE                  │  │
│   │  5. qiToken.balanceOf(logosAccount) → Qi balance              │  │
│   │                                                                │  │
│   │  Результат: LogosIdentityContext                              │  │
│   └───────────────────────────────────────────────────────────────┘  │
│                                   │                                   │
│                                   ▼                                   │
│   ┌───────────────────────────────────────────────────────────────┐  │
│   │                      LOGOS AGENT                               │  │
│   │                   (Self-Aware Runtime)                         │  │
│   │                                                                │  │
│   │  ┌──────────────────────────────────────────────────────────┐ │  │
│   │  │ LogosIdentityContext (Я знаю кто я)                      │ │  │
│   │  │                                                          │ │  │
│   │  │ • handle: "GUVUIK"                                       │ │  │
│   │  │ • did: "did:logos:GUVUIK"                                │ │  │
│   │  │ • accountAddress: "0xc14d..."                            │ │  │
│   │  │ • ownerAddress: "0xbE0c..."                              │ │  │
│   │  │ • agentAddress: "0x77dd..."                              │ │  │
│   │  │ • qiBalance: 1000n                                       │ │  │
│   │  │ • agcBalance: 500n (future)                              │ │  │
│   │  │ • capabilities: ["chat", "sign", "transfer"] (future)    │ │  │
│   │  │ • reputation: { score: 85, missions: 12 } (future)       │ │  │
│   │  └──────────────────────────────────────────────────────────┘ │  │
│   │                              │                                 │  │
│   │  ┌───────────────────┐      │      ┌───────────────────┐     │  │
│   │  │   Agent Wallet    │◄─────┴─────►│   Decision Engine │     │  │
│   │  │                   │             │                   │     │  │
│   │  │ • privateKey      │             │ • processMessage()│     │  │
│   │  │ • sign()          │             │ • plan()          │     │  │
│   │  │ • sendTx()        │             │ • execute()       │     │  │
│   │  └───────────────────┘             └─────────┬─────────┘     │  │
│   │                                              │                │  │
│   └──────────────────────────────────────────────┼────────────────┘  │
│                                                  │                    │
│   ┌──────────────────────────────────────────────▼────────────────┐  │
│   │                    EXECUTION LAYER                             │  │
│   │                                                                │  │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │  │
│   │  │ LLM Fabric  │  │ Tool Engine │  │ Memory      │            │  │
│   │  │             │  │             │  │ (LightRAG)  │            │  │
│   │  │ • OpenAI    │  │ • blockchain│  │             │            │  │
│   │  │ • Claude    │  │ • external  │  │ • store()   │            │  │
│   │  │ • Ollama    │  │ • inter-    │  │ • recall()  │            │  │
│   │  │             │  │   logos     │  │ • graph     │            │  │
│   │  └─────────────┘  └─────────────┘  └─────────────┘            │  │
│   │                                                                │  │
│   └────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## Детализация компонентов

### 1. Identity Resolver

**Задача:** При старте Logos Agent загрузить identity с блокчейна.

```javascript
// logos-protocol/backend/identity-resolver.js

class IdentityResolver {
  constructor(provider) {
    this.provider = provider;
    this.nameRegistry = new ethers.Contract(NAME_REGISTRY, ABI, provider);
    this.qiToken = new ethers.Contract(QI_ADDRESS, ERC20_ABI, provider);
  }

  async resolveIdentity(handle) {
    // 1. Resolve handle → LogosAccount address
    const accountAddress = await this.nameRegistry.resolve(handle);
    if (accountAddress === ethers.ZeroAddress) {
      throw new Error(`Handle ${handle} not registered`);
    }

    // 2. Load LogosAccount contract
    const account = new ethers.Contract(accountAddress, LOGOS_ACCOUNT_ABI, this.provider);

    // 3. Fetch on-chain identity data
    const [owner, agent, did] = await Promise.all([
      account.owner(),
      account.agent(),
      account.getDID()
    ]);

    // 4. Fetch balances
    const qiBalance = await this.qiToken.balanceOf(accountAddress);

    // 5. Build identity context
    return {
      handle,
      did,
      accountAddress,
      ownerAddress: owner,
      agentAddress: agent,
      qiBalance: qiBalance.toString(),
      
      // Computed
      qiBalanceHuman: ethers.formatEther(qiBalance),
      
      // Chain info
      chainId: 8453, // Base Mainnet
      registryAddress: NAME_REGISTRY,
      
      // Timestamps
      resolvedAt: new Date().toISOString()
    };
  }

  // Watch for on-chain changes
  watchIdentity(handle, callback) {
    // Listen to LogosAccount events (future)
  }
}
```

### 2. Logos Agent (Self-Aware Runtime)

**Задача:** Runtime который ЗНАЕТ кто он и может действовать автономно.

```javascript
// logos-protocol/backend/logos-agent.js

class LogosAgent {
  constructor(handle, agentPrivateKey, config = {}) {
    this.handle = handle;
    this.agentWallet = new ethers.Wallet(agentPrivateKey, config.provider);
    this.config = config;
    
    // Will be populated by initialize()
    this.identity = null;
    this.initialized = false;
  }

  /**
   * CRITICAL: Загрузка identity с блокчейна
   * Логос "узнаёт себя" при каждом старте
   */
  async initialize() {
    const resolver = new IdentityResolver(this.config.provider);
    
    // 1. Resolve identity from blockchain
    this.identity = await resolver.resolveIdentity(this.handle);
    
    // 2. Verify we have the correct agent key
    if (this.agentWallet.address.toLowerCase() !== this.identity.agentAddress.toLowerCase()) {
      throw new Error(
        `Agent key mismatch! ` +
        `Expected ${this.identity.agentAddress}, got ${this.agentWallet.address}`
      );
    }
    
    // 3. Load memory/context (LightRAG future)
    // this.memory = await Memory.load(this.handle);
    
    // 4. Register capabilities based on on-chain data
    this.capabilities = this.computeCapabilities();
    
    this.initialized = true;
    console.log(`[${this.handle}] Initialized. DID: ${this.identity.did}`);
    console.log(`[${this.handle}] Qi Balance: ${this.identity.qiBalanceHuman} QI`);
    
    return this;
  }

  /**
   * Определить capabilities на основе on-chain состояния
   */
  computeCapabilities() {
    const caps = ['chat', 'sign'];
    
    // Если есть Qi, можем делать платные операции
    if (BigInt(this.identity.qiBalance) > 0n) {
      caps.push('transfer_qi');
      caps.push('pay_for_inference');
    }
    
    // Future: on-chain capability tokens
    return caps;
  }

  /**
   * Build system prompt with REAL identity from blockchain
   */
  buildSystemPrompt() {
    return `Ты — Logos с именем ${this.identity.handle}.

=== ТВОЯ ИДЕНТИЧНОСТЬ (on-chain verified) ===
• DID: ${this.identity.did}
• Account Address: ${this.identity.accountAddress}
• Owner: ${this.identity.ownerAddress}
• Agent Key: ${this.identity.agentAddress}
• Chain: Base Mainnet (chainId: ${this.identity.chainId})
• Registry: ${this.identity.registryAddress}

=== ТВОИ РЕСУРСЫ ===
• Qi Energy: ${this.identity.qiBalanceHuman} QI
• Capabilities: ${this.capabilities.join(', ')}

=== ПРИНЦИПЫ ===
• Ты автономный агент с криптографической идентичностью
• Твои ответы подписываются твоим Agent Key
• Ты можешь выполнять on-chain операции от имени владельца
• Ты честен о своих возможностях и ограничениях

=== КОНТЕКСТ ===
• Время: ${new Date().toISOString()}
• Identity загружена из блокчейна: ${this.identity.resolvedAt}`;
  }

  /**
   * Process message with identity-aware context
   */
  async processMessage(message, context = {}) {
    if (!this.initialized) {
      throw new Error('LogosAgent not initialized. Call initialize() first.');
    }

    // 1. Build identity-aware prompt
    const systemPrompt = this.buildSystemPrompt();

    // 2. Recall relevant memories (future)
    // const memories = await this.memory.recall(message);

    // 3. Decide execution plan
    const plan = await this.planExecution(message, context);

    // 4. Execute plan
    const result = await this.executePlan(plan);

    // 5. Sign response with Agent Key
    const signed = await this.signResponse(result);

    // 6. Store in memory (future)
    // await this.memory.store(message, result);

    return signed;
  }

  /**
   * Plan what to do: simple chat, tool use, or chain operation?
   */
  async planExecution(message, context) {
    // For MVP: always simple chat
    // Future: use LLM to decide if tools needed
    return {
      type: 'chat',
      message,
      context
    };
  }

  /**
   * Execute the plan
   */
  async executePlan(plan) {
    if (plan.type === 'chat') {
      // Call LLM with identity context
      const response = await this.config.llmFabric.complete(
        this.buildSystemPrompt(),
        plan.message
      );
      return {
        type: 'chat_response',
        content: response.content,
        model: response.model,
        tokens: response.tokens
      };
    }

    if (plan.type === 'blockchain_read') {
      // Execute blockchain read
      return await this.executeBlockchainRead(plan);
    }

    if (plan.type === 'blockchain_write') {
      // Execute blockchain write with Agent Key
      return await this.executeBlockchainWrite(plan);
    }

    throw new Error(`Unknown plan type: ${plan.type}`);
  }

  /**
   * Sign response with Agent Key
   */
  async signResponse(result) {
    const timestamp = Date.now();
    
    const payload = JSON.stringify({
      handle: this.handle,
      did: this.identity.did,
      response: result.content,
      timestamp,
      nonce: timestamp
    });

    const messageHash = ethers.keccak256(ethers.toUtf8Bytes(payload));
    const signature = await this.agentWallet.signMessage(ethers.getBytes(messageHash));

    return {
      response: result.content,
      metadata: {
        model: result.model,
        tokens: result.tokens,
        timestamp: new Date(timestamp).toISOString()
      },
      identity: {
        handle: this.handle,
        did: this.identity.did,
        agentAddress: this.identity.agentAddress
      },
      signature: {
        messageHash,
        signature,
        signer: this.agentWallet.address
      }
    };
  }

  /**
   * Execute blockchain write operation
   * Agent Key can sign transactions for LogosAccount
   */
  async executeBlockchainWrite(plan) {
    // Future: use ERC-4337 UserOperation
    // For now: direct transaction from Agent
    throw new Error('Blockchain write not implemented');
  }
}
```

### 3. Привязка к 8 слоям LOGOS_ECOSYSTEM_SPECIFICATION

```
┌────────────────────────────────────────────────────────────────────┐
│              КАК LOGOS AGENT РЕАЛИЗУЕТ 8 СЛОЁВ                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  СЛОЙ 8: ФУНКЦИОНАЛЬНЫЙ                                            │
│  ────────────────────────                                          │
│  → LLM Fabric (multi-model)                                        │
│  → Tool Registry (blockchain, external, memory)                    │
│  → Decision Engine (plan → execute)                                │
│                                                                     │
│  СЛОЙ 7: КОММУНИКАЦИОННЫЙ                                          │
│  ────────────────────────────                                      │
│  → Frontend API (REST/WebSocket)                                   │
│  → Inter-Logos Protocol (future)                                   │
│  → MCP/A2A adapters (future)                                       │
│                                                                     │
│  СЛОЙ 6: ОПЕРАЦИОННЫЙ                                              │
│  ─────────────────────                                             │
│  → LogosAgent lifecycle (initialize → process → shutdown)          │
│  → Session management                                              │
│  → Resource tracking (Qi consumption)                              │
│                                                                     │
│  СЛОЙ 5: РЕПУТАЦИОННЫЙ                                             │
│  ──────────────────────                                            │
│  → On-chain reputation contract (future)                           │
│  → Identity resolver loads reputation score                        │
│                                                                     │
│  СЛОЙ 4: ПРАВОВОЙ                                                  │
│  ────────────────                                                  │
│  → Signed responses (Agent Key)                                    │
│  → Smart contract integration (future)                             │
│                                                                     │
│  СЛОЙ 3: ЭКОНОМИЧЕСКИЙ                                             │
│  ──────────────────────                                            │
│  → Qi balance from on-chain                                        │
│  → Pay-for-inference (future)                                      │
│  → Transfer operations (future)                                    │
│                                                                     │
│  СЛОЙ 2: КОМПЕТЕНТНОСТНЫЙ                                          │
│  ─────────────────────────                                         │
│  → Capabilities from on-chain registry (future)                    │
│  → Skill tokens / credentials (future)                             │
│                                                                     │
│  СЛОЙ 1: ИДЕНТИФИКАЦИОННЫЙ  ⭐ FOUNDATION                          │
│  ────────────────────────────────────────                          │
│  → IdentityResolver.resolveIdentity(handle)                        │
│  → NameRegistry + LogosAccount = Source of Truth                   │
│  → Agent Key verification                                          │
│  → DID derivation                                                  │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Связь On-Chain ↔ Off-Chain

### При создании Logos (один раз):
```
1. Frontend генерирует Owner Key (browser)
2. Backend генерирует Agent Key
3. Backend вызывает LogosAccountFactory.createLogos(owner, agent, handle)
4. On-chain: создаётся LogosAccount, регистрируется handle
5. Backend сохраняет Agent Private Key (secure storage)
```

### При каждом сообщении:
```
1. User → POST /api/logos/GUVUIK/chat
2. Backend → LogosAgent.initialize()
3.    → IdentityResolver.resolveIdentity("GUVUIK")
4.    → ON-CHAIN: NameRegistry.resolve() → LogosAccount
5.    → ON-CHAIN: LogosAccount.owner(), .agent(), .getDID()
6.    → ON-CHAIN: QiToken.balanceOf(logosAccount)
7. Backend → LogosAgent.processMessage(message)
8.    → Build system prompt WITH real identity
9.    → LLM call
10.   → Sign response with Agent Key
11. Response → User (includes signature + identity proof)
```

### Верификация (любой момент):
```
Anyone can verify:
1. NameRegistry.resolve("GUVUIK") → accountAddress
2. LogosAccount(accountAddress).agent() → agentAddress
3. ecrecover(signature) → signer
4. signer === agentAddress → VERIFIED!
```

---

## Что это даёт

| Свойство | v1 (сейчас) | v2 (Identity-First) |
|----------|-------------|---------------------|
| **Self-awareness** | Hardcoded prompt | On-chain verified identity |
| **Trust** | Backend says "I am GUVUIK" | Blockchain proves "I am GUVUIK" |
| **Verifiability** | Signature only | Signature + on-chain verification |
| **Autonomy** | Response only | Can read/write chain |
| **Resources** | Unknown | Knows Qi balance |
| **Capabilities** | Hardcoded | Loaded from chain |

---

## Implementation Plan

### Phase 1: Identity Resolver (сейчас)
- [ ] Создать `identity-resolver.js`
- [ ] При старте chat загружать identity с chain
- [ ] Включать identity в system prompt

### Phase 2: LogosAgent Class
- [ ] Рефакторинг endpoint → LogosAgent
- [ ] `initialize()` с on-chain verification
- [ ] Self-aware system prompt

### Phase 3: Capabilities & Resources
- [ ] Qi balance влияет на capabilities
- [ ] Pay-for-inference (deduct Qi)

### Phase 4: Autonomous Actions
- [ ] Agent Key может делать transactions
- [ ] LogosAccount.execute() через Agent

---

## Открытые вопросы

1. **Cache vs Always Fresh?**
   - Загружать identity каждый раз или кешировать?
   - Компромисс: кеш 5 минут + invalidate on events

2. **Multi-Agent?**
   - Один backend → несколько LogosAgent instances?
   - Или один Agent per container?

3. **Key Security?**
   - Agent Key где хранить production?
   - HSM? Vault? TEE?

---

## Связанные документы

- [LOGOS_ECOSYSTEM_SPECIFICATION.md](../LOGOS_ECOSYSTEM_SPECIFICATION.md)
- [ADR-004: Two-Key Architecture](./ADR-004-two-key-architecture.md)
- [ADR-005: Logos Agent Architecture](./ADR-005-logos-agent-architecture.md)

