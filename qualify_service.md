### ТЗ: MVP «Qualify Company Service» (rule → embedding → LLM-optional)

### 1. Цель

Сервис принимает данные о компании (Google/LinkedIn/сайт) и ICP-профиль, рассчитывает соответствие компании ICP и возвращает: метку, уверенность, объяснение, сигналы и эмбеддинг. Архитектура: **rule-based → embedding → LLM (переключаемо в конфиге)**.

---

### 2. Контракты API

#### 2.1 `POST /qualify`

* **Описание:** синхронная квалификация одной компании.
* **Вход:** **ровно** структура, которую ты утвердил (см. ниже).
* **Выход:** структура результата, которую ты утвердил.
* **Коды:** `200`, `400` (валидация), `409` (idempotency конфликт формата), `422` (неполный ICP), `500`.

#### 2.2 `POST /qualify/batch`

* **Описание:** массив входов (до 200 элементов).
* **Вход:** `[{…как /qualify…}]`
* **Выход:** массив ответов в исходном порядке; для упавших — объект `{"error":{code,message}}`.
* **Коды:** `200`, `207 Multi-Status` (если часть упала).

#### 2.3 `GET /healthz` / `GET /livez` / `GET /readyz`

* Проверки приложения, загрузки моделей, наличия центроидов для заявленного `icp.icp_id`.

---

### 3. Вход/Выход (фиксируем)

**Input**:

```json


{
  "idempotency_key": "hash(project44.com|2025-10-02)",
  "fetched_at": "2025-10-02T06:20:00Z",
  "primary_domain": "project44.com",

  "company": {
    "name": "project44",
    "country_code": "US",
    "website_url": "https://www.project44.com",
    "linkedin_url": "https://www.linkedin.com/company/project-44/",
    "language": "en"
  },

  "signals": {
    "website": {
      "title": "Movement. The Decision Intelligence Platform...",
      "content_markdown": "<очищенный markdown>",
      "headings": [{"level": 1, "text": "Decision Intelligence Platform"}],
      "metrics": { "char_len": 24500, "link_ratio": 0.08, "estimated_read_time_sec": 540 }
    },
    "linkedin": {
      "id": "123456",
      "followers": 120000,
      "employees_in_linkedin": 1800,
      "about": "project44 provides real-time visibility...",
      "industries": "Software Development; Transportation/Logistics",
      "company_size": "1001-5000",
      "headquarters": "Chicago, Illinois"
    },
    "google": {
      "keyword": "project44 supply chain",
      "page_title": "Decision Intelligence Platform | project44",
      "organic_top": [
        { "rank": 1, "title": "project44 — Movement", "url": "https://www.project44.com", "description": "Real-time visibility..." },
        { "rank": 2, "title": "About project44", "url": "https://www.project44.com/company", "description": "About p44..." }
      ],
      "related": ["supply chain visibility", "real time tracking"]
    }
  },

  "embedding": {
    "text": "title + H1..H3 + LI about + 2-3 organic descriptions",
    "model": "text-embedding-3-large",
    "vector": null
  },

  "icp": {
    "icp_id": "global.logistics_smb",
    "title": "Logistics — SMB",
    "market": "global",

    "include": {
      "industries": ["Logistics", "Supply Chain", "Transportation"],
      "countries": ["US", "GB", "DE"],
      "size_band": [{"50":"1000"}],
      "keywords": ["visibility", "tracking", "supply chain"],
      "ban_words": ["porn", "tabacco"],
      "description": "We need company with size 1,2,3 etc."
    },

    "scoring": {
      "embedding_model": "bge-m3",
      "alpha": 0.7,
      "beta": 0.3,
      "candidate_labels": ["supply_chain_platform", "ai_analytics_for_logistics", "not_a_fit"]
    }
  },

  "rules":{
   "include_words": ["word1"],
   "exclude_words": ["word2"],
   "employees_in_linkedin": {"min": 1800, "max": 5000}, //optional
  },

  "raw": {
    "google": { "…": "усечённый сырой json" },
    "linkedin": { "…": "усечённый сырой объект" },
    "website": {
      "cleaner_version": "llm:v1",
      "raw_content_sample": "часть raw для аудита"
    }
  }
}
```

**Output**:

```json
{
  "company_id": "hash(project44.com|project44)",
  "name": "project44",
  "primary_domain": "project44.com",
  "industry": "Logistics / Supply Chain",
  "classification": {
    "label": "supply_chain_platform",
    "confidence": 0.91,
    "method": "rule+embedding+llm",
    "explanation": "Detected keywords: 'visibility', 'supply chain AI'"
  },
  "fingerprint": {
    "top_terms": [
      {"term": "visibility", "score": 0.85},
      {"term": "supply chain", "score": 0.82},
      {"term": "logistics", "score": 0.77},
      {"term": "ai disruption", "score": 0.62},
      {"term": "decision intelligence", "score": 0.59}
    ],
    "distinctive_terms": [
      {"term": "tariff analytics", "delta": 0.45},
      {"term": "yard automation", "delta": 0.41},
      {"term": "port intel", "delta": 0.36}
    ],
    "assigned_tags": [
      "Logistics",
      "Visibility",
      "AI/Analytics"
    ]
  },
  "signals": {
    "followers": 120000,
    "employees_in_linkedin": 1800,
    "content_score": 0.82,
    "language": "en"
  },
  "embedding": {
    "model": "text-embedding-3-large",
    "vector": null
  },
  "company": "same as input data",
  "timestamp": "2025-10-02T07:30:00Z",
  "version": "classifier:v2"
}
```
---

