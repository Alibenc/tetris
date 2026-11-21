# Визуальная новелла на Pygame — пишем движок с нуля

**Цель:** написать минимальный расширяемый движок визуальной новеллы, не привязанный к сюжету. **Предварительные знания:** основы Python, классы, функции, словари, циклы. **Версия:** Python 3.10+.

## 0. Готовим проект

Создайте папку `visual_novel_starter` и внутри следующую структуру (пустые папки можно создавать постепенно):

```text
visual_novel_starter/
├── run.py
├── requirements.txt
├── novel/
│   ├── __init__.py
│   ├── settings.py
│   ├── story.py
│   ├── resources.py
│   ├── ui.py
│   ├── save.py
│   └── app.py
├── story/
│   └── demo.json
├── assets/
│   ├── backgrounds/
│   └── characters/
└── saves/
```

В `requirements.txt` запишите:

```text
pygame>=2.5,<3
```

В терминале, открытом в корне проекта, выполните:

```bash
python -m pip install -r requirements.txt
```

`novel/__init__.py` пока оставьте **пустым**. Он обозначает пакет Python, чтобы работали импорты `from novel.app import Game` и относительные импорты внутри пакета.

**Как мы разделяем ответственность:**

| Файл | Что хранит | Чего в нём быть не должно |
| --- | --- | --- |
| `settings.py` | Константы, пути, цвета | Игрового цикла |
| `story/demo.json` | Сценарные данные | Python-кода |
| `story.py` | Структуры и проверку сценария | Отрисовки |
| `resources.py` | Загрузку PNG и кэш | Логики диалогов |
| `ui.py` | Панель, кнопки, шрифты | Чтения JSON |
| `save.py` | Запись и чтение сохранения | Pygame |
| `app.py` | Состояние, обработку событий, игровой цикл | Реплик конкретной истории |
| `run.py` | Запуск игры | Игровой логики |

**Порядок работы:** сначала создаём все независимые модули, затем `app.py`, затем запускаем приложение. До появления `app.py` отдельные файлы сами по себе полноценную игру не запустят — это нормально.

---

## 1. Пишем `novel/settings.py` — Настройки и пути

С этого файла начинаем: остальным модулям нужны размеры окна, пути к файлам и цвета. `Path(__file__).resolve()` позволяет находить ресурсы относительно самого проекта, а не текущей папки терминала.

**Что важно в коде:** `ROOT` — корень проекта; оператор `/` у `Path` соединяет части пути; `TEXT_SPEED` задаёт символы в секунду. Цвета записаны тройками RGB.

**Проверка:** измените `TITLE` и `TEXT_SPEED` после окончания сборки. Окно должно получить новый заголовок, а буквы — другую скорость.

**Полное содержимое файла:**

```python
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent
ASSETS = ROOT / "assets"
STORY_FILE = ROOT / "story" / "demo.json"
SAVE_FILE = ROOT / "saves" / "save.json"

WIDTH, HEIGHT = 1000, 600
FPS = 60
TITLE = "Visual Novel Starter"
TEXT_SPEED = 45  # символов в секунду

BG_COLOR = (28, 32, 50)
PANEL_COLOR = (15, 20, 34)
TEXT_COLOR = (246, 247, 251)
MUTED_COLOR = (180, 188, 202)
ACCENT_COLOR = (102, 174, 255)
```

---

## 2. Пишем `story/demo.json` — Сценарий: данные без игрового кода

JSON — это обычные данные, а не Python-код. `start` указывает ID стартовой реплики, `nodes` хранит все реплики. Узел с `next` автоматически ведёт к следующему. Узел с `choices` предлагает варианты ответа. Узел без обоих полей — концовка.

**Что важно в коде:** название узла (`intro`) — его ID; `next` должен точно совпадать с ID другого узла; изображения указываются без `.png`. Поле `character` необязательно, и если его нет, персонаж не отображается.

**Проверка:** проследите цепочку `intro → question → ending_a` и `intro → question → ending_b`. До запуска игры вы уже должны уметь объяснить, куда ведёт каждая кнопка.

