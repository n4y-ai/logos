# Logos MVP — Deployment Registry

## Статус: ✅ LIVE на Base Mainnet

**Дата первого деплоя:** 2024-12-17  
**Сеть:** Base Mainnet (chainId: 8453)  
**Deployer:** `0xB12C128b8C0dE438AD0D4b8987FD074eF8be5B3A`

---

## Контракты

### NameRegistry V2 (Production)
| Параметр | Значение |
|----------|----------|
| **Адрес** | `0x87B9fD42553067BeBc2Abf42c7045C8F2F7D1A79` |
| **Сеть** | Base Mainnet (chainId: 8453) |
| **Статус** | ✅ LIVE |
| **Дата деплоя** | 2024-12-17 |

**Функции:**
- `registerName(string handle, address controller)` — регистрация для factory
- `isAvailable(string handle)` → `bool` — проверка доступности
- `resolve(string handle)` → `address` — получить controller по handle
- `reverseResolve(address controller)` → `string` — получить handle по адресу

**Исходный код:** https://github.com/n4y-ai/logos-protocol/blob/main/contracts/core/NameRegistry.sol

---

## LogosAccountFactory (Production)

| Параметр | Значение |
|----------|----------|
| **Адрес** | `0x7553244FB04ff64EA0B86cA16F0427feb7F4E263` |
| **Сеть** | Base Mainnet (chainId: 8453) |
| **Статус** | ✅ LIVE |
| **Дата деплоя** | 2024-12-17 |

---

## Зарегистрированные Handles (V2)

| Handle | Controller (LogosAccount) | Owner | Agent | Дата |
|--------|---------------------------|-------|-------|------|
| `4DTOCH` | `0xc14dFaf414b560bA36e5787ae8e2d1443b60FE3e` | `0xbE0cAE6396d4E1e323825b5bE1dD5003e4c47322` | `0x77dd967Df661D6fEdBb180Cb61BC45e6eCEeAD39` | 2024-12-17 |
| `8LXJXM` | TBD | `0x5E50a13A2BE8B7b5532c5c049fc587b5B71aa490` | `0x3D64d5Af04309FBf160fdBA4B0D782082576AdA3` | 2024-12-17 |
| `GUVUIK` | TBD | `0x92dA8259CD3cf4b33e0322dEd61d3a1E38e77A8f` | `0x943f2482283CD53d795b956fc1CCC673Ff26EbB8` | 2024-12-17 |

**Всего Logos:** 3

---

## Legacy Contracts (Deprecated)

### NameRegistry V1 (deprecated)
| Параметр | Значение |
|----------|----------|
| **Адрес** | `0x96A6802D7721016bB9c1181aaf95900335734115` |
| **Статус** | ⚠️ DEPRECATED |

### LogosAccountFactory V1 (deprecated)
| Параметр | Значение |
|----------|----------|
| **Адрес** | `0xc70230FfE1bB39c8aeDE63aedf2026372aa7d514` |
| **Статус** | ⚠️ DEPRECATED |

