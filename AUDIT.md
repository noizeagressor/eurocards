# Frontend Audit: EuroCards Zen Premium
> Telegram Mini App — чистый HTML/CSS/JS, MVP
> Дата аудита: 2026-02-25

---

## Сильные стороны

**1. Самодостаточный деплой без сборки**
Весь проект — один `index.html`. Для Telegram Mini App это оправданная архитектура MVP: деплой через GitHub Pages, нулевой CI/CD-оверхед, мгновенные итерации.

**2. Продуманная дизайн-система**
CSS-переменные (`--accent-green`, `--bg-dark`, `--glass-border`) определены в `:root` и последовательно используются. Есть брендовая палитра, кастомные анимации (`fadeIn`, `slideUp`, `checkShort`/`checkLong`), единый стиль card-компонентов.

**3. Telegram WebApp API интегрирован правильно**
`tg.ready()`, `tg.expand()`, `HapticFeedback`, `openTelegramLink()`, `initDataUnsafe.user` — всё использовано корректно и по назначению.

**4. UX-детали на уровне**
- Haptic feedback на ключевых действиях (отправка, промокод, копирование)
- Age-gate (18+) с визуальной валидацией до сабмита
- Промо-код со стрйкзру-ценой и визуальным подтверждением
- Drawer-паттерн с анимацией + backdrop overlay

**5. Defensive coding в ключевых местах**
`loadPythonData()` обёрнут в `try/catch`. Nullable-chain `tg.initDataUnsafe?.user?.id` защищает от краша без контекста Telegram. Fallback-состояние "emptyOrders" для пустого списка заказов.

**6. Структурированные данные `bankData`**
Данные о продуктах изолированы в объект-конфиг, а не разбросаны по HTML. Это будет легко вынести в JSON-файл или API при подключении бэкенда.

---

## Критические недостатки

### 1. СЛОМАННЫЙ POST-запрос (`no-cors` + JSON) — заявки могут не доходить

```javascript
// ПРОБЛЕМА: эти два параметра несовместимы
await fetch(GOOGLE_SCRIPT_URL, {
    method: 'POST',
    mode: 'no-cors',                               // ← режим "opaque"
    headers: { 'Content-Type': 'application/json' }, // ← БУДЕТ УДАЛЁН браузером
    body: JSON.stringify(orderPayload)
});
```

Режим `no-cors` автоматически удаляет все "непростые" заголовки, включая `Content-Type: application/json`. Google Apps Script получит тело без указания типа и может его не распознать. При этом `response` всегда будет `opaque` (статус 0, тело пустое) — **вы никогда не узнаете, дошла ли заявка**. Success overlay показывается безусловно даже при сетевой ошибке.

**Исправление при подключении бэкенда:** убрать `mode: 'no-cors'`, настроить CORS на стороне сервера, обрабатывать `response.ok`.

---

### 2. XSS-уязвимость в `renderOrders()` — при подключении бэкенда критично

```javascript
// Данные от бэкенда вставляются напрямую в innerHTML
container.innerHTML = orders.map(order => `
    <h3 ...>${order.name || 'Тариф'}</h3>  // ← XSS если бэкенд скомпрометирован
    <span ...>${order.status || 'В обработке'}</span>
`).join('');
```

Сейчас данные приходят через URL-параметр `orders` (base64). Если бэкенд вернёт `order.name = '<script>...'` или `order.status = '<img onerror=...>'` — скрипт выполнится в контексте Mini App с доступом к `tg.initDataUnsafe`.

**Исправление:** заменить `innerHTML` на `textContent` / `innerText` или использовать безопасный sanitizer.

---

### 3. Хардкод прomo-кода виден всем пользователям

```javascript
if (code === "AVITO1000") { // ← виден в DevTools любому пользователю
```

Любой пользователь может открыть исходный код в DevTools и узнать промокод без реферального приглашения.

---

### 4. Захардкоженный курс EUR теряет актуальность

```javascript
let currentEurRate = 90.31; // ← не обновляется
```

Курс будет неверным уже через несколько дней. Конвертер в drawer будет показывать ложные данные и подрывать доверие.

---

### 5. Fallback Telegram ID раскрывает внутренний ID

