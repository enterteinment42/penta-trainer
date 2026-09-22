# penta-trainer — справочник

Устройство приложения: структура, ключевые структуры данных, аудио-движок, джем-луп,
состояние и хранение, деплой. Вынесено из `CLAUDE.md` 22.09.2026 — в автозагрузку
не входит, читается по необходимости.

---

## Структура

**Один самодостаточный файл** — `index.html` (~3070 строк): HTML + CSS + весь JS инлайн, без сборки, без зависимостей, без внешних библиотек. PWA-мета и inline-манифест в `<head>`.

```
index.html
├── <head>          PWA-мета, inline MANIFEST, inline CSS
├── <body>          две вкладки: 🎸 Пентатоника | 🎵 Аккорды (+блок «6. Минусовка» внизу пентатоники)
└── <script>        весь JS (одна область видимости, всё глобальное)
    ├── CONSTANTS        строка ~543 — CHROMATIC, OPEN_MIDI, боксы
    ├── SCALES           строка ~590 — 7 ладов (мажор/минор + характерная нота)
    ├── AUDIO            строка ~660 — makeVoice (acoustic KS / distortion KS+2 каскада+кабинет cabIR ~718),
    │                                  getDistBus ~742, kick/snare/bassNote/hat/tom, humT/humV
    ├── STATE            строка ~1050 — LS_KEY, saveState/loadState
    ├── RENDER           строка ~1100 — гриф, селектор ладов, инфо
    ├── LICKS            строка ~1477 — 6 ликов с подсветкой и звуком
    ├── RHYTHMS          строка ~1647 — 42 паттерна по 5 жанрам
    ├── JAM ENGINE       строка ~1753 — BAND, JAM_INTRO_BARS, jamChordIdx, jamBassLine, jamDrums, scheduleJamBand
    ├── PROGRESSION PLAYER строка ~1907 — scheduleBar, джем-луп, метроном, подсветка тонов
    ├── PROGRESSIONS     строка ~2066 — 29 минорных + 20 мажорных
    ├── МОИ ДЖЕМЫ        строка ~2437 — myJams (localStorage), конструктор, buildProgCard (реюз карточки)
    ├── МИНУСОВКА        строка ~2580 — bt-плеер: A-B луп, IndexedDB ('penta-trainer'/'tracks'), renderBackingTrack
    └── CHORD EXPLORER   строка ~2876 — CHORD_TYPES (16), VOICINGS (120), генератор
```

---

## Ключевые структуры данных

| Структура | Строка | Содержание |
|-----------|--------|-----------|
| `CHROMATIC` / `OPEN_MIDI` | ~543 | 12 нот хроматики; MIDI открытых струн `[40,45,50,55,59,64]` (0=lowE..5=highE) |
| `AM_BOXES` / `AMAJ_BOXES` | ~556/571 | 5 боксов пентатоники (минор/мажор), ключ midiIdx → `[лад1, лад2]` |
| `SCALES` | ~590 | 7 ладов: `base` (мин/маж пентатоника внутри), `char` (интервал характерной ноты), `charLabel`, `tip`, `genre` |
| `LICKS` | ~1477 | 6 ликов, `events:[{s,f,t,d,bend?,slide?}\|{x,t}]` |
| `RHYTHMS` | ~1647 | 42 боя (blues/rock/pop/funk/reggae), `steps`+`grid`+`swing` |
| `BAND` | ~1753 | простая ритм-секция по жанрам (🥁 без 🎛): kick/snare/bass по долям |
| `PROGRESSIONS_MINOR` / `_MAJOR` | ~2066 | 29+20 прогрессий, `degrees:[{d,q}]` (d=сдвиг, q=качество) |
| `myJams` | ~2437 | свои джемы `{name, genre, chords:[{d,q}]}`, d=абсолютный pitch class от C, играются с ri=0 (НЕ транспонируются тональностью) |
| `bt` | ~2580 | состояние минусовки: buf/offset/loopA/loopB/vol; файл в IndexedDB, настройки в LS `penta-trainer-backing` |
| `CHORD_TYPES` / `VOICINGS` | ~2876/2900 | 16 типов аккордов; 120 аппликатур «Нота-тип» (−1=мут, 0=открытая) |