**Полное содержимое файла:**

```json
{
  "start": "intro",
  "nodes": {
    "intro": {
      "speaker": "Система",
      "text": "Это тестовый сценарий движка. Здесь нет привязки к сюжету: вы сможете заменить все реплики и локации.",
      "background": "default",
      "character": "placeholder",
      "next": "question"
    },
    "question": {
      "speaker": "Система",
      "text": "Выберите направление. Каждая кнопка ведёт к другому узлу истории.",
      "background": "default",
      "character": "placeholder",
      "choices": [
        {
          "text": "Вариант А",
          "next": "ending_a"
        },
        {
          "text": "Вариант Б",
          "next": "ending_b"
        }
      ]
    },
    "ending_a": {
      "speaker": "Система",
      "text": "Вы выбрали вариант А. Это тестовая концовка. Нажмите R для перезапуска.",
      "background": "default"
    },
    "ending_b": {
      "speaker": "Система",
      "text": "Вы выбрали вариант Б. Это другая тестовая концовка. Нажмите R для перезапуска.",
      "background": "default"
    }
  }
}
```

---

## 3. Пишем `novel/story.py` — Модели и загрузчик истории

Здесь превращаем сырой JSON в структурированные Python-объекты. `Choice` описывает вариант ответа, `Node` — одну реплику. `@dataclass(frozen=True)` создаёт конструкторы и защищает поля от случайных изменений.

**Разбор по шагам:** `json.loads(...)` читает JSON; `for node_id, raw in data["nodes"].items()` перебирает все узлы; `raw.get(...)` возвращает запасное значение, если поля нет; `tuple(...)` собирает варианты ответа; затем идёт проверка корректности связей. Если отсутствует стартовый узел или `next` указывает в пустоту, сразу выдаём понятную ошибку, а не падаем посреди диалога.

**Проверка:** намеренно измените `"next": "question"` на `"next": "missing"`, запустите программу после сборки и прочитайте текст исключения. Потом верните всё обратно.

**Полное содержимое файла:**

```python
import json
from dataclasses import dataclass
from pathlib import Path


@dataclass(frozen=True)
class Choice:
    text: str
    target: str


@dataclass(frozen=True)
class Node:
    id: str
    speaker: str
    text: str
    background: str
    character: str | None
    next_id: str | None
    choices: tuple[Choice, ...]


class Story:
    def __init__(self, path: Path):
        data = json.loads(path.read_text(encoding="utf-8"))
        self.start = data["start"]
        self.nodes: dict[str, Node] = {}

        for node_id, raw in data["nodes"].items():
            choices = tuple(Choice(c["text"], c["next"]) for c in raw.get("choices", []))
            node = Node(
                id=node_id,
                speaker=raw.get("speaker", ""),
                text=raw["text"],
                background=raw.get("background", "default"),
                character=raw.get("character"),
                next_id=raw.get("next"),
                choices=choices,
            )
            if node.next_id is not None and choices:
                raise ValueError(f"У узла {node_id} одновременно next и choices")
            self.nodes[node_id] = node

        if self.start not in self.nodes:
            raise ValueError(f"Не найден стартовый узел: {self.start}")
        for node in self.nodes.values():
            targets = ([node.next_id] if node.next_id else []) + [c.target for c in node.choices]
            for target in targets:
                if target not in self.nodes:
                    raise ValueError(f"Узел {node.id} ведёт на несуществующий {target}")

    def get(self, node_id: str) -> Node:
        return self.nodes[node_id]
```

---

## 4. Пишем `novel/resources.py` — Загрузка и кэширование изображений

Этот модуль ничего не знает про диалоги: ему передают имя изображения, он возвращает `pygame.Surface`. Кэш нужен, чтобы не читать файл с диска каждый кадр.

**Разбор:** словари `backgrounds` и `characters` — кэши; `path.exists()` проверяет наличие файла; `.convert()` подходит фонам, `.convert_alpha()` сохраняет прозрачность персонажа; `smoothscale()` меняет размер. Когда PNG отсутствует, программа рисует заглушку — благодаря этому класс можно тестировать до подготовки графики.

