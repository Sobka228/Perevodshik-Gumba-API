# Perevodshik-Gumba-API
Документация для API https://sobka-sus.ru/gumba/.
# Gumba Translate API

HTTP API переводчика между русским языком и языком Гумба. Работает на PHP, данные хранятся в `gumba_v2.json`.

- **Сайт:** https://sobka-sus.ru
- **Базовый URL:** `https://sobka-sus.ru/api.php`
- **Формат ответов:** JSON, UTF-8
- **CORS:** разрешён для любых источников (`Access-Control-Allow-Origin: *`)
- **Авторизация:** не требуется

## Содержание

- [Быстрый старт](#быстрый-старт)
- [Как устроен перевод](#как-устроен-перевод)
- [Методы](#методы)
  - [translate — перевод](#translate--перевод)
  - [words — список слов словаря](#words--список-слов-словаря)
  - [word — одно слово](#word--одно-слово)
  - [alphabet — алфавит](#alphabet--алфавит)
- [Ошибки](#ошибки)
- [Ограничения](#ограничения)
- [Примеры кода](#примеры-кода)

---

## Быстрый старт

```bash
curl "https://sobka-sus.ru/api.php?text=кот&dir=ru_gm"
```

```json
{
  "ok": true,
  "result": "IkiK+O+T",
  "info": "",
  "direction": "ru_gm"
}
```

Все ответы содержат поле `ok`: `true` при успехе, `false` при ошибке (тогда есть поле `error`).

---

## Как устроен перевод

1. Каждая русская буква заменяется кодом из таблицы алфавита (см. [alphabet](#alphabet--алфавит)). Например: `к` → `IkiK`, `о` → `O`, `т` → `T`.
2. Буквы внутри слова соединяются знаком `+`: `кот` → `IkiK+O+T`.
3. Слова разделяются пробелом. Знаки препинания, цифры и неизвестные символы остаются без изменений.
4. **Словарь имеет приоритет.** Если слово или фраза целиком есть в словаре, возвращается код из словаря, а не побуквенный перевод.
5. Регистр входного русского текста не учитывается (текст приводится к нижнему регистру).

### Обратный перевод (Гумба → русский)

- Сначала проверяется, нет ли целой фразы или слова в словаре (по точному совпадению кода).
- Иначе код разбирается по таблице алфавита: берётся самый длинный подходящий код буквы, затем пропускается один разделитель `+`.
- Поэтому код `+MƧ` (буква «ж») не путается с разделителем: `+MƧ+U+IkiK` → «жук».
- У некоторых букв общий код, при обратном переводе выбирается первая: `E` → «е» (а не «ё»/«э»), `I` → «и» (а не «й»).
- Регистр кода при разборе не учитывается.
- Неизвестные фрагменты возвращаются как есть.

---

## Методы

Метод выбирается параметром `action`. Если параметр не указан, используется `translate`.

Параметры можно передавать:
- в строке запроса (GET),
- в теле POST как JSON (`Content-Type: application/json`),
- в теле POST как обычная форма (`application/x-www-form-urlencoded`).

> ⚠️ В GET-запросе символ `+` нужно кодировать как `%2B`, иначе он превратится в пробел. Используйте `encodeURIComponent` / `urlencode` или передавайте данные через POST.

### translate — перевод

`GET` или `POST` · `?action=translate` (по умолчанию)

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `text` | string | да | Текст для перевода, до 5000 символов |
| `dir` | string | нет | Направление: `ru_gm` (русский → Гумба, по умолчанию) или `gm_ru` (Гумба → русский) |

**Пример (русский → Гумба)**

```bash
curl "https://sobka-sus.ru/api.php?text=%D0%BA%D0%BE%D1%82&dir=ru_gm"
```

```json
{
  "ok": true,
  "result": "IkiK+O+T",
  "info": "",
  "direction": "ru_gm"
}
```

**Пример (Гумба → русский, POST JSON)**

```bash
curl -X POST "https://sobka-sus.ru/api.php" \
  -H "Content-Type: application/json" \
  -d '{"text":"IkiK+O+T","dir":"gm_ru"}'
```

```json
{
  "ok": true,
  "result": "кот",
  "info": "",
  "direction": "gm_ru"
}
```

**Поля ответа**

| Поле | Описание |
|---|---|
| `ok` | `true` при успехе |
| `result` | Переведённый текст |
| `info` | Описание слова из словаря, если оно есть. Для нескольких слов строки вида `слово: описание`, по одной на строку. Для встроенных и автоматических слов пусто |
| `direction` | Использованное направление: `ru_gm` или `gm_ru` |

---

### words — список слов словаря

`GET` или `POST` · `?action=words`

Возвращает слова словаря с кодами и описаниями, отсортированные по слову.

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `q` | string | нет | Поиск по подстроке в русском слове (без учёта регистра) |

**Пример**

```bash
curl "https://sobka-sus.ru/api.php?action=words&q=жук"
```

```json
{
  "ok": true,
  "count": 1,
  "words": [
    {
      "word": "жук",
      "code": "+MƧ+U+IkiK",
      "info": "Жуки, или жесткокрылые (лат. Coleoptera) — это огромный отряд насекомых."
    }
  ]
}
```

| Поле | Описание |
|---|---|
| `count` | Количество найденных слов |
| `words[].word` | Русское слово или фраза |
| `words[].code` | Код на языке Гумба |
| `words[].info` | Описание (может быть пустым) |

> Ответ без `q` содержит весь словарь и может быть большим. Для поиска по словарю используйте `q`.

---

### word — одно слово

`GET` или `POST` · `?action=word`

| Параметр | Тип | Обязательный | Описание |
|---|---|---|---|
| `word` | string | да | Русское слово или фраза (точное совпадение, без учёта регистра) |

**Пример**

```bash
curl "https://sobka-sus.ru/api.php?action=word&word=жук"
```

```json
{
  "ok": true,
  "word": "жук",
  "code": "+MƧ+U+IkiK",
  "info": "Жуки, или жесткокрылые..."
}
```

Если слова нет в словаре, вернётся `404`:

```json
{ "ok": false, "error": "Слово не найдено." }
```

---

### alphabet — алфавит

`GET` или `POST` · `?action=alphabet`

Возвращает таблицу «русская буква → код Гумба».

```bash
curl "https://sobka-sus.ru/api.php?action=alphabet"
```

```json
{
  "ok": true,
  "alphabet": {
    "а": "A",
    "б": "B",
    "в": "V",
    "г": "G",
    "д": "HhD",
    "е": "E",
    "ё": "E",
    "ж": "+MƧ",
    "з": "Z",
    "и": "I",
    "й": "I",
    "к": "IkiK",
    "л": "SrL",
    "м": "M",
    "н": "N",
    "о": "O",
    "п": "P",
    "р": "Re",
    "с": "FiS",
    "т": "T",
    "у": "U",
    "ф": "Ff",
    "х": "H",
    "ц": "C",
    "ч": "Ch",
    "ш": "Sh",
    "щ": "Sch",
    "ы": "Y",
    "э": "E",
    "ю": "Yu",
    "я": "Ya"
  }
}
```

Букв `ъ` и `ь` в таблице нет: при переводе они остаются без изменений. Таблица может дополняться администратором, поэтому актуальные значения всегда можно получить этим методом.

---

---

## Ошибки

Тело ошибки всегда имеет вид `{"ok": false, "error": "..."}`.

| HTTP-код | Когда возникает |
|---|---|
| `400` | Не указан обязательный параметр, неверное значение `dir` или неизвестный `action` |
| `404` | Слово не найдено (`action=word`) |
| `413` | Текст длиннее 5000 символов (`translate`) |

---

## Ограничения

| Параметр | Значение |
|---|---|
| Максимальная длина `text` | 5000 символов |

---

## Примеры кода

### JavaScript (браузер / Node.js 18+)

```js
const API = 'https://sobka-sus.ru/api.php';

async function translate(text, dir = 'ru_gm') {
  const url = `${API}?text=${encodeURIComponent(text)}&dir=${dir}`;
  const res = await fetch(url);
  const data = await res.json();
  if (!data.ok) throw new Error(data.error);
  return data.result;
}

translate('кот').then(console.log);           // IkiK+O+T
translate('IkiK+O+T', 'gm_ru').then(console.log); // кот
```

### Python

```python
import requests

API = "https://sobka-sus.ru/api.php"

def translate(text, direction="ru_gm"):
    r = requests.post(API, json={"text": text, "dir": direction}, timeout=10)
    data = r.json()
    if not data["ok"]:
        raise RuntimeError(data["error"])
    return data["result"]

print(translate("кот"))                 # IkiK+O+T
print(translate("IkiK+O+T", "gm_ru"))   # кот
```

### Kotlin (Android, OkHttp)

```kotlin
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import okhttp3.Request
import okhttp3.RequestBody.Companion.toRequestBody
import org.json.JSONObject

val client = OkHttpClient()
const val API = "https://sobka-sus.ru/api.php"

fun translate(text: String, dir: String = "ru_gm"): String {
    val body = JSONObject()
        .put("text", text)
        .put("dir", dir)
        .toString()
        .toRequestBody("application/json".toMediaType())

    val request = Request.Builder().url(API).post(body).build()
    client.newCall(request).execute().use { response ->
        val json = JSONObject(response.body!!.string())
        if (!json.getBoolean("ok")) error(json.getString("error"))
        return json.getString("result")
    }
}
```

> Вызовы сети в Android нужно выполнять не в главном потоке (корутины, `Thread`, `ExecutorService`).

### PHP

```php
<?php
$api = 'https://sobka-sus.ru/api.php';
$url = $api . '?' . http_build_query(['text' => 'кот', 'dir' => 'ru_gm']);
$data = json_decode(file_get_contents($url), true);

echo $data['ok'] ? $data['result'] : $data['error'];   // IkiK+O+T
```
