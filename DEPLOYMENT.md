# Logos MVP — Deployment Registry

## Статус: ✅ LIVE на Base Mainnet

**Дата первого деплоя:** 2024-12-17  
**Сеть:** Base Mainnet (chainId: 8453)  
**Deployer:** `0xB12C128b8C0dE438AD0D4b8987FD074eF8be5B3A`

---

## Контракты

### NameRegistry (Handle Registry)
| Параметр | Значение |
|----------|----------|
| **Адрес** | `0x96A6802D7721016bB9c1181aaf95900335734115` |
| **TX деплоя** | [0xec5f35d0...](https://basescan.org/tx/0xec5f35d04f72347cda63bd219c3fc3b526a923490e8b539dfef2723489c6312d) |
| **Блок** | 39586942 |
| **Статус** | ✅ Работает |
| **Верификация** | ⏳ Pending |

**Функции:**
- `registerName(string _handle)` — регистрация handle (4-10 символов, Base36)
- `isHandleAvailable(string _handle)` — проверка доступности
- `resolveHandle(string _handle)` → `address` — получить владельца handle
- `registeredHandles(address)` → `string` — получить handle по адресу

**Исходный код:** `n4y.ai/demo/contracts/core/NameRegistry.sol`

---

## Зарегистрированные Handles

| Handle | Owner | TX | Дата |
|--------|-------|----|----|
| `MARAT` | `0xB12C128b8C0dE438AD0D4b8987FD074eF8be5B3A` | [0x181bdf8c...](https://basescan.org/tx/0x181bdf8cab7bbcfdc1a0c71fa5942ad78807cb6b0a68ea6babed71b6567d2398) | 2024-12-17 |

---

## Отложенные компоненты

| Контракт | Статус | Причина |
|----------|--------|---------|
| TreasuryMultisig2of3 | ⏸️ | Требуется 3 уникальных адреса для 2-of-3 |
| LogosRegisterPaymaster | ⏸️ | Зависит от Treasury |

---

## Как зарегистрировать handle

### Вариант 1: Через скрипт (требуется ETH на Base)

```bash
cd n4y.ai/demo
HANDLE=YOURNAME npx hardhat run scripts/register-handle.js --network base
```

### Вариант 2: Через ethers.js напрямую

```javascript
const { ethers } = require('ethers');

const NAME_REGISTRY = '0x96A6802D7721016bB9c1181aaf95900335734115';
const ABI = ['function registerName(string _handle) external'];

const provider = new ethers.JsonRpcProvider('https://mainnet.base.org');
const wallet = new ethers.Wallet(PRIVATE_KEY, provider);
const contract = new ethers.Contract(NAME_REGISTRY, ABI, wallet);

await contract.registerName('YOURHANDLE');
```

### Вариант 3: Через BaseScan (после верификации)

1. Открыть https://basescan.org/address/0x96A6802D7721016bB9c1181aaf95900335734115#writeContract
2. Connect Wallet
3. Вызвать `registerName` с желаемым handle

---

## Правила Handle (NEXO)

- **Длина:** 4–10 символов
- **Алфавит:** A–Z, 0–9 (Base36)
- **Регистр:** Нормализуется к uppercase (MARAT = marat = MaRaT)
- **Уникальность:** Глобальная, on-chain
- **Передача:** Запрещена (non-transferable)

---

## Связанные документы

- [ADR-001: Controller](./ADR/ADR-001-controller.md)
- [ADR-002: Name Registry](./ADR/ADR-002-name-registry.md)
- [ADR-003: Paymaster + Treasury](./ADR/ADR-003-paymaster-treasury.md)
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
| 2024-12-17 | NameRegistry задеплоен на Base Mainnet |
| 2024-12-17 | Первый handle `MARAT` зарегистрирован |