```javascript
const link = `https://t.me/eurocardsapp_bot/?startapp=${tg.initDataUnsafe?.user?.id || '8829102'}`;
//                                                                                      ↑ hardcoded real ID
```

При открытии вне Telegram (например, в браузере при тестировании) будет использован реальный чужой Telegram ID в реферальной ссылке.

---

### 6. Устаревший API `document.execCommand('copy')`

```javascript
document.execCommand('copy'); // ← deprecated, удалён из спецификации
```

Работает в большинстве браузеров сейчас, но может перестать работать. Современная замена: `navigator.clipboard.writeText()`.

---

### 7. Отсутствие валидации ФИО перед отправкой

```javascript
if (!fio || !dob || !validateAge()) { ... return; }
```

Проверяется только "не пустое", но не формат. Можно отправить `"123"` или emoji как имя. При подключении реального бэкенда это нужно валидировать.

---

### 8. Опасный `innerHTML` в `openVaultDrawer()`

```javascript
document.getElementById('drawerDynamicDetails').innerHTML = `<p>${data.desc}</p>`;
```

Сейчас `data.desc` из локального `bankData` — безопасно. Но если в будущем описания начнут приходить с бэкенда, здесь откроется XSS.

---

## План улучшений (Roadmap)

### Шаг 1: Исправить сломанный POST-запрос (приоритет: КРИТИЧНО)

Убрать `mode: 'no-cors'` и добавить обработку реального ответа сервера. До момента подключения собственного бэкенда — передавать данные как `FormData` (это "простой" запрос, не требует preflight и работает с Apps Script):

```javascript
// Вместо no-cors + JSON
const form = new FormData();
Object.entries(orderPayload).forEach(([k, v]) => form.append(k, v));
const response = await fetch(GOOGLE_SCRIPT_URL, { method: 'POST', body: form });
if (!response.ok) throw new Error('Server error');
// Теперь можно читать ответ
```

---

### Шаг 2: Санитизация данных от бэкенда в `renderOrders()`

Заменить innerHTML-шаблон на безопасное DOM-строительство:

```javascript
function createOrderCard(order) {
    const card = document.createElement('div');
    card.className = 'flex flex-col p-5 bg-white/5 rounded-[22px] border border-white/5 text-left mb-3';

    const num = document.createElement('span');
    num.className = 'text-xs font-black uppercase text-[#9FE870] tracking-widest';
    num.textContent = order.num || 'ЗАКАЗ'; // textContent, не innerHTML

    // ... остальные поля
    card.appendChild(num);
    return card;
}
```

---

### Шаг 3: Вынести конфигурацию в верхний блок `<script>` отдельным объектом `CONFIG`

Сгруппировать все константы вместо разброса по файлу:

```javascript
const CONFIG = {
    GOOGLE_SCRIPT_URL: "https://script.google.com/...",
    EUR_RATE_FALLBACK: 90.31,    // использовать до получения актуального с API
    BOT_USERNAME: "eurocardsapp_bot",
    PROMO_CODE: "AVITO1000",     // TODO: перенести на бэкенд при интеграции
    PROMO_DISCOUNT: 1000,
};
```

---

### Шаг 4: Заменить все inline `onclick` на `addEventListener` в блоке инициализации

```javascript
// Вместо onclick="switchTab('home')" в HTML
window.addEventListener('DOMContentLoaded', () => {
    document.getElementById('nav-home').addEventListener('click', () => switchTab('home'));
    document.getElementById('nav-orders').addEventListener('click', () => switchTab('orders'));
    // ...
});
```

Это позволит явно видеть все обработчики в JS, а не искать их по всему HTML.

---

### Шаг 5: Заменить `document.execCommand('copy')` на `Clipboard API`

```javascript
async function copyRefLink() {
    const link = `https://t.me/${CONFIG.BOT_USERNAME}/?startapp=${tg.initDataUnsafe?.user?.id ?? ''}`;
    try {
        await navigator.clipboard.writeText(link);
        tg.HapticFeedback.notificationOccurred('success');
    } catch {
        // Fallback для старых браузеров
        const input = document.createElement('input');
        input.value = link;
        document.body.appendChild(input);
        input.select();
        document.execCommand('copy');
        document.body.removeChild(input);
    }
}
```

---

### Шаг 6: Вынести `bankData` и `logos` в отдельный `<script type="application/json">` или файл `data.js`

```html
<!-- В <head> -->
<script id="bankDataJson" type="application/json">
{
    "skrill": { "desc": "...", "delivery": "от 2 часов", "features": [...] },
    ...
}
</script>
```

```javascript
// В основном скрипте
const bankData = JSON.parse(document.getElementById('bankDataJson').textContent);
```

Или создать отдельный файл `data.js` с `export const bankData = {...}` — он напрямую станет модулем React при миграции.

---

### Шаг 7: Убрать задублированные классы и починить минорные HTML-баги

В коде есть повторяющиеся классы и опечатки:

```html
<!-- Дублирование text-center в одном элементе -->
<div id="pricePromoContainer" class="hidden flex flex-col items-center text-center text-center">

<!-- Дублирование text-left -->
<label class="order-label text-left text-left">Комментарий</label>

