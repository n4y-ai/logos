# LOGOS_TOOLS_GMAIL_TELEGRAM_SPEC_v1 — Инструменты Gmail/IMAP и Telegram для Logos

**Статус:** 📋 DRAFT  
**Версия:** 1.0  
**Дата:** 2025-12-22  
**Цель:** стандартизировать подключение и работу Logos с почтой и Telegram так, чтобы:
- доступы выдаёт только владелец,
- Logos может читать/классифицировать/предлагать действия,
- любые “опасные” действия выполняются только после подтверждения владельца,
- данные пригодны для памяти (RAG/Graph) с provenance.

---

## 1) Модель: Tools как “Коннекторы”, а Logos Runtime как “Оркестратор”

- **Tool Connector** отвечает за протокол (OAuth/IMAP/Telegram) и безопасное хранение токенов.
- **Logos Runtime** вызывает tool через абстрактные операции (`list`, `get`, `draft`, `send`, `reply`, `mute`).
- Все результаты tool‑операций возвращают **структурные артефакты** + provenance.

---

## 2) Требования безопасности (Owner Control Plane)

### 2.1 Подключение/отключение источников
- **MUST** выполняться только владельцем.
- **MUST** требовать “владельческое подтверждение”:
  - в MVP: owner session (веб‑вход),
  - в будущем: owner signature.

### 2.2 Токены и секреты
- **MUST NOT** хранить секреты в git.
- **MUST** шифровать at-rest (прод: KMS/Vault; MVP: хотя бы encrypt‑key‑file).

### 2.3 Опасные действия требуют подтверждения

По умолчанию в MVP **только чтение/классификация/черновики** могут происходить автономно.

**Требуют подтверждения владельца:**
- отправка письма (`email.send`)
- ответ/сообщение в Telegram (`tg.send`, `tg.reply`)
- удаление писем (`email.delete`)
- архивирование/перемещение в папку (`email.archive`, `email.move`) — можно сделать “опционально безопасным” позже
- блокировка/мьют (`tg.mute`, `tg.block`)

**Разрешены без подтверждения:**
- чтение метаданных/контента (`email.get`, `tg.get`)
- классификация/теги локально (`email.classify`, `tg.classify`)
- создание черновика (`email.draft`)
- генерация предложений ответа (как черновик)

---

## 3) Gmail/IMAP Tool

### 3.1 Поддерживаемые режимы

**Режим A (предпочтительный): Gmail OAuth**
- плюсы: безопасно, управляемые scopes, простая отзывчивость.
- минусы: зависит от Google OAuth flow.

**Режим B (универсальный): IMAP/POP3**
- плюсы: работает почти со всем.
- минусы: хранение пароль/ключа, меньше “богатых” метаданных.

В MVP допускается начать с Gmail OAuth, а IMAP добавить как fallback.

### 3.2 Domain модель Email‑артефакта

```json
{
  "id": "email:provider:messageId",
  "threadId": "string",
  "from": { "name": "string", "email": "string" },
  "to": [{ "name": "string", "email": "string" }],
  "cc": [{ "name": "string", "email": "string" }],
  "subject": "string",
  "snippet": "string",
  "body": { "text": "string", "html": "string|null" },
  "receivedAt": "iso",
  "labels": ["string"],
  "attachments": [
    { "id": "string", "filename": "string", "mime": "string", "size": 123 }
  ],
  "provenance": {
    "source": "gmail|imap",
    "account": "redacted",
    "rawId": "string"
  }
}
```

### 3.3 API операций (концептуально)

**Чтение**
- `email.list({ since, query, limit }) → Email[]`
- `email.get(emailId) → Email`

**Черновики**
- `email.draft({ to, subject, body, threadId? }) → Draft`
- `email.updateDraft(draftId, patch) → Draft`

**Опасные**
- `email.send(draftId|payload)` — требует подтверждения владельца
- `email.delete(emailId)` — требует подтверждения владельца
- `email.archive(emailId)` — требует подтверждения в MVP

### 3.4 Рекомендации “входящих”

Каждое письмо получает:
- `category`, `urgency`, `actionProposal[]`, `confidence`
- `why` (коротко)
- `provenance` (emailId, threadId)

---

## 4) Telegram Tool

### 4.1 Поддерживаемые режимы (MVP)

**Режим A (MVP‑реалистичный): Telegram Bot API**
- обработка событий: каналы/группы/личка с ботом,
- бот должен быть добавлен владельцем.

**Режим B (расширение): userbot**
- даёт доступ к личным сообщениям, но сильно сложнее в эксплуатации и рисках.
В MVP рекомендуется начинать с Bot API.

### 4.2 Domain модель Telegram‑события

```json
{
  "id": "tg:chatId:messageId",
  "chat": { "id": "string", "type": "private|group|supergroup|channel", "title": "string|null" },
  "from": { "id": "string", "username": "string|null", "displayName": "string|null" },
  "text": "string|null",
  "attachments": [{ "type": "photo|doc|voice|video", "id": "string", "size": 123 }],
  "sentAt": "iso",
  "provenance": {
    "source": "telegram",
    "chatId": "string",
    "messageId": "string"
  }
}
```

### 4.3 Операции (концептуально)

**Чтение/подписки**
- `tg.subscribe(chatId, filters?)`
- `tg.get(eventId)` / `tg.list({ since, chatId, limit })`

**Опасные**
- `tg.send(chatId, text)` — требует подтверждения владельца в MVP
- `tg.reply(eventId, text)` — требует подтверждения владельца в MVP
- `tg.mute(chatId)` — требует подтверждения владельца

### 4.4 Рекомендации по сообщениям
Для каждого события:
- “что это”, “насколько срочно”, “что предлагаешь сделать”, “почему”

---

## 5) Подтверждение действий (Intent → Approve → Execute)

### 5.1 Модель
1) Logos формирует **Intent** на опасное действие (send/reply/delete).
2) Intent показывается владельцу в UI.
3) Владелец: Approve/Reject (+ опционально правка).
4) Только после Approve tool выполняет действие.

### 5.2 Intent поля (минимум)
- `intentId`
- `actionType` (`email.send`, `tg.reply`, ...)
- `payloadPreview`
- `riskNotes`
- `provenanceRefs` (ссылки на emailId/messageId)

---

## 6) Связь с памятью (RAG/Graph)

Из Gmail/Telegram поступают:
- **документы** (контент письма/сообщения) → Vector RAG
- **сущности** (люди, проекты, договорённости) → Graph

**MUST** хранить provenance:
- откуда взята информация (emailId/threadId/chatId/messageId)
- когда
- какую версию обработали

Детали — `LOGOS_MEMORY_SPEC_v1.md`.

---

## 7) Acceptance Criteria

- [ ] Владелец может подключить Gmail/IMAP и Telegram.
- [ ] Logos получает входящие (email + tg) и делает triage: категории/срочность/предложения.
- [ ] Любое send/reply/delete требует подтверждения владельца.
- [ ] Любой артефакт для памяти содержит provenance.

---

## 8) Связанные документы

- `LOGOS_INITIATION_SPEC_v1.md`
- `LOGOS_MEMORY_SPEC_v1.md`
- `LOGOS_ENVELOPE_SPEC_v1.md`