### Legacy Handles (V1 — MARAT only)
| Handle | Owner | TX | Дата |
|--------|-------|----|----|
| `MARAT` | `0xB12C128b8C0dE438AD0D4b8987FD074eF8be5B3A` | [0x181bdf8c...](https://basescan.org/tx/0x181bdf8cab7bbcfdc1a0c71fa5942ad78807cb6b0a68ea6babed71b6567d2398) | 2024-12-17 |

**Функции:**
- `createLogos(owner, agent, handle)` — создать Smart Account + зарегистрировать handle
- `ownerToAccount(address)` → `address` — получить аккаунт по owner
- `totalAccounts()` → `uint256` — количество созданных аккаунтов

**Исходный код:** https://github.com/n4y-ai/logos-protocol/blob/main/contracts/core/LogosAccountFactory.sol

---

## Как зарегистрировать handle

### Вариант 1: Через скрипт (требуется ETH на Base)

```bash
cd logos-protocol
HANDLE=YOURNAME npx hardhat run scripts/register-handle.js --network base
```

### Вариант 2: Через Backend API (рекомендуется)

```bash
curl -X POST http://localhost:3001/api/logos/create \
  -H "Content-Type: application/json" \
  -d '{"handle": "YOURNAME", "ownerPublicKey": "0x..."}'
```

### Вариант 3: Через ethers.js напрямую

```javascript
const { ethers } = require('ethers');

const FACTORY = '0x7553244FB04ff64EA0B86cA16F0427feb7F4E263';
const ABI = ['function createLogos(address owner, address agent, string handle) returns (address)'];

const provider = new ethers.JsonRpcProvider('https://mainnet.base.org');
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const contract = new ethers.Contract(FACTORY, ABI, wallet);

await contract.createLogos(ownerAddress, agentAddress, 'YOURHANDLE');
```

---

## Правила Handle (NEXO)

- **Длина:** 4–10 символов
- **Алфавит:** A–Z, 0–9 (Base36)
- **Регистр:** Нормализуется к uppercase (MARAT = marat = MaRaT)
- **Уникальность:** Глобальная, on-chain
- **Передача:** Запрещена (non-transferable)

---

---

## Frontend

**Репозиторий:** https://github.com/n4y-ai/n4y.ai  
**Production:** https://n4y.ai

| Страница | Описание |
|----------|----------|
| `/` | Главная: авторизация + Logos интерфейс |
| `/create` | Создание Логоса + экспорт/импорт backup |

**Компоненты:**
- `lib/logos-wallet.ts` — Генерация, шифрование, backup/restore
- `lib/logos-api.ts` — API client

**Auth Flow:**
```
/ (главная)
├── Нет identity → "Войти с файлом" / "Создать"
├── Есть identity, заблокирован → ввод пароля
└── Авторизован → Logos интерфейс + badge
```

**Backup/Restore:**
- Экспорт: кнопка "Скачать ключи" → файл `.logos`
- Импорт: "Восстановить из файла" или "Войти с файлом"

**Запуск:**
```bash
cd n4y.ai
npm run dev
```

---

## E2E Flow

```
Пользователь → Frontend → Backend → Blockchain
     │             │          │          │
     │   1. Click  │          │          │
     │   "Create"  │          │          │
     │─────────────►          │          │
     │             │          │          │
     │   2. Generate          │          │
     │   Owner Key │          │          │
     │◄────────────│          │          │
     │             │          │          │
     │   3. Enter  │          │          │
     │   Password  │          │          │
     │─────────────►          │          │
     │             │          │          │
     │   4. Select │  5. POST │          │
     │   Handle    │  /create │          │
     │─────────────►─────────►│          │
     │             │          │          │
     │             │  6. Generate        │
     │             │  Agent Key│         │
     │             │          │          │
     │             │  7. Register        │
     │             │  Handle   │─────────►
     │             │          │          │
     │   8. Save   │  Response │          │
     │   to localStorage      │          │
     │◄───────────────────────│          │
     │             │          │          │
     │   DID: did:logos:HANDLE│          │
```

---

## Связанные документы

- [ADR-001: Controller](./ADR/ADR-001-controller.md)
- [ADR-002: Name Registry](./ADR/ADR-002-name-registry.md)
- [ADR-003: Paymaster + Treasury](./ADR/ADR-003-paymaster-treasury.md)
- [ADR-004: Two-Key Architecture](./ADR/ADR-004-two-key-architecture.md)
- [MVP Spec v1](./MVP_IDENTITY_REGISTRATION_SPEC_v1.md)

---

## Токен QI Energy (справочно)

| Параметр | Значение |
|----------|----------|
| **Адрес** | `0x29897314A089fA75AC48eb66408c91751c26c588` |
| **Symbol** | QI |
| **Decimals** | 18 |
| **Сеть** | Base Mainnet |

---

## История изменений

| Дата | Событие |
|------|---------|
| 2024-12-17 | NameRegistry V1 задеплоен на Base Mainnet |
| 2024-12-17 | Первый handle `MARAT` зарегистрирован (V1) |
| 2024-12-17 | **NameRegistry V2 + LogosAccountFactory V2 задеплоены** |
| 2024-12-17 | **E2E flow — первый Logos `4DTOCH` создан on-chain** |
| 2024-12-17 | Исправлена генерация ключей (ethers.js secp256k1) |
| 2024-12-17 | **Auth flow реализован (challenge → sign → session)** |
| 2024-12-17 | **Backup/Restore: экспорт/импорт `.logos` файлов** |
| 2024-12-17 | **UI авторизации на главной странице** |
| 2024-12-17 | Logos `GUVUIK` создан с полным auth flow |