<!-- Опечатка в данных банка -->
"Личныый IBAN и SWIFT"  <!-- ← два "ы" в слове "Личный" -->
```

Провести прогон по всему файлу с поиском `text-center text-center` и аналогичных дублей.

---

## Pre-React рекомендации

### 1. Разделить один HTML на логические "компоненты" уже сейчас

Каждый раздел — будущий React-компонент. Добавить комментарии-границы и вынести в отдельные `<template>` или отдельные файлы-заготовки:

```
index.html
├── <!-- COMPONENT: HomeTab -->
├── <!-- COMPONENT: BonusesTab -->
├── <!-- COMPONENT: OrdersTab -->
├── <!-- COMPONENT: SettingsTab -->
├── <!-- COMPONENT: ProductDrawer -->
├── <!-- COMPONENT: OrderFormDrawer -->
└── <!-- COMPONENT: SuccessOverlay -->
```

---

### 2. Перейти на BEM для пользовательских классов (не Tailwind)

Сейчас кастомные классы имеют плоские имена: `.vault-card`, `.nav-dock`, `.drawer`. В React с CSS Modules или styled-components BEM читается как:

```css
/* Сейчас */
.vault-card { }
.vault-card.skrill-gradient { }  /* плохо — mix of semantic and style */

/* BEM для React-миграции */
.vault-card { }
.vault-card--skrill { }
.vault-card__price { }
.vault-card__delivery { }
```

---

### 3. Заменить глобальное состояние на State-объект уже сейчас

Вместо 6 разрозненных глобальных переменных — один объект:

```javascript
// Сейчас (плохо для React)
let currentBankId = "";
let basePrice = 0;
let finalPrice = 0;
let promoApplied = false;
let activeTab = 'home';
let currentEurRate = 90.31;

// Pre-React паттерн (легко мигрирует в useState)
const state = {
    currentBankId: "",
    basePrice: 0,
    finalPrice: 0,
    promoApplied: false,
    activeTab: 'home',
    eurRate: 90.31,
};
function setState(patch) {
    Object.assign(state, patch);
}
```

При миграции `state.X` → `const [x, setX] = useState(...)`, `setState` → `setX`.

---

### 4. Инкапсулировать каждую логическую область в IIFE-модуль или объект

```javascript
// Сейчас функции в глобальном scope
function switchTab(id) { ... }
function handleCardClick(id, asset, priceNum) { ... }

// Pre-React: логические модули (легко станут хуками useTab, useDrawer)
const TabManager = {
    active: 'home',
    switch(id) { ... }
};

const DrawerManager = {
    open(bankId, asset) { ... },
    close() { ... },
    setTab(t) { ... }
};

const OrderManager = {
    currentBankId: '',
    basePrice: 0,
    finalPrice: 0,
    promoApplied: false,
    checkPromo() { ... },
    submit() { ... }
};

const AiChat = {
    messages: [],
    ask(prompt) { ... },
    addMessage(text, role) { ... }
};
```

---

### 5. Создать `js/api.js` как точку входа для всех HTTP-вызовов

```javascript
// js/api.js — в будущем станет слоем сервисов React
const Api = {
    ENDPOINT: "https://script.google.com/macros/s/...",

    async submitOrder(payload) {
        const form = new FormData();
        Object.entries(payload).forEach(([k, v]) => form.append(k, v));
        const res = await fetch(this.ENDPOINT, { method: 'POST', body: form });
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
    },

    async askAi(prompt) {
        const res = await fetch(this.ENDPOINT, {
            method: 'POST',
            body: JSON.stringify({ action: 'ask_ai', prompt })
        });
        return res.json();
    },

    async getEurRate() {
        // TODO: подключить реальный API курсов
        return 90.31;
    }
};
```

Даже находясь в том же `index.html`, этот модуль полностью готов к `import api from './api.js'` в React.

---

### 6. Перейти с Tailwind CDN на Tailwind CLI (сборка)

```bash
# Установить один раз
npm init -y
npm install tailwindcss --save-dev
npx tailwindcss init

# tailwind.config.js
module.exports = { content: ["./index.html"] }

# Добавить в package.json
"scripts": {
    "build:css": "tailwindcss -i ./src/input.css -o ./dist/output.css --minify",
    "watch": "tailwindcss -i ./src/input.css -o ./dist/output.css --watch"
}
```

Это уберёт 300KB CDN-скрипта, создаст `package.json` (точка входа для будущего Vite/Create React App) и улучшит производительность загрузки в Telegram.

---

## Итого по приоритетам

| Приоритет | Задача | Сложность |
|-----------|--------|-----------|
| 🔴 Критично | Исправить `no-cors` POST | Низкая |
| 🔴 Критично | Санитизация `renderOrders()` innerHTML | Низкая |
| 🟠 Высокий | Вынести `CONFIG` объект | Низкая |
| 🟠 Высокий | Убрать прomo-код с фронта | Средняя (нужен бэкенд) |
| 🟡 Средний | Заменить глобалы на `state` объект | Низкая |
| 🟡 Средний | Инкапсуляция в модульные объекты | Средняя |
| 🟡 Средний | `Api` объект для HTTP | Низкая |
| 🟢 Низкий | Clipboard API вместо execCommand | Низкая |
| 🟢 Низкий | Tailwind CLI вместо CDN | Средняя |
| 🟢 Низкий | BEM для кастомных классов | Средняя |
| 🟢 Низкий | Убрать дубли классов/опечатки | Низкая |
