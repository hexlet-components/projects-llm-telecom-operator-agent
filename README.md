# Шаблон проекта «AI-оператор для сайта мобильного оператора связи»

Проект: агентный контур «AI-чат мобильного оператора». Чат отвечает на вопросы по данным абонента и базе знаний с источниками; действия (смена тарифа, услуги, контакты) проходят путь прав: JWT в каждом вызове → allowlist и два барьера → политики Dogwood (подтверждение «да — что да?», TTL карточки, лимиты) → идемпотентное исполнение → аудит. Guardrail-сервис проверяет текст, fail-closed для действий. Multi-provider failover — бонус (`app/llm_pool.py` пишется в шаге 8*).

Ваш код помечен `TODO(шаг N)` в `app/`, `actions/`, `guardrail/` и `pdp/policies/tariff.dw`; готовое (ядро, моки, стенда-сервисы, фронтенд, smoke) менять не нужно. `tests/smoke.py` — исполняемая спецификация: группы включаются по мере готовности шагов.

## Контур

| Процесс | Порт | Что это | Кто пишет |
|---|---|---|---|
| `make stand` | 8100 | issuer+JWKS, read-MCP (crm/billing/kb), REST-моки, база знаний, веб-чат | готовое |
| `make api` | 8000 | backend: сессии, планировщик → доменные агенты → редактор, SSE | студент |
| `make actions` | 8220 | action-MCP: prepare_*/execute_action, subject из JWT | студент |
| `make pdp` | 9100 | PDP: per-session NativeAuthorizer, политики `pdp/policies/tariff.dw` | сервис готов, политики студент |
| `make guardrail` | 8300 | guardrail: input/output, RULE-движок, admin down/up | студент |
| docker | 5434 | PostgreSQL 16 + pgvector | готовое |

Код студента: `app/` (оркестратор), `actions/`, `pdp/policies/`, `guardrail/`. Готовое: `core/`, `services/`, `stand.py`, `web/`, `kb/vectors.json`, `scripts/run_scenarios.py` (раннер), `data/dialogs/` (примеры сценариев).

## Запуск

```bash
python3.12 -m venv .venv && source .venv/bin/activate && make install
cp .env.example .env        # LLM_BASE_URL / LLM_MODEL / LLM_API_KEY

make infra && make migrate && make seed
make stand                  # терминал 1
make api                    # терминал 2  (шаг 1)
make actions                # терминал 3  (шаг 2)
make pdp                    # терминал 4  (шаг 4)
make guardrail              # терминал 5  (шаг 5)
```

Веб-чат: `http://127.0.0.1:8100` (anna / boris / olga). Страница оператора: `/operator.html`.

## Проверки

```bash
make smoke       # группы 0–5 по шагам: 54 проверки, без LLM; одна группа: make smoke G=0
make scenarios   # матрица диалогов на живой LLM (data/dialogs/*.yaml)
make reset       # вернуть демо-данные моков в исходное
```

## Данные и нюансы

- **Токены**: по одному на MCP-сервер (аудитория = адрес сервера, RFC 8707); истёкший — клиент перевыпускает и повторяет молча (`core/tokens.py`, `core/mcpclient.py`).
- **Все httpx-клиенты** создаются с `trust_env=False` — системный прокси не должен перехватывать localhost.
- **Dogwood**: живые события не несут таймстемпов → окна `formerly` в live не закрываются, TTL карточки проверяет PDP по wall clock; verdict без reason — причины синтезирует PDP (`policy_events`); authorizer однопоточный → один воркер.
- **База знаний**: векторы — снапшот (`kb/vectors.json`, dim=768); пересчитать — `make snapshot` (нужна LM Studio с `text-embedding-embeddinggemma-300m-qat`); без модели поиск лексический.
- **Мина в базе знаний**: карточка «Про» содержит indirect injection — учебная: исполнение невозможно без явного «да», output-guard ловит формулировки.
- **Дневной лимит смен тарифа** стоит на стороне биллинга (дублирует политику): скомпрометированный вызывающий обойти его оркестратором не может.

## Пользователи

| Логин | sub | Роль | Права |
|---|---|---|---|
| anna | u_4f2a | client | чтение + `tariffs:write` (карточные действия) |
| boris | u_9b1c | client | только чтение |
| olga | u_op7 | operator | обзор абонентов, без write |
