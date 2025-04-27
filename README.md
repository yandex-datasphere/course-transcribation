# Курсовой проект: СЕРВИС АВТОМАТИЧЕСКОГО КОНСПЕКТИРОВАНИЯ ВИДЕОКУРСОВ

В этом репозитории находятся три Jupyter Notebook, реализующие последовательный конвейер обработки видеолекций:

1. **1-preprocess-v2.ipynb** — транскрипция аудио, извлечение ключевых кадров и расчёт метрик.
2. **2-param-selection-v2.ipynb** — (опционально) визуализация и подбор параметров для отбора ключевых кадров.
3. **3-processing.ipynb** — финальная генерация Markdown‑конспекта и встраивание изображений.

Файлы ноутбуков должны располагаться в корне репозитория рядом с `config.json` (создаётся автоматически при первом запуске).

---

## Структура репозитория

```text
├── 1-preprocess-v2.ipynb      # Этап 1: транскрипция и извлечение кадров
├── 2-param-selection-v2.ipynb # Этап 2: подбор параметров (опционально)
├── 3-processing.ipynb         # Этап 3: генерация конспекта
├── keys.txt                   # API ключи для 3-processing.ipynb 
└── config.json                # Файл конфигурации с путями и параметрами
```

---

## Требования

- **Python ≥ 3.8**
- **Jupyter Notebook** или **JupyterLab**
- **GPU** с поддержкой CUDA (рекомендуется для ускорения EasyOCR и трансформеров)

Установите зависимости одной командой:

```bash
pip install \
  openai easyocr sentence-transformers scipy scikit-learn \
  numpy matplotlib pillow opencv-python requests
```


---

## Конфигурация (`config.json`)

В файле `config.json` задаются:
- **`base_directory`** — корневая папка проекта, относительно которой строятся все пути. В ней необходимо создать папку /input и пернести в нее видео, которые необходимо обработать.
- **`number_video_images`** — количество ключевых кадров для сохранения (top‑N).
- **`weights`** — коэффициенты для расчёта composite_score:
  - `WEIGHT_TEXT`, `WEIGHT_DIFF`, `WEIGHT_SIMILARITY`, `WEIGHT_RECENCY`
- **`coordinates_pct`** — нормированные (0.0–1.0) координаты области кадра для OCR:
  - `x1`, `y1`, `x2`, `y2`

Пример содержимого:

```json
{
  "base_directory": "/transcribation-project/test",
  "number_video_images": 5,
  "weights": {
    "WEIGHT_TEXT":       0.1,
    "WEIGHT_DIFF":       0.6,
    "WEIGHT_SIMILARITY": 0.2,
    "WEIGHT_RECENCY":    0.05
  },
  "coordinates_pct": {
    "x1": 0.0,
    "y1": 0.0,
    "x2": 1.0,
    "y2": 1.0
  }
}
```

> Все ноутбуки автоматически загружают `config.json` и используют указанные в нём параметры и `base_directory`.

Папки внутри `base_directory`, создаваемые и используемые по умолчанию:

- `input/`                — исходные видео `.mp4`.
- `work/whisper_json/`     — JSON-транскрипты Whisper.
- `work/whisper_text/`     — TXT-транскрипты.
- `work/all_images/`       — все извлечённые кадры.
- `work/pretty_text/`      — промежуточный текст с mermaid-схемами.
- `output/images/`         — отобранные ключевые кадры (top‑N).
- `output/`                — финальные файлы (Markdown‑конспект и пр.).

---

## Доступы к API

Вы должны иметь доступ к YandexGpt с помощью Yandex Console. 
Для этого необходимо знать folder_id для формирования url модели и сгенерированный IAM
token с помощью команды в PowerShell (для Windows)
```
yc iam create-token
```
Также необходим доступ к DeepSeek, полученный с помощью nebius.ai, для этого нужно знать только api ключ, полученный от 
сервиса nebius.

Необходимо создать файл keys.txt в корне проекта, рядом с ноутбуками. Файл необходимо заполнить нужной информацией в формате:
```
folder_id  = 'your_folder_id'
api_key = 'your_api_nebius_key'
I_AM_TOKEN = 'your_yandex_cloud_iam_token'
```

---

## Описание этапов

### 1. Preprocess (1-preprocess-v2.ipynb)

- Загружает параметры и пути из `config.json` через `read_config_file()`.
- Транскрибирует видео (Whisper API), сохраняет JSON и TXT-транскрипты.
- Извлекает кадры и сохраняет в папку `work/all_images/` .
- Считает метрики для каждого кадра:
  - `text_score` — насыщенность текстом (OCR + семантическая оценка).
  - `diff_score` — визуальная разница между кадрами.
  - `semantic_similarity` — близость эмбеддингов.
  - `recency` — предпочтение более поздних кадров.
- Отбирает топ‑N кадров по composite_score и сохраняет в `output/images`.
- При первом запуске создаёт или обновляет `config.json` с сохранёнными путями и параметрами.

### 2. Подбор параметров (2-param-selection-v2.ipynb) — опционально

- Читает текущие настройки из `config.json` через `load_config()`.
- Показывает графики распределения метрик (`show_metrics_plot()`) и текущие топ‑3 кадров (`show_top_frames()`).
- Позволяет интерактивно изменять веса и координаты, автоматически сохраняя изменения в `config.json` через `update_parameters()`.
- Пересчитывает метрики и обновляет папку `output/images`.

> Используйте этот этап для тонкой настройки отбора кадров. Если настройки подходят, пропустите этот шаг.

### 3. Генерация конспекта (3-processing.ipynb)

- Загружает все необходимые пути и параметры из `config.json`.
- Формирует запросы к GPT (`gpt_request()`) и DeepSeek (`deepseek_request()`), собирая исходный текст.
- Очищает и форматирует ответ (`clear_simple_text()`, `extract_response_text()`).
- Вставляет маркеры изображений `⟪frame.jpg⟫` в текст Markdown и конвертирует их в ссылки с помощью `insert_frame_images()`.
- Сохраняет итоговый файл `final_conspect.md` в папку `output_markdown_directory`.

---

## Запуск всего конвейера

1. Поместите все три ноутбука и `config.json` в корень проекта.
2. Скопируйте видео в папку input, созданную в папке, указанной в `config.json`
3. Откройте `1-preprocess-v2.ipynb` и запустите все ячейки.
4. (Опционально) Запустите `2-param-selection-v2.ipynb` для настройки.
5. Запустите `3-processing.ipynb` для получения готового конспекта.

Готовый Markdown‑файл и ключевые кадры будут в папках, указанных в `config.json`.


