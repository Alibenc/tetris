# Практическая работа: создаём «Тетрис» на PyGame

**Уровень:** Python на базовом уровне, PyGame с нуля  
**Продолжительность:** 3–4 занятия по 60 минут (или самостоятельный проект)  
**Результат:** работающий Тетрис с движением, вращением, удалением заполненных строк, очками и проигрышем.

> **Как работать:** создайте файл `main.py`. На каждом этапе добавляйте указанный код в обозначенное место, а не создавайте новый файл. Не переходите дальше, пока предыдущий этап не работает. После каждого этапа есть задания без готового решения.

## 1. Установка и первое окно

Установите библиотеку в терминале:

```bash
python -m pip install pygame
```

Создайте `main.py`:

```python
import pygame
import random

pygame.init()

CELL = 30
COLS = 10
ROWS = 20
WIDTH = COLS * CELL
HEIGHT = ROWS * CELL

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Мой Тетрис")
clock = pygame.time.Clock()

BLACK = (20, 20, 28)
GRID = (55, 55, 65)

running = True
while running:
    clock.tick(60)
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    screen.fill(BLACK)
    pygame.display.flip()

pygame.quit()
```

**Что происходит:** `pygame.init()` запускает модули библиотеки; `set_mode` создаёт окно; цикл `while` повторяет действия каждый кадр; `event.get()` получает события; `flip()` показывает новый кадр; `tick(60)` ограничивает частоту до 60 кадров в секунду.

**Задание:** измените заголовок окна и цвет фона. Вычислите самостоятельно, почему ширина окна 300, а высота 600 пикселей.

## 2. Рисуем игровую сетку

Тетрис удобнее строить не по отдельным пикселям, а по **клеткам**. Клетка с координатами `(x, y)` рисуется на экране в точке `(x * CELL, y * CELL)`.

**Перед `running = True`** добавьте функцию:

```python
def draw_grid():
    for x in range(COLS):
        for y in range(ROWS):
            rect = pygame.Rect(x * CELL, y * CELL, CELL, CELL)
            pygame.draw.rect(screen, GRID, rect, 1)
```

**В игровом цикле** сразу после `screen.fill(BLACK)` добавьте:

```python
    draw_grid()
```

Параметр `1` в `pygame.draw.rect` означает, что мы рисуем только контур.

**Задание:** измените размер клетки на 25. Что произошло с окном? Почему сетка по-прежнему занимает его целиком?

## 3. Описываем фигуры

Фигура — это список из четырёх пар координат. Например, квадрат состоит из клеток `(0,0)`, `(1,0)`, `(0,1)`, `(1,1)`.

**Перед функцией `draw_grid`** добавьте:

```python
SHAPES = [
    [(0, 0), (1, 0), (0, 1), (1, 1)],   # O — квадрат
    [(0, 0), (1, 0), (2, 0), (3, 0)],   # I — линия
    [(0, 0), (1, 0), (2, 0), (1, 1)],   # T
    [(0, 0), (1, 0), (1, 1), (2, 1)],   # Z
    [(1, 0), (2, 0), (0, 1), (1, 1)],   # S
    [(0, 0), (0, 1), (1, 1), (2, 1)],   # J
    [(2, 0), (0, 1), (1, 1), (2, 1)],   # L
]

COLORS = [
    (240, 210, 65), (60, 210, 230), (175, 90, 220),
    (235, 75, 75), (85, 205, 100), (75, 115, 225),
    (245, 155, 60),
]
```

Создадим текущую фигуру. **Перед `running = True`**:

```python
def new_piece():
    kind = random.randrange(len(SHAPES))
    return {"cells": SHAPES[kind][:], "x": 3, "y": 0, "color": COLORS[kind]}

piece = new_piece()
```

Словарь `piece` хранит форму, позицию и цвет. Запись `[:]` создаёт отдельный список клеток.

Добавьте функцию рисования:

```python
def draw_piece(p):
    for dx, dy in p["cells"]:
        x = (p["x"] + dx) * CELL
        y = (p["y"] + dy) * CELL
        pygame.draw.rect(screen, p["color"], (x, y, CELL, CELL))
        pygame.draw.rect(screen, BLACK, (x, y, CELL, CELL), 2)
```

**В игровом цикле после `draw_grid()`** добавьте:

```python
    draw_piece(piece)
```

**Проверьте:** при каждом запуске должна появляться одна из семи фигур.

**Задание:** нарисуйте на бумаге фигуру `T` по её координатам. Попробуйте изменить одну пару чисел и предсказать результат.

## 4. Двигаем фигуру с клавиатуры

Нужно запрещать движение за границы поля и сквозь другие блоки. Для этого создадим игровое поле: 20 строк, в каждой 10 ячеек. Значение `None` означает пустую клетку; цвет — занятую.

**Перед `piece = new_piece()`**:

```python
board = [[None for _ in range(COLS)] for _ in range(ROWS)]
```

Добавьте функцию проверки положения:

```python
def valid_position(p, cells=None, offset_x=0, offset_y=0):
    if cells is None:
        cells = p["cells"]

    for dx, dy in cells:
        x = p["x"] + dx + offset_x
        y = p["y"] + dy + offset_y
        if x < 0 or x >= COLS or y >= ROWS:
            return False
        if y >= 0 and board[y][x] is not None:
            return False
    return True
```

**Внутри `for event in pygame.event.get():`**, после проверки `QUIT`, добавьте:

```python
        if event.type == pygame.KEYDOWN:
            if event.key == pygame.K_LEFT and valid_position(piece, offset_x=-1):
                piece["x"] -= 1
            if event.key == pygame.K_RIGHT and valid_position(piece, offset_x=1):
                piece["x"] += 1
            if event.key == pygame.K_DOWN and valid_position(piece, offset_y=1):
                piece["y"] += 1
```

`KEYDOWN` срабатывает при нажатии клавиши. `valid_position` проверяет, можно ли передвинуть фигуру.

**Задание:** добавьте управление `A`, `D`, `S` в дополнение к стрелкам. Почему нельзя просто уменьшать `piece["x"]` без проверки?

## 5. Фигура падает автоматически

FPS и скорость падения — разные вещи. FPS задаёт частоту обновления экрана, а падение мы запускаем по таймеру.

**Перед `running = True`** добавьте:

```python
fall_timer = 0
fall_delay = 500  # миллисекунд на одну клетку
```

**В начале игрового цикла** замените `clock.tick(60)` на:

```python
    dt = clock.tick(60)
    fall_timer += dt
```

**После обработки всех событий, но перед `screen.fill(BLACK)`**:

```python
    if fall_timer >= fall_delay:
        fall_timer = 0
        if valid_position(piece, offset_y=1):
            piece["y"] += 1
```

`dt` — количество миллисекунд, прошедших с предыдущего кадра.

**Пока что фигура остановится внизу, и это нормально.** На следующем этапе научим игру сохранять её на поле.

**Задание:** измените `fall_delay` на 200 и 1000. Что происходит? Почему не стоит использовать `pygame.time.wait(500)` внутри игрового цикла?

## 6. Фиксируем упавшую фигуру и создаём новую

Добавьте функцию, которая переносит клетки фигуры в `board`:

```python
def lock_piece(p):
    for dx, dy in p["cells"]:
        x = p["x"] + dx
        y = p["y"] + dy
        if 0 <= y < ROWS:
            board[y][x] = p["color"]
```

Добавьте рисование уже упавших блоков:

```python
def draw_board():
    for y in range(ROWS):
        for x in range(COLS):
            color = board[y][x]
            if color is not None:
                rect = (x * CELL, y * CELL, CELL, CELL)
                pygame.draw.rect(screen, color, rect)
                pygame.draw.rect(screen, BLACK, rect, 2)
```

