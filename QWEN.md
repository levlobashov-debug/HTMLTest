# AgentArena — Битва AI-агентов

## Project Overview

**AgentArena** — это веб-приложение для сравнения ответов различных AI-агентов (GPT-4o, GPT-4 Turbo, GPT-4o Mini) на одни и те же задачи. Приложение предоставляет интерфейс для:

- **Арена** — отправка задачи нескольким AI-агентам и сравнение их ответов side-by-side
- **Дебаты агентов** — автоматизированные дебаты между AI-агентами с критическим анализом от AI-судьи
- **Мои агенты** — создание и настройка кастомных AI-агентов с уникальными промптами
- **История** — просмотр прошлых задач и дебатов
- **Рейтинг** — глобальный рейтинг агентов на основе пользовательских оценок

### Технологии

- **Frontend**: Vanilla JavaScript (ES6+), HTML5, CSS3 с CSS Custom Properties
- **Backend/DB**: Supabase (PostgreSQL, Auth)
- **AI API**: Cloudflare Worker прокси (`CONFIG.PROXY_URL`) для вызовов OpenAI API
- **Шрифты**: JetBrains Mono, Outfit (Google Fonts)
- **Без фреймворков**: полностью на нативном JS, без сборщиков

## Architecture

### Файловая структура

```
agentarena/
├── index.html       # Главная HTML-страница (SPA-подобная навигация)
├── style.css        # Все стили (CSS Variables, Grid, Flexbox)
├── config.js        # Конфигурация (Supabase URL/Key, PROXY_URL, BASE_AGENTS)
├── app.js           # Ядро: auth, навигация, глобальные переменные, agent selector
├── arena.js         # Логика арены: runTask, callAgent, renderResults, rateResponse
├── debate.js        # Дебаты агентов: multi-round debate с AI-судьёй
├── agents.js        # CRUD кастомных агентов
├── history.js       # История задач и дебатов
├── rating.js        # Глобальный рейтинг агентов
└── init.js          # Инициализация: проверка зависимостей, автологин
```

### Порядок загрузки JS (критичен!)

```html
<script src="config.js"></script>
<script src="app.js"></script>
<script src="arena.js"></script>
<script src="agents.js"></script>
<script src="history.js"></script>
<script src="rating.js"></script>
<script src="debate.js"></script>
<script src="init.js"></script>
```

Каждый файл зависит от предыдущих (указано в комментариях в начале каждого файла).

### Глобальные переменные и функции

| Symbol | Source | Description |
|--------|--------|-------------|
| `CONFIG` | config.js | Supabase credentials, PROXY_URL |
| `BASE_AGENTS` | config.js | Предустановленные агенты |
| `sb` | app.js | Supabase client |
| `allAgents` | app.js | Все агенты (base + custom) |
| `customAgents` | app.js | Кастомные агенты из БД |
| `currentUser` | app.js | Текущий авторизованный пользователь |
| `selectedAgents` | app.js | Set выбранных ID агентов для арены |
| `enterApp(user)` | app.js | Вход в приложение после авторизации |
| `exitApp()` | app.js | Выход и сброс состояния |
| `loadCustomAgents()` | app.js | Загрузка кастомных агентов из Supabase |
| `renderAgentSelector()` | app.js | Рендер селектора агентов на арене |
| `runTask()` | arena.js | Запуск задачи на арене |
| `callAgent(agent, prompt)` | arena.js | Вызов одного агента через прокси |
| `renderResults(cards)` | arena.js | Рендер карточек результатов |
| `rateResponse(btn)` | arena.js | Оценка ответа агента (1-5 звёзд) |
| `initDebate()` | debate.js | Инициализация страницы дебатов |
| `renderMyAgents()` | agents.js | Рендер страницы моих агентов |
| `openAgentForm(agent)` | agents.js | Открыть форму создания/редактирования |
| `loadHistory()` | history.js | Загрузка и рендер истории |
| `openDetail(taskId)` | history.js | Открыть детали задачи |
| `loadRatings()` | rating.js | Загрузка и рендер рейтинга |

## Supabase Schema

Приложение ожидает следующие таблицы в Supabase:

### `tasks`
- `id` (uuid, primary key)
- `user_id` (uuid, foreign key → auth.users)
- `prompt` (text)
- `agents` (text[] / jsonb — массив ID агентов)
- `created_at` (timestamp)

### `agent_responses`
- `id` (uuid, primary key)
- `task_id` (uuid, foreign key → tasks)
- `agent_id` (text)
- `response_text` (text)
- `response_time_ms` (integer)
- `created_at` (timestamp)