**Проверка:** без картинок должны показываться временный фон и условный персонаж. Затем положите PNG в `assets/backgrounds/room.png` и укажите `"background": "room"`.

**Полное содержимое файла:**

```python
import pygame
from .settings import ASSETS, WIDTH, HEIGHT


class Resources:
    """Кэш изображений: файлов может не быть — тогда используем заглушки."""

    def __init__(self):
        self.backgrounds: dict[str, pygame.Surface] = {}
        self.characters: dict[str, pygame.Surface | None] = {}

    def background(self, name: str) -> pygame.Surface:
        if name not in self.backgrounds:
            path = ASSETS / "backgrounds" / f"{name}.png"
            if path.exists():
                image = pygame.image.load(str(path)).convert()
                image = pygame.transform.smoothscale(image, (WIDTH, HEIGHT))
            else:
                image = pygame.Surface((WIDTH, HEIGHT))
                image.fill((41, 53, 79))
                pygame.draw.circle(image, (69, 87, 118), (WIDTH - 130, 115), 85)
                pygame.draw.rect(image, (33, 42, 65), (0, HEIGHT - 180, WIDTH, 180))
            self.backgrounds[name] = image
        return self.backgrounds[name]

    def character(self, name: str | None) -> pygame.Surface | None:
        if not name:
            return None
        if name not in self.characters:
            path = ASSETS / "characters" / f"{name}.png"
            if path.exists():
                img = pygame.image.load(str(path)).convert_alpha()
                max_height = 470
                factor = min(1, max_height / img.get_height())
                img = pygame.transform.smoothscale(
                    img, (max(1, int(img.get_width() * factor)), max(1, int(img.get_height() * factor)))
                )
            else:
                img = pygame.Surface((220, 390), pygame.SRCALPHA)
                pygame.draw.circle(img, (119, 171, 225), (110, 85), 65)
                pygame.draw.rect(img, (75, 115, 182), (40, 155, 140, 235), border_radius=32)
            self.characters[name] = img
        return self.characters[name]
```

---

## 5. Пишем `novel/ui.py` — Отрисовка окна диалога и кнопок

Здесь сосредоточена **только отрисовка интерфейса**. Игровой цикл не должен вручную создавать шрифты и рисовать каждую кнопку. `Interface` умеет рисовать диалог, варианты выбора и подсказки.

**Разбор:** `wrap_text` собирает строки по ширине пикселей через `font.size()`, а не по количеству букв. `draw_dialogue` рисует панель и имя говорящего. `choice_buttons` возвращает пары `(прямоугольник, вариант)` — эти же прямоугольники потом используются для проверки мыши в `app.py`. `draw_choices` подсвечивает кнопку под курсором.

**Ограничение учебной версии:** выводятся не больше трёх строк диалога; длинные тексты нужно сокращать или позже добавить прокрутку. Количество кнопок также следует держать небольшим.

**Проверка:** после сборки измените цвет панели и размеры кнопок, а потом убедитесь, что клики по ним работают.

**Полное содержимое файла:**