**В игровом цикле после `screen.fill(BLACK)`**, но перед `draw_grid()`, добавьте:

```python
    draw_board()
```

**Замените блок автоматического падения из этапа 5** следующим:

```python
    if fall_timer >= fall_delay:
        fall_timer = 0
        if valid_position(piece, offset_y=1):
            piece["y"] += 1
        else:
            lock_piece(piece)
            piece = new_piece()
```

Теперь фигуры остаются на поле, а сверху появляется следующая.

**Задание:** объясните, зачем мы храним отдельно `board` и `piece`. Что будет, если убрать `lock_piece(piece)`?

## 7. Удаляем заполненные строки и считаем очки

Если строка заполнена целиком, она исчезает, а всё, что выше, опускается вниз.

**Перед `running = True`** добавьте:

```python
score = 0
font = pygame.font.SysFont("Arial", 24)
```

Добавьте функцию:

```python
def clear_lines():
    global board
    remaining = [row for row in board if any(cell is None for cell in row)]
    removed = ROWS - len(remaining)
    board = [[None for _ in range(COLS)] for _ in range(removed)] + remaining
    return removed
```

**В блоке падения**, после `lock_piece(piece)` и перед `piece = new_piece()`, добавьте:

```python
            lines = clear_lines()
            score += lines * 100
```

**В игровом цикле после `draw_piece(piece)`**, но перед `pygame.display.flip()`, добавьте:

```python
    score_surface = font.render(f"Очки: {score}", True, (255, 255, 255))
    screen.blit(score_surface, (8, 8))
```

**Разбор:** `any(cell is None for cell in row)` проверяет, есть ли в строке хоть одна пустая клетка. Если нет — строка заполнена и не попадает в `remaining`. Затем сверху добавляются пустые строки.

**Задание:** начисляйте 100, 300, 500 и 800 очков за удаление соответственно 1, 2, 3 и 4 строк за один раз. Подсказка: используйте список значений.

## 8. Вращение фигур

Вращать будем координаты относительно начала фигуры. Для поворота на 90° по часовой стрелке используется преобразование `(x, y) → (-y, x)`. После поворота сдвинем фигуру так, чтобы её минимальные координаты стали нулевыми.

Добавьте функцию:

```python
def rotate_cells(cells):
    rotated = [(-y, x) for x, y in cells]
    min_x = min(x for x, y in rotated)
    min_y = min(y for x, y in rotated)
    return [(x - min_x, y - min_y) for x, y in rotated]
```

**В обработку `KEYDOWN`** добавьте:

```python
            if event.key == pygame.K_UP:
                rotated = rotate_cells(piece["cells"])
                if valid_position(piece, cells=rotated):
                    piece["cells"] = rotated
```

**Важно:** это упрощённое вращение, не полная система поворотов современного Тетриса. Возле стены поворот иногда будет запрещён, даже если его можно было бы выполнить со сдвигом.

**Задание:** запретите вращение квадратной фигуры `O` (у неё и так одинаковый вид при повороте). Подсказка: можно добавить тип фигуры в словарь `piece`.

## 9. Проигрыш и перезапуск

Когда новой фигуре некуда появиться, игра заканчивается.

**Перед `running = True`** добавьте:

```python
game_over = False
```

**В блоке автоматического падения** после `piece = new_piece()` добавьте:

```python
            if not valid_position(piece):
                game_over = True
```

Чтобы игра перестала двигать фигуру после проигрыша, **замените условие таймера**:

```python
    if fall_timer >= fall_delay:
```

на:

```python
    if not game_over and fall_timer >= fall_delay:
```

И **внутри `if event.type == pygame.KEYDOWN:`** оберните управление фигурой условием `if not game_over:`. Проще всего заменить весь обработчик на:

