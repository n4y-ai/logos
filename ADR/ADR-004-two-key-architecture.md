# ADR-004: Двухключевая архитектура Логоса (Owner + Agent)

## Статус
✅ **Принято и реализовано** (2024-12-17)

## Контекст

Для корректной работы Логоса как автономного агента необходимо:

1. **Пользователь** должен владеть и контролировать своего Логоса
2. **Логос (AI)** должен иметь возможность идентифицировать себя и подписывать сообщения
3. Разделение ответственности: пользователь контролирует средства, Логос выполняет задачи

**Проблема:** Если у Логоса нет своего ключа — он не может:
- Подписывать сообщения от своего имени
- Идентифицироваться перед другими Логосами
- Выполнять автономные действия

**Источники истины:**
- `n4y.ai/logos/.ai-knowledge.md` — раздел **3**, **4**
- `n4y.ai/logos/MVP_IDENTITY_REGISTRATION_SPEC_v1.md` — раздел **7**, **9**

## Решение

### Двухключевая архитектура

```
┌─────────────────────────────────────────────────────────┐
│                  ИДЕНТИЧНОСТЬ ЛОГОСА                    │
│                    did:logos:HANDLE                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   OWNER KEY                      AGENT KEY              │
│   ─────────                      ─────────              │
│   Где: браузер пользователя      Где: backend           │
│   Кто: пользователь              Кто: AI агент          │
│                                                         │
│   Права:                         Права:                 │
│   ✓ Полный контроль              ✓ Подписывать сообщения│
│   ✓ Вывод средств                ✓ Идентифицироваться   │
│   ✓ Отзыв agent key              ✗ НЕ вывод средств     │
│   ✓ Удаление Логоса              ✗ НЕ изменение owner   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Реализация

**Smart Contract:** `LogosAccount.sol`
```solidity
contract LogosAccount {
    address public owner;   // Ключ пользователя (полный контроль)
    address public agent;   // Ключ Логоса (ограниченные права)
    string public handle;   // Зарегистрированный handle
    
    // Owner может всё
    function execute(...) onlyOwner { ... }
    function withdraw(...) onlyOwner { ... }
    function setAgent(...) onlyOwner { ... }
    function revokeAgent() onlyOwner { ... }
    
    // Agent может только подписывать
    function signMessage(...) onlyOwnerOrAgent { ... }
    
    // Верификация подписей
    function isOwnerSignature(...) view returns (bool);
    function isAgentSignature(...) view returns (bool);
}
```

**Factory:** `LogosAccountFactory.sol`
```solidity
function createLogos(
    string handle,
    address ownerKey,   // Публичный ключ пользователя
    address agentKey    // Публичный ключ Логоса
) returns (address account);
```

### Хранение ключей

| Ключ | Где хранится | Безопасность |
|------|--------------|--------------|
| **Owner** | localStorage браузера | Зашифрован паролем (AES-GCM, PBKDF2) |
| **Agent** | Backend secure storage | HSM/KMS в продакшене, файл в MVP |

### DID Document

```json
{
  "id": "did:logos:HANDLE",
  "verificationMethod": [
    {
      "id": "did:logos:HANDLE#owner",
      "type": "EcdsaSecp256k1VerificationKey2019",
      "blockchainAccountId": "eip155:8453:0x..."
    },
    {
      "id": "did:logos:HANDLE#agent",
      "type": "EcdsaSecp256k1VerificationKey2019",
      "blockchainAccountId": "eip155:8453:0x..."
    }
  ],
  "authentication": ["#owner", "#agent"],
  "capabilityDelegation": ["#agent"]
}
```

## Почему это соответствует спекам

- Пользователь владеет своим Логосом через owner key
- Логос может действовать автономно через agent key
- Разделение прав предотвращает злоупотребления
- Пользователь может в любой момент отозвать agent key

## Последствия

- Backend должен безопасно хранить agent keys
- Frontend должен генерировать и шифровать owner key
- При потере пароля — потеря доступа (можно добавить recovery позже)

## Acceptance Criteria

- [x] `LogosAccount.sol` реализует двухключевую архитектуру
- [x] `LogosAccountFactory.sol` создаёт аккаунты с обоими ключами
- [x] Тесты подтверждают разграничение прав (15/16 passing)
- [x] Backend генерирует и хранит agent keys
- [x] Frontend генерирует owner key и шифрует паролем
- [ ] E2E тест полного flow создания Логоса

## Связанные файлы

| Файл | Описание |
|------|----------|
| `demo/contracts/core/LogosAccount.sol` | Smart Account контракт |
| `demo/contracts/core/LogosAccountFactory.sol` | Factory контракт |
| `demo/test/LogosAccount.test.cjs` | Тесты контрактов |
| `demo/backend-service/logos-api.js` | Backend API |
| `n4y.ai/lib/logos-wallet.ts` | Frontend crypto |
| `n4y.ai/app/create/page.tsx` | UI создания |


