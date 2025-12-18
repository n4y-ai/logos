# ADR-002: On-chain NameRegistry (handle → controller) на Base, без transfer

## Статус
✅ **Принято и реализовано** (2024-12-17)

## Deployment

| Сеть | Адрес | TX |
|------|-------|-----|
| **Base Mainnet (V2, production)** | `0x87B9fD42553067BeBc2Abf42c7045C8F2F7D1A79` | см. `DEPLOYMENT.md` |
| Base Mainnet (V1, legacy) | `0x96A6802D7721016bB9c1181aaf95900335734115` | [0xec5f35d0...](https://basescan.org/tx/0xec5f35d04f72347cda63bd219c3fc3b526a923490e8b539dfef2723489c6312d) |

**Исходный код (V2):** `logos-protocol/contracts/core/NameRegistry.sol`

## Контекст
Требования MVP к имени (handle):
- Пользователь видит **только handle**.
- Handle: Base36 `A–Z` и `0–9`, нормализация uppercase, длина **4–10**.
- Default handle: **6 символов**, генерируется локально/детерминированно из публичного идентификатора контроллера, с проверкой занятости.
- Уникальность: **on-chain реестр в сети Base**.
- Непередаваемость: **без transfer/перепродажи** на уровне протокола.

**Источники истины:**
- `n4y.ai/logos/.ai-knowledge.md` — раздел **4**
- `n4y.ai/logos/MVP_IDENTITY_REGISTRATION_SPEC_v1.md` — разделы **4**, **6**, **7.1**, **9.2**, **11**

## Решение
Делаем минимальный контракт `NameRegistry`:
- Хранит `handleHash (bytes32) -> controller (address)`.
- Публичные операции:
  - `isAvailable(handle) -> bool`
  - `resolve(handle) -> controller`
  - `registerName(handle, controller)` — **однократная** регистрация свободного handle на указанный `controller`.
- Непередаваемость:
  - Нет функций transfer/update/approve; зарегистрированное имя нельзя перепривязать.
- Нормализация:
  - Контракт принимает строку, нормализует к uppercase и проверяет допустимый алфавит/длину (4–10).
  - `handleHash = keccak256(normalizedHandleBytes)`.

## Почему это соответствует спекам
- Реестр обеспечивает on-chain уникальность (см. `MVP_IDENTITY_REGISTRATION_SPEC_v1.md` §7.1).
- Переносимость/неприсвоение: отсутствие transfer на уровне контракта (см. §4, §7.1).
- Ограничения строго формальные (алфавит/длина/занятость), без “админских политик”.

## Последствия
- Хеширование означает, что контракт не обязан хранить исходную строку; UX остаётся off-chain.
- On-chain проверки детерминированы и минимальны по поверхности атаки.

## Acceptance criteria (проверяемо)
- `registerName("NEXO")` проходит только если имя свободно и соответствует правилам; повторная регистрация того же handle невозможна.
- `isAvailable(handle)` и `resolve(handle)` работают только через нормализованный формат (lowercase вход должен приводиться к uppercase).