```python
        if event.type == pygame.KEYDOWN:
            if game_over:
                if event.key == pygame.K_r:
                    board = [[None for _ in range(COLS)] for _ in range(ROWS)]
                    piece = new_piece()
                    score = 0
                    fall_timer = 0
                    game_over = False
            else:
                if event.key == pygame.K_LEFT and valid_position(piece, offset_x=-1):
                    piece["x"] -= 1
                if event.key == pygame.K_RIGHT and valid_position(piece, offset_x=1):
                    piece["x"] += 1
                if event.key == pygame.K_DOWN and valid_position(piece, offset_y=1):
                    piece["y"] += 1
                if event.key == pygame.K_UP:
                    rotated = rotate_cells(piece["cells"])
                    if valid_position(piece, cells=rotated):
                        piece["cells"] = rotated
```

**В игровом цикле после вывода очков** добавьте:

```python
    if game_over:
        overlay = pygame.Surface((WIDTH, HEIGHT), pygame.SRCALPHA)
        overlay.fill((0, 0, 0, 180))
        screen.blit(overlay, (0, 0))
        message = font.render("Игра окончена", True, (255, 255, 255))
        hint = font.render("R — заново", True, (255, 255, 255))
        screen.blit(message, message.get_rect(center=(WIDTH // 2, HEIGHT // 2 - 20)))
        screen.blit(hint, hint.get_rect(center=(WIDTH // 2, HEIGHT // 2 + 20)))
```

**Задание:** сделайте так, чтобы на экране проигрыша показывался итоговый счёт.

## 10. Проверяем игру

Пройдитесь по чек-листу:

- [ ] Окно открывается и закрывается без ошибок.
- [ ] На поле видна сетка 10 × 20.
- [ ] Появляются случайные фигуры разных цветов.
- [ ] Стрелки влево и вправо двигают фигуру.
- [ ] Стрелка вниз ускоряет опускание на одну клетку при нажатии.
- [ ] Стрелка вверх вращает фигуру, если хватает места.
- [ ] Фигуры не выходят за границы и не проходят сквозь блоки.
- [ ] Упавшие фигуры остаются на поле.
- [ ] Полные строки удаляются, начисляются очки.
- [ ] Когда поле заполнено до верха, появляется проигрыш.
- [ ] Клавиша R перезапускает игру.

### Дополнительные задания (делайте самостоятельно)

1. **Простой уровень:** показывайте количество удалённых линий рядом с очками.
2. **Средний уровень:** добавьте пробел для мгновенного падения фигуры вниз (`hard drop`). Не забудьте после этого зафиксировать фигуру.
3. **Средний уровень:** каждые 5 удалённых строк уменьшайте `fall_delay`, но не ниже 100 мс.
4. **Сложный уровень:** показывайте справа следующую фигуру. Для этого увеличьте ширину окна и храните `next_piece`.
5. **Сложный уровень:** реализуйте «поворот у стены» — попробуйте сдвинуть фигуру на одну клетку, если вращение на месте невозможно.
6. **Сложный уровень:** добавьте паузу на клавишу `P`.

## Вопросы для самопроверки

1. Зачем нужен главный цикл игры?
2. Чем клеточные координаты отличаются от пиксельных?
3. Для чего используется двумерный список `board`?
4. Почему нельзя двигать фигуру, не проверяя новое положение?
5. Что делает `clock.tick(60)` и чем он отличается от `fall_delay`?
6. Как программа понимает, что строка заполнена?
7. Почему после удаления строки нужно добавлять пустую строку сверху?
8. Зачем мы храним текущую фигуру отдельно от зафиксированных блоков?
9. Как определяется конец игры?
10. Что нужно сбросить при перезапуске?

---

**Совет преподавателю.** После этапов 1–3 попросите детей объяснить систему координат и нарисовать фигуру по списку клеток. После этапов 4–6 дайте им самостоятельно найти место для проверки столкновения. Этапы 7–9 удобно вынести на второе или третье занятие. В инструкции специально нет единого готового файла: ученики собирают проект по частям и видят, как появляется каждая механика.
