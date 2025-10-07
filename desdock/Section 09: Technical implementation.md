# 9. Техническая реализация (по текущему проекту)

---

## 9.1. Архитектура в Unity

Проект построен как **односценная структура с управляющими менеджерами**.
Основные игровые состояния — утро → день → вечер — переключаются логикой внутри `GameManager` без перезагрузки сцены.

**Главная сцена содержит:**

```
Canvas
 ├── NoticeBoardUI
 ├── RoundSummary
 ├── WhisperUI
 ├── CharacterBar (портреты героев)
 └── StatusBar (репутация / золото / хаос)
EventSystem
WorldStateManager
GameManager
```

UI-элементы активируются и скрываются кодом (через `SetActive`, `CanvasGroup.alpha`, `interactable`).

---

## 9.2. Основные скрипты

| Класс                                         | Назначение                                                                                                      |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **GameManager**                               | Точка входа. Инициализирует мир, загружает данные из JSON, следит за стадией дня.                               |
| **WorldStateManager**                         | Хранит ресурсы (`reputation`, `gold`, `chaos`) и флаги мира. Обновляет их по результатам заданий.               |
| **NoticeBoardManager**                        | Создаёт карточки заявок, подтягивая данные из `notices.json` с фильтром по текущему дню (`rounds_config.json`). |
| **NoticeController**                          | Управляет отдельной карточкой: заголовок, описание, кнопка выбора героя, печать.                                |
| **PopulateExecutors(List<CharacterProfile>)** | Создаёт карточки исполнителей в рамках `executorsContainer`, передаёт ссылку в `SetCharacter(character, this)`. |
| **CharacterManager**                          | Подгружает данные из `characters.json`, отображает портреты внизу, обновляет их статус (хаос, реплики).         |
| **RoundSummary**                              | Показывает итоги дня (конверты, результаты, газета).                                                            |
| **ChaosWhisperUI**                            | Появление и исчезновение фраз Богини Хаоса (Shader Graph эффект «растекания чернил»).                           |
| **DialogController** (заготовка)              | Используется для магограмм — диалоговых веток с автопечатью и кнопками вариантов.                               |

---

## 9.3. JSON-файлы (данные)

### `characters.json`

* описание героев (имя, класс, цитата, черты, портрет).
* используется при инициализации нижней панели портретов.

### `notices.json`

* список всех заявок с ID, заголовком, типом, описанием и днём (`round_id`).
* подгружается в `NoticeBoardManager`.

### `rules.json`

* описание исходов для каждого задания (по героям).
* содержит поля `rep`, `gold`, `flag`, `chaos`.
* обрабатывается при завершении миссии в `RoundSummary`.

### `rounds_config.json`

* управляет тем, какие задания активны в текущем дне и какие герои доступны.
* структура: `day_id`, `active_notices[]`, `available_characters[]`.

### `dialogs.json`

* заготовка для будущих магограмм (поля `character_id`, `text`, `choices[]`, `next_id`).

---

## 9.4. Игровой цикл (реально реализованный)

```
Start()
 ├── GameManager.Init() 
 │     ├── Load WorldState
 │     ├── Load Characters
 │     ├── ShowMorningScene()
 │
 ├── NoticeBoardManager.UpdateAvailableNotices()
 │     ├── Spawn NoticeControllers
 │     ├── PopulateExecutors(characters)
 │
 ├── Player selects hero
 │     └── NoticeController.SetCharacter()
 │
 ├── After all notices processed:
 │     └── RoundSummary.ShowResults()
 │           ├── Update WorldState
 │           ├── Trigger ChaosWhisper (if flag)
 │           └── Show Newspaper
 │
 └── Proceed to next round (GameManager.NextDay)
```

---

## 9.5. UI и Canvas

| Компонент         | Реализация                                                                                        |
| ----------------- | ------------------------------------------------------------------------------------------------- |
| **NoticeBoardUI** | Grid Layout с пергаментными карточками. Создаётся и заполняется динамически.                      |
| **RoundSummary**  | Рандомно разбросанные конверты на Canvas. Клик → анимация вскрытия (`Animator` или `scale/fade`). |
| **WhisperUI**     | Текстовый элемент TMP, прозрачный фон, shader эффект.                                             |
| **CharacterBar**  | Портреты в нижней панели, кнопка → `CharacterProfileUI`.                                          |
| **StatusBar**     | Три иконки (репутация, золото, хаос). Обновляются через `WorldStateManager.UpdateUI()`.           |

---

## 9.6. Работа с флагами и ресурсами

```csharp
public void ApplyOutcome(NoticeData notice, CharacterProfile hero)
{
    var rule = RulesDB.GetRule(notice.id, hero.id);
    reputation += rule.rep;
    gold += rule.gold;
    if (rule.chaos > 0) hero.AddChaos(rule.chaos);
    if (!string.IsNullOrEmpty(rule.flag)) flags.Add(rule.flag);
    UpdateUI();
}
```

* Репутация и золото — числа (±1 по правилам).
* Флаги — строки в списке `flags`.
* Хаос накапливается на герое (в его профиле).

---

## 9.7. Состояния героев

Каждый герой имеет:

```csharp
int chaosLevel;    // 0–3
Sprite portrait;   // обычный или хаос-версия
string[] traits;
```

Изменение хаоса вызвает `CharacterManager.UpdatePortrait()` и подмену спрайта.

---

## 9.8. Шёпот Богини (ChaosWhisperUI)

Вызывается из `WorldStateManager` или `RoundSummary`:

```csharp
ChaosWhisperUI.Instance.Whisper("Ты чувствуешь, как закон дрожит...");
```

* Shader Graph эффект растекания чернил.
* Время появления → 2 с. Исчезновение → fade 2 с.
* Триггеры: хаос-результат, откат дня, события флагов.

---

## 9.9. Сцена итогов (RoundSummary)

* Карточки заявок преобразуются в конверты.
* При клике открывается панель с текстом результата.
* После всех конвертов → кнопка “Следующий день”.
* В нижней части отображается газета (если есть публикация).

---

## 9.10. Минимальный набор для релиза MVP

* Все менеджеры (`GameManager`, `WorldStateManager`, `NoticeBoardManager`, `RoundSummary`).
* Загрузка из JSON.
* 3 героя (Лада, Тихон, Шлюмп) с портретами и хаос-версиями.
* 4 дня контента (заявки, результаты, газеты).
* Флаги и ресурсы работают в реальном времени.
* Шёпот Богини и анимации конвертов.
* Стабильная петля утро → день → вечер.
