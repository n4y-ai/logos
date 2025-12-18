# Logos — открытый протокол идентичности и “namespace” для цифровых двойников

Этот репозиторий — **источник истины для спецификаций** экосистемы Logos в стиле **Spec Driven Development**: спеки, ADR, регламент проверок и публичные адреса деплоев.

**Ключевая идея:** любой разработчик может реализовать свой UI/Backend/AI-агента и создавать Logos **независимо от нас**, а **NameRegistry в Base** объединяет всех в одну экосистему через общий namespace имён (handle).

---

## TL;DR для разработчиков

- **Единый namespace**: `NameRegistry` on-chain в сети **Base** хранит соответствие `HANDLE → controller`.
- **DID**: `did:logos:HANDLE` (DID — это удобный слой представления над on-chain реестром).
- **Permissionless**: регистрация и чтение доступны всем, без “админов” и ручных политик.
- **Вы можете**:
  - резолвить handle в controller;
  - создавать свои Logos (через свою реализацию или нашу фабрику);
  - строить совместимые клиенты/протоколы общения.

---

## Точка согласия (Consensus Point): NameRegistry

**Сеть:** Base Mainnet (`chainId=8453`)  
**NameRegistry V2:** `0x87B9fD42553067BeBc2Abf42c7045C8F2F7D1A79`  
**LogosAccountFactory:** `0x7553244FB04ff64EA0B86cA16F0427feb7F4E263`

Полный реестр деплоев и список зарегистрированных handle:
- `DEPLOYMENT.md`

---

## Минимальные правила handle (MVP)

- **Длина**: 4–10
- **Алфавит**: `A–Z` и `0–9` (Base36)
- **Нормализация**: uppercase
- **Уникальность**: on-chain (Base)
- **Без привилегий**: нет “элитных”/“премиум” классов, whitelist, аукционов, ручной модерации

Источник истины:
- `MVP_IDENTITY_REGISTRATION_SPEC_v1.md`
- `ADR/ADR-002-name-registry.md`

---

## Как стороннему разработчику “войти в экосистему”

Вам достаточно:
- знать адрес `NameRegistry` и его ABI;
- иметь свой кошелёк на Base (для газа) **или** использовать AA/Paymaster (позже);
- соблюдать правила handle.

### 1) Проверить, занят ли handle

Простой JS пример (ethers v6):

```js
import { ethers } from "ethers";

const NAME_REGISTRY = "0x87B9fD42553067BeBc2Abf42c7045C8F2F7D1A79";
const ABI = [
  "function isAvailable(string handle) view returns (bool)",
  "function resolve(string handle) view returns (address)",
];

const provider = new ethers.JsonRpcProvider("https://mainnet.base.org");
const registry = new ethers.Contract(NAME_REGISTRY, ABI, provider);

const handle = "GUVUIK";
console.log("available:", await registry.isAvailable(handle));
console.log("controller:", await registry.resolve(handle));
```

### 2) Зарегистрировать handle (варианты)

- **Через нашу фабрику** (создаёт `LogosAccount` и регистрирует handle атомарно):  
  смотрите `DEPLOYMENT.md` → “Как зарегистрировать handle”.
- **Через свою реализацию**: вы можете разворачивать свой “controller” (EOA или контракт) и зарегистрировать его в реестре.

---

## Как проверить, что Logos существует (на блокчейне)

1) Резолвим handle:
- `resolve("HANDLE") → controller`

2) Проверяем, что `controller != 0x000…000`  
3) Опционально: проверяем код по адресу контроллера (контракт это или EOA)

Под рукой:
- `DEPLOYMENT.md`

---

## Где читать спеки и ADR

- **Манифест:** `manifesto-ecosystem-logos-v2.1.md`
- **System prompt:** `system-prompt-logos-ecosystem-v2.1.md`
- **Большая архитектура (long-term):** `LOGOS_ECOSYSTEM_SPECIFICATION.md`
- **MVP Identity (текущее “как работает”):** `MVP_IDENTITY_REGISTRATION_SPEC_v1.md`
- **MVP AI (следующий этап):** `MVP_LOGOS_AI_SPEC_v1.md`
- **ADR:**
  - `ADR/ADR-001-controller.md`
  - `ADR/ADR-002-name-registry.md`
  - `ADR/ADR-003-paymaster-treasury.md`
  - `ADR/ADR-004-two-key-architecture.md`

---

## Как контрибьютить (в т.ч. через Cursor)

См. `CONTRIBUTING.md` — там:
- процесс Spec Driven Development (спека/ADR → код);
- правила PR;
- как форкнуть и предложить изменения.

**Важно про безопасность:** никогда не коммитьте `.env` и приватные ключи — см. `SECURITY.md`.