```python
import pygame
from .settings import WIDTH, HEIGHT, PANEL_COLOR, TEXT_COLOR, MUTED_COLOR, ACCENT_COLOR


def font(size: int) -> pygame.font.Font:
    # Системный шрифт с кириллицей. На Windows обычно доступен Arial.
    return pygame.font.SysFont("arial", size)


def wrap_text(text: str, text_font: pygame.font.Font, max_width: int) -> list[str]:
    lines: list[str] = []
    for paragraph in text.split("\n"):
        if not paragraph:
            lines.append("")
            continue
        current = ""
        for word in paragraph.split():
            test = f"{current} {word}".strip()
            if text_font.size(test)[0] <= max_width:
                current = test
            else:
                if current:
                    lines.append(current)
                current = word
        lines.append(current)
    return lines


class Interface:
    def __init__(self):
        self.body = font(26)
        self.name = font(28)
        self.small = font(20)
        self.choice = font(23)

    def draw_dialogue(self, surface: pygame.Surface, speaker: str, text: str, ready: bool, ending: bool):
        panel = pygame.Rect(35, HEIGHT - 205, WIDTH - 70, 170)
        pygame.draw.rect(surface, PANEL_COLOR, panel, border_radius=15)
        pygame.draw.rect(surface, ACCENT_COLOR, panel, 2, border_radius=15)

        if speaker:
            surface.blit(self.name.render(speaker, True, ACCENT_COLOR), (65, HEIGHT - 188))
        for index, line in enumerate(wrap_text(text, self.body, WIDTH - 140)[:3]):
            surface.blit(self.body.render(line, True, TEXT_COLOR), (65, HEIGHT - 141 + index * 34))

        hint = "Конец · R — заново" if ending else "Пробел / клик — далее"
        if ready:
            label = self.small.render(hint, True, MUTED_COLOR)
            surface.blit(label, (WIDTH - label.get_width() - 62, HEIGHT - 62))

    def choice_buttons(self, choices) -> list[tuple[pygame.Rect, object]]:
        count = len(choices)
        first_y = max(55, HEIGHT - 215 - count * 58)
        return [(pygame.Rect(140, first_y + i * 58, WIDTH - 280, 48), choice)
                for i, choice in enumerate(choices)]

    def draw_choices(self, surface: pygame.Surface, choices, mouse_pos):
        for rect, choice in self.choice_buttons(choices):
            hover = rect.collidepoint(mouse_pos)
            pygame.draw.rect(surface, (65, 94, 138) if hover else (32, 49, 78), rect, border_radius=10)
            pygame.draw.rect(surface, ACCENT_COLOR, rect, 2, border_radius=10)
            label = self.choice.render(choice.text, True, TEXT_COLOR)
            surface.blit(label, (rect.centerx - label.get_width() // 2,
                                 rect.centery - label.get_height() // 2))

    def draw_hint(self, surface: pygame.Surface):
        label = self.small.render("S — сохранить   L — загрузить   R — сначала   Esc — выход", True, TEXT_COLOR)
        surface.blit(label, (20, 15))
```

---

## 6. Пишем `novel/save.py` — Сохранения

Движок сохраняет **ID текущей реплики**, а не весь объект игры. Поэтому формат прост: `{"node_id": "question"}`.

**Разбор:** `mkdir(..., exist_ok=True)` создаёт папку, `json.dumps` превращает словарь в строку JSON, `write_text` записывает файл. При загрузке сначала проверяем существование файла; ошибки чтения и неправильный JSON безопасно превращаются в `None`.

**Проверка:** нажмите `S`, откройте `saves/save.json`; затем смените реплику и нажмите `L`. Прогресс должен вернуться к сохранённому узлу.

**Полное содержимое файла:**

```python
import json
from pathlib import Path


def save_progress(path: Path, node_id: str) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps({"node_id": node_id}, ensure_ascii=False, indent=2), encoding="utf-8")


def load_progress(path: Path) -> str | None:
    if not path.exists():
        return None
    try:
        data = json.loads(path.read_text(encoding="utf-8"))
        node_id = data.get("node_id")
        return node_id if isinstance(node_id, str) else None
    except (OSError, json.JSONDecodeError):
        return None
```

---

## 7. Пишем `novel/app.py` — Связываем все части в работающую игру

Это центральный модуль, но он **не содержит сценарий и не рисует интерфейс вручную**. Объект `Game` соединяет `Story`, `Resources` и `Interface`.

**Разбор методов:**

- `__init__` — инициализирует Pygame, окно, часы, историю и текущее состояние.
- `node` — свойство, которое возвращает текущий узел истории по `node_id`.
- `go_to` — меняет узел и сбрасывает счётчик показанных символов.
- `advance` — первый клик мгновенно допечатывает фразу, следующий переводит на `next`. Если доступны варианты ответа, требуется выбрать кнопку.
- `handle_event` — получает события клавиатуры и мыши; здесь не должно быть отрисовки.
- `draw` — собирает кадр из фона, персонажа, диалога, выбора и подсказки.
- `run` — главный цикл. `clock.tick(FPS) / 1000` превращает миллисекунды в секунды (`dt`); `TEXT_SPEED * dt` обеспечивает стабильную скорость букв.