**Проверенная корректность (ревизия):** боксы пентатоники (все 12 транспозиций, полнота, стыковка), все 120 VOICINGS (правильные тоны, бас=тоника), генератор аппликатур (add9/madd9/9/dim/aug/dim7), 7 ладов (ротации мажорной гаммы), 40 ритмов (steps==grid). Единственный «флаг» — открытый C7 (x32310) без квинты, но это стандартная гитарная аппликатура.

---

## Аудио-движок

- **`getCtx()`** — ленивый AudioContext, resume при suspended.
- **Master-цепь** — мягкий компрессор-лимитер (ловит пики полифонии) → `masterOutNode` (громкость, слайдер 0–1.5). ⚠️ баг: громкость применяется ПОСЛЕ лимитера (см. баги выше).
- **`makeVoice(midi, sustain, opts)`** (строка 712) — один голос. Два тембра:
  - **acoustic** — Karplus-Strong (`ksBuffer`, строка 691): физически честный щипок, высокие затухают быстрее. ⚠️ генерируется синхронно (см. баги).
  - **distortion** — пила + суб-октава через общий перегруз-бус (`getDistBus`, строка 662): настоящая интермодуляция струн, «жужжащий» сустейн + щелчок медиатора.
- **`chick()`** — глушёный «чик» для фанка/регги. **`metroClick()`** — клик метронома.
- **Штрих боя** (`playStrumStroke`) — down = все ноты, up = верхние снизу-вверх (тише), X = чик. ⚠️ down/up одинаковы по тембру, строб фиксирован 12 мс (см. улучшения звука).

---

## Джем-луп (Progression Player)

- **`startPlayer(prog, ri, cardEl, rhythmIdx)`** (1430) → `scheduleBar` (1341) планирует такт по аудио-часам (sample-accurate lookahead, без дрожания setTimeout).
- **Подсветка тонов аккорда** (`highlightChordTones`) — при смене такта тоны текущего аккорда загораются среди нот пентатоники/лада на грифе. **Это главная обучающая фича.**
- **Режим «только ритм»** (`rhythmOnlyOn`) — луп на одной тонике, отработка боя. ⚠️ не сохраняется в LS.
- **Метроном** (`metroOn`), **тап-темпо**, **BPM-слайдер** — в раскрывающейся панели ритма у каждой прогрессии. Настройки джема по прогрессии в `jamPrefs` (LS).
- **Мёртвый код:** отсчёт «1-2-3-4» (countEvents) — вся логика есть, но не заполняется (см. баги).

---

## Состояние и хранение

- **`localStorage['penta-trainer-state']`** (`LS_KEY`, строка 844) — `saveState`/`loadState`.
- Хранится: `selKey, selBox, selMode, selScale, selGenre, timbre, noteLabels, jamPrefs, activeTab, metroOn, masterVol`.
- ⚠️ `loadState` проверяет совместимость лада с режимом (`SCALES[x].base === selMode`), несовместимый отбрасывает. `setMode` сбрасывает лад при смене минор/мажор.
- ⚠️ **В артефактах Claude.ai localStorage НЕ работает** — но это живой сайт на Netlify, там работает нормально.

---

## Деплой

**Netlify ↔ GitHub, авто-деплой** (настроено 2026-07-14):

1. Правим `index.html` локально
2. Коммитим и пушим в `main` на `github.com/enterteinment42/penta-trainer`
3. Netlify ловит push → пересобирает (билда нет, чистый HTML) → обновляет `penta-trainer.netlify.app` (1-2 мин)

- **Репозиторий:** `github.com/enterteinment42/penta-trainer` (Public), ветка `main`, один файл `index.html` в корне.
- **Netlify:** проект `penta-trainer`, команда Poigraem, Build command пусто, Publish directory пусто (корень). Раньше деплоился вручную через Netlify Drop — теперь связан с Git.
- **Без токенов через браузер:** если нет доступа к git-пушу — GitHub принимает файл drag-and-drop (Add file → Upload files → Commit), Netlify подхватит.

⚠️ **Отличие от магазина Poigraem:** здесь НЕТ backend, VPS, Supabase, БД, секретов, `.env`, параллельных сессий на VPS. Это статический фронт целиком. Ничего из инфраструктуры Poigraem сюда не переносить.

---