### 4. Технологический стек

* **API:** FastAPI + Uvicorn (ASGI), Pydantic v2 для схем/валидации.
* **Embeddings:** sentence-transformers (`bge-m3`/`e5-large-v2`), onnxruntime (FP16/INT8), CPU-first.
* **Rules:** простой движок на Python (веса/порогов нет — всё приходит из `icp` и `rules`).
* **LLM-gate (опц.):** Используем deepseek v3 на базе dashscope alibaba cloud.
* **Storage (MVP):** in-memory + файловый кэш центроидов (`.npy`). Опционально SQLite/Postgres для кэша и аудита.
* **Логи/метрики:** structlog (JSON), Prometheus client, `/metrics`.
* **Деплой:** Docker, docker-compose; позже Helm.

---

### 5. Данные модели (в коде)

* `QualifyIn` / `QualifyOut`: Pydantic-схемы ровно по контракту.
* `IcpConfigResolved`: нормализованный ICP (industries/countries/size_band → списки/диапазон; candidate_labels).
* `CentroidStore`: мапа `{(icp_id, label) -> np.ndarray}`; на старте загружаем центроиды (файлы в `centroids/{icp_id}/{label}.npy`) или строим из `include.keywords` (см. fallback ниже).
* `EmbeddingRuntime`: обёртка над моделью (encode(text) -> np.ndarray).

---

### 6. Алгоритм расчёта (MVP)

#### 6.1 Нормализация входа

* Проверяем `idempotency_key` (если включим кэш).
* Нормализуем `icp.include.size_band`:

  * если `[{ "50":"1000"}]` → делаем `[50, 1000]`; если список диапазонов — берём первый.
* Парсим `signals.linkedin.industries` в список по `;`/`,`.

#### 6.2 Rule-based score (без весов — просто доля совпадений)

* Источники текста для слов: `website.content_markdown` + `linkedin.about` + `google.page_title` + `organic_top[].title/description`.

* **Позитивные правила (булевые):**

  1. Индустрия совпадает с `icp.include.industries` → `+1`
  2. Страна ∈ `icp.include.countries` → `+1`
  3. Размер сотрудников попадает в `size_band` (если указан и есть `employees_in_linkedin`) → `+1`
  4. В тексте найдены `icp.include.keywords` (≥1 слово) → `+1`
  5. В тексте найдены `rules.include_words` (≥1 слово) → `+1`
  6. `signals.website.metrics.char_len ≥ icp.thresholds.content_len_min` (если порог есть) → `+1`
  7. `followers ≥ icp.thresholds.followers_min` (если есть) → `+1`
  8. `employees_in_linkedin ≥ icp.thresholds.employees_min` (если есть) → `+1`

* **Негативные правила:**

  1. Слова из `icp.include.ban_words` найдены → `-1` (каждый факт — одноразовый штраф)
  2. `rules.exclude_words` найдены → `-1`
  3. `link_ratio > icp.thresholds.link_ratio_max` (если задан) → `-1`

* **Расчёт:**

  ```
  positives = count_true(пункты 1..8)
  negatives = count_true(пункты N1..N3)
  rule_raw = positives - negatives               # ∈ ℤ
  rule_max = 8                                   # число позитивных правил
  rule_score = clip(rule_raw / rule_max, 0.0, 1.0)
  ```

  (Если часть сигналов отсутствует — соответствующее правило просто не учитываем в `rule_max`.)

#### 6.3 Embedding score

* Текст для эмбеддинга: `embedding.vector` если дано, иначе `embedding.text` → `encode()` (нормируем L2).
* Для каждого `label` из `icp.scoring.candidate_labels`:

  * Берём центроид `c_{icp,label}` из `CentroidStore`.

    * **Fallback (если нет файла):** считаем вектор от `join(icp.include.keywords)` и считаем это временным прототипом.
  * Косинус: `sim = dot(v_company, c)`.
  * В [0,1]: `sim01 = (sim + 1)/2`.

#### 6.4 Смешивание и выбор класса

* Для каждого класса `k`:

  ```
  score_k = α * sim01_k + β * rule_score        # α=icp.scoring.alpha, β=icp.scoring.beta
  ```
* Выбираем `label = argmax(score_k)` и `confidence = score_label`.
* **Порог fit/not_fit (для внутренней логики/фильтра):** дефолт 0.65 (глобально в конфиге).