**Важно:** `draw()` вызывается каждый кадр, но ресурсы загружаются через кэш, а не с диска каждый раз. При клике по кнопке проверяется тот же прямоугольник, который UI использует при рисовании.

**Проверка:** пробел должен допечатывать текст, следующий пробел — переводить на следующую реплику; в сцене с выбором щёлкните по кнопке. `R` — перезапуск, `S` / `L` — сохранение / загрузка, `Esc` — выход.

**Полное содержимое файла:**

```python
import pygame
from .settings import WIDTH, HEIGHT, FPS, TITLE, STORY_FILE, SAVE_FILE, TEXT_SPEED
from .story import Story
from .resources import Resources
from .ui import Interface
from .save import save_progress, load_progress


class Game:
    def __init__(self):
        pygame.init()
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT))
        pygame.display.set_caption(TITLE)
        self.clock = pygame.time.Clock()
        self.story = Story(STORY_FILE)
        self.resources = Resources()
        self.ui = Interface()
        self.running = True
        self.node_id = self.story.start
        self.visible_chars = 0.0

    @property
    def node(self):
        return self.story.get(self.node_id)

    def go_to(self, node_id: str):
        self.node_id = node_id
        self.visible_chars = 0.0

    def advance(self):
        if self.visible_chars < len(self.node.text):
            self.visible_chars = float(len(self.node.text))
        elif not self.node.choices and self.node.next_id:
            self.go_to(self.node.next_id)

    def handle_event(self, event):
        if event.type == pygame.QUIT:
            self.running = False
        elif event.type == pygame.KEYDOWN:
            if event.key == pygame.K_ESCAPE:
                self.running = False
            elif event.key == pygame.K_r:
                self.go_to(self.story.start)
            elif event.key == pygame.K_s:
                save_progress(SAVE_FILE, self.node_id)
            elif event.key == pygame.K_l:
                saved = load_progress(SAVE_FILE)
                if saved in self.story.nodes:
                    self.go_to(saved)
            elif event.key in (pygame.K_SPACE, pygame.K_RETURN):
                self.advance()
        elif event.type == pygame.MOUSEBUTTONDOWN and event.button == 1:
            if self.visible_chars >= len(self.node.text) and self.node.choices:
                for rect, choice in self.ui.choice_buttons(self.node.choices):
                    if rect.collidepoint(event.pos):
                        self.go_to(choice.target)
                        return
            else:
                self.advance()

    def draw(self):
        self.screen.blit(self.resources.background(self.node.background), (0, 0))
        character = self.resources.character(self.node.character)
        if character:
            rect = character.get_rect(midbottom=(WIDTH // 2, HEIGHT - 185))
            self.screen.blit(character, rect)

        ready = self.visible_chars >= len(self.node.text)
        ending = ready and not self.node.choices and not self.node.next_id
        self.ui.draw_dialogue(self.screen, self.node.speaker,
                              self.node.text[:int(self.visible_chars)], ready, ending)
        if ready and self.node.choices:
            self.ui.draw_choices(self.screen, self.node.choices, pygame.mouse.get_pos())
        self.ui.draw_hint(self.screen)
        pygame.display.flip()

    def run(self):
        while self.running:
            dt = min(self.clock.tick(FPS) / 1000, 0.1)
            for event in pygame.event.get():
                self.handle_event(event)
            self.visible_chars = min(len(self.node.text), self.visible_chars + TEXT_SPEED * dt)
            self.draw()
        pygame.quit()
```

---

## 8. Пишем `run.py` — Точка входа

Это самый маленький файл. Он создаёт `Game` и запускает метод `run()`. Проверка `if __name__ == "__main__"` означает, что программа запускается только при непосредственном выполнении файла, а не при его импорте из другого модуля.

**Проверка:** выполните `python run.py` из корня проекта.

**Полное содержимое файла:**

```python
from novel.app import Game

if __name__ == "__main__":
    Game().run()
```