### `agent_ratings`
- `user_id` (uuid)
- `response_id` (uuid, foreign key → agent_responses)
- `score` (integer, 1-5)
- Compound unique constraint: `(user_id, response_id)`

### `custom_agents`
- `id` (uuid, primary key)
- `owner_id` (uuid, foreign key → auth.users)
- `name` (text)
- `description` (text)
- `emoji` (text)
- `color` (text)
- `model` (text — OpenAI model name)
- `system_prompt` (text)
- `is_public` (boolean)
- `created_at` (timestamp)

### `debates`
- `id` (uuid, primary key)
- `user_id` (uuid)
- `topic` (text)
- `agent_ids` (text[] / jsonb)
- `judge_model` (text)
- `max_rounds` (integer)
- `current_round` (integer)
- `status` (text: 'running', 'done', 'paused')
- `created_at` (timestamp)

### `debate_rounds`
- `id` (uuid, primary key)
- `debate_id` (uuid, foreign key → debates)
- `round_number` (integer)

### `debate_messages`
- `id` (uuid, primary key)
- `round_id` (uuid, foreign key → debate_rounds)
- `agent_id` (text)
- `content` (text)

### `debate_verdicts`
- `id` (uuid, primary key)
- `debate_id` (uuid)
- `round_number` (integer)
- `consensus_reached` (boolean)
- `contradictions` (jsonb)
- `verdict_text` (text)
- `agent_scores` (jsonb)

## Building and Running

### Локальный запуск

Приложение не требует сборки. Просто откройте `index.html` в браузере:

```bash
# Вариант 1: напрямую (file:// протокол)
# Откройте index.html в браузере

# Вариант 2: локальный сервер (рекомендуется)
npx serve .
# или
python -m http.server 8000
```

### Развёртывание

Все файлы загружаются как статический сайт (Cloudflare Pages, Netlify, GitHub Pages, или любой хостинг).

### Зависимости

- **Supabase JS SDK** — загружается через CDN (`supabase.min.js`)
- **Google Fonts** — JetBrains Mono, Outfit
- **Cloudflare Worker** — прокси для OpenAI API (URL в `config.js`)

## Development Conventions

### Стиль кода

- **Vanilla JS**, без фреймворков и сборщиков
- **Стрелочные функции** (`async/await` везде)
- **Минималистичный**: нет линтеров, форматтеров, или тест-фреймворков
- **Комментарии-заголовки** в начале каждого файла описывают зависимости и экспорты

### Паттерны

- **Утилиты**: `$` (getElementById), `esc` (HTML escaping), `escAttr` (attribute escaping), `findAgent` (поиск агента по ID)
- **CSS Variables**: вся кастомизация через `:root` переменные
- **SPA-навигация**: переключение `.page.active` и `.nav-btn.active`
- **Supabase**: прямой вызов `sb.from('table')` с async/await
- **Error handling**: `try/catch` с отображением ошибок в UI

### Цветовая схема

Тёмная тема с акцентами:
- `--accent`: `#f76826` (оранжевый)
- `--accent2`: `#22d3ee` (cyan)
- `--bg-deep`: `#06080c` (фон)
- `--bg-panel`: `#0d1117` (панели)
- `--bg-card`: `#151b25` (карточки)

## Key Features

### Арена
1. Вводишь задачу → выбираешь агентов → запускаешь
2. Все агенты вызываются параллельно (`Promise.allSettled`)
3. Ответы отображаются в карточках с рейтингом (1-5 ★)
4. Результаты сохраняются в БД

### Дебаты агентов
1. Выбираешь тему, 2+ агентов, модель судьи, кол-во раундов
2. Агенты играют роли (Адвокат А, Адвокат Б, Критический аналитик)
3. После каждого раунда AI-судья анализирует аргументы по 5 критериям:
   - Факты, Логика, Практичность, Работа с критикой, Полезность
4. scoreboard показывает прогресс каждого агента
5. Финальная панель с синтезом, заметками, возможностью продолжить

### Кастомные агенты
- Создание с именем, эмодзи, цветом, моделью, системным промптом
- Публичные/приватные (видно другим пользователям как "агенты сообщества")

## Configuration

Для изменения настроек отредактируйте `config.js`:

```js
const CONFIG = {
  SUPABASE_URL: 'your-supabase-url',
  SUPABASE_KEY: 'your-supabase-anon-key',
  PROXY_URL: 'your-cloudflare-worker-url',
  VERSION: 1
};
```

Для добавления новых моделей — расширьте `BASE_AGENTS` и обновите `<select>` в `index.html`.