#### 6.5 LLM Gate (опционально, выключено в MVP-конфиге)

* Условие запуска: `(top1 - top2) < gap_threshold` (например, 0.05) и длина контента ≤ лимита.
* Промпт: краткая выжимка (title/H1/LI about/keywords + почему сработали правила).
* Ответ LLM: `confirm|reject|uncertain` + краткое объяснение.
* Коррекция `confidence`: `+0.03 / -0.03` максимум. Метка меняется **только** при `reject` и очень маленьком gap.

---

### 7. Объяснение в ответе (fingerprint/explanation)

* `fingerprint.top_terms`: извлекаем топ-термы из `website.content_markdown` (YAKE/KeyBERT, top-5).
* `fingerprint.distinctive_terms`: отбираем термы, редкие в нашем фоне (MVP: эвристика — слова длиной ≥6, встречающиеся ≤3 раза в тексте, но совпадающие с `icp.include.keywords` или `rules.include_words` → сортировка по tf-idf surrogate).
* `fingerprint.assigned_tags`: маппим найденные ключевые слова в теги (Logistics, AI/Analytics, Visibility) — словарём в коде.
* `classification.explanation`: короткая строка из сработавших правил и top-слов:
  напр. `"industry match; size in [50,1000]; keywords: visibility, tracking"`.

---

### 8. Производительность, лимиты, идемпотентность

* p95 `< 600ms` на CPU при одной модели эмбеддинга и 3–5 классах.
* Rate limit: 20 rps на инстанс (конфигурируется).
* Идемпотентность: опциональный in-memory кэш по `idempotency_key` (TTL 24h).

---

### 9. Ошибки и валидация

* `400`: неверный JSON/тип поля; отсутствуют обязательные поля `company`, `signals`, `icp`.
* `422`: некорректный `icp` (нет `candidate_labels` и/или не найден ни один центроид, и нет `keywords`).
* `500`: внутренняя ошибка инференса модели (отчёт с трассой в лог).

---

### 10. Конфигурация сервиса

* `config.yaml`:

  * `embedding_model: "bge-m3"`
  * `fit_threshold: 0.65`
  * `llm_gate: { enabled: false, gap_threshold: 0.05, max_chars: 8000, model: "local-7b" }`
  * `centroids_dir: "./centroids"` (ожидаем файлы `{icp_id}/{label}.npy`)
  * `metrics: enabled: true`
  * `logging: json: true, level: info`

---

### 11. Псевдокод

```python
def qualify(payload: QualifyIn) -> QualifyOut:
    icp = resolve_icp(payload.icp)                  # нормализуем include/size_band/candidates
    text = payload.embedding.vector or embed(payload.embedding.text)
    rule_score = compute_rule_score(payload, icp, payload.rules)

    scores = {}
    for label in icp.scoring.candidate_labels:
        c = centroids.get(icp.icp_id, label) or centroid_from_keywords(icp.include.keywords)
        sim01 = cosine01(text, c)
        scores[label] = icp.scoring.alpha * sim01 + icp.scoring.beta * rule_score

    label = max(scores, key=scores.get)
    top1, top2 = scores[label], sorted(scores.values(), reverse=True)[1] if len(scores) > 1 else 0.0
    confidence = float(top1)

    if cfg.llm_gate.enabled and (top1 - top2) < cfg.llm_gate.gap_threshold:
        decision, note = llm_review(compact_context(payload), label)
        if decision == "confirm": confidence = min(1.0, confidence + 0.03)
        elif decision == "reject": label, confidence = second_best_label(scores)

    return build_output(payload, label, confidence, rule_score, text)
```

---

### 13. Деплой

* **Dockerfile** (python:3.11-slim, install: fastapi, pydantic, sentence-transformers, onnxruntime, uvicorn).
* **ENV:** `EMBEDDING_MODEL`, `CENTROIDS_DIR`, `FIT_THRESHOLD`, `LLM_ENABLED`.

---

### 14. Важные нюансы MVP

* Центроиды для `icp.scoring.candidate_labels` держим в `centroids/{icp_id}/{label}.npy`.
  Если файла нет — используем **временный** центроид из `icp.include.keywords` (эмбеддинг строки `keywords.join(" ")`).
* `rules` в запросе **перекрывают** логику только на один вызов (ad-hoc). Общая логика — в `icp` (`include`, `keywords`, пороги).
* Всё работает без LLM. LLM — лишь «воротца» для спорных случаев.

---

### 15. Что сдаём по результату разработки

* Репозиторий: код сервиса, конфиг, README (запуск/примеры вызова).
* Pydantic-схемы, валидация, OpenAPI (авто от FastAPI).
* Набор тестов, фикстуры компаний.
* Папка `centroids/` с примерами (для `global.logistics_smb`: `supply_chain_platform`, `ai_analytics_for_logistics`, `not_a_fit`).
* Скрипт `tools/make_centroid.py` — строит `.npy` из списка ключевых слов или текстов эталонных компаний.