---

## 9. Запускаем и проверяем по шагам

1. Запустите `python run.py`. Должны появиться фон-заглушка, персонаж и окно диалога.
2. Дождитесь появления текста. Нажмите `Пробел` один раз — будет переход к следующей реплике; в середине печати первое нажатие лишь показывает оставшиеся буквы.
3. На узле `question` нажмите на вариант А, проверьте концовку. Нажмите `R` и проверьте вариант Б.
4. Нажмите `S`, перейдите дальше, затем `L` — загрузится сохранённая реплика.
5. Закройте программу через `Esc` или крестик окна.

### Как движок обрабатывает действие

```text
Клавиатура / мышь
       ↓
Game.handle_event(event)
       ↓
Game.advance() или Game.go_to(node_id)
       ↓
Story.get(node_id) → объект Node
       ↓
Game.draw()
       ├── Resources.background() / Resources.character()
       └── Interface.draw_dialogue() / draw_choices()
       ↓
pygame.display.flip()
```

### Почему такая архитектура удобна

Чтобы написать новый сюжет, меняем **только JSON** и PNG. Чтобы изменить оформление кнопок, работаем в **`ui.py`**. Чтобы добавить новую игровую механику — обновляем её данные в `story.py`, поведение в `app.py`, а визуальное отображение в `ui.py`. Так изменения не превращаются в один огромный файл.

## 10. Практические задания: сначала простое, затем сложное

1. **Настройки:** поменяйте заголовок окна, размер и скорость текста. Проверьте, что диалоги продолжают работать.
2. **Новый узел:** добавьте между `intro` и `question` реплику `extra` и правильную ссылку `next`.
3. **Третья концовка:** добавьте `ending_c` и третью кнопку. Убедитесь, что каждое ответвление заканчивается своим узлом.
4. **Фоны:** добавьте `assets/backgrounds/room.png` и используйте `"background": "room"` в JSON.
5. **Персонаж:** добавьте прозрачный `assets/characters/hero.png`; задайте `"character": "hero"`.
6. **Скрытие диалога:** добавьте булево поле `dialogue_visible` в `Game` и переключение по `H`. Подсказка: меняйте его в `handle_event`, используйте в `draw`.
7. **История сообщений:** заведите список просмотренных реплик, добавляйте запись при переходе к новому узлу. Сделайте отдельный экран истории.
8. **Переменные выбора:** заведите словарь `variables`, научите сюжетные узлы проверять условия. Сначала продумайте формат JSON и валидатор, затем поведение и интерфейс.

**Важно:** задания 6–8 — расширения, их готового кода в стартовом движке нет. Не пытайтесь решить их переписыванием всего `app.py`.

## 11. Частые ошибки и как их найти

- **`ModuleNotFoundError: pygame`:** установите библиотеку в тот интерпретатор, которым запускаете игру: `python -m pip install -r requirements.txt`.
- **`JSONDecodeError`:** в JSON нужны двойные кавычки, нельзя ставить запятую после последнего элемента.
- **`ValueError: ... несуществующий ...`:** поле `next` или кнопка указывает на ID, которого нет в `nodes`.
- **`ImportError` / `ModuleNotFoundError: novel`:** запускать нужно `python run.py` из корня проекта; проверьте наличие `novel/__init__.py`.
- **Не отображается PNG:** название файла должно совпадать с `background` или `character` в JSON, учитывайте регистр букв.
- **Прямоугольная подложка у персонажа:** используйте PNG с прозрачностью; `convert_alpha()` её сохраняет.
- **Текст не влезает:** в учебной версии диалог рассчитан на три строки; разбивайте длинные речи на несколько узлов.
- **Русские буквы выглядят квадратами:** замените `arial` в `font()` на установленный шрифт, поддерживающий кириллицу.

## Итог

Вы не просто скопировали одну игру, а собрали части небольшого движка: **сценарий → модели → ресурсы → интерфейс → игровое состояние → запуск**. Теперь каждый модуль можно изменять отдельно, и новые истории не требуют переписывать игровой цикл.
