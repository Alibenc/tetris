# Практическая работа: «Дуэль рыцарей» на Pygame

**Сложность:** 7/10  
**Язык:** Python  
**Библиотека:** Pygame  
**Цель:** создать игру для двух игроков на одной клавиатуре.

## Что получится

Два рыцаря сражаются на арене. Каждый умеет ходить, атаковать мечом и защищаться щитом. Побеждает тот, кто первым уменьшит здоровье соперника до нуля. После завершения боя можно начать новый раунд.

| Действие | Игрок 1 | Игрок 2 |
|---|---|---|
| Движение влево | A | ← |
| Движение вправо | D | → |
| Атака | F | K |
| Защита | G | L |
| Перезапуск | R | R |

## Этап 1. Подготовка

Установите Pygame:

```bash
pip install pygame
```

Создайте папку `knight_duel` и файл `main.py`. Начните с игрового окна:

```python
import pygame

pygame.init()

WIDTH = 1000
HEIGHT = 600
FPS = 60

screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Дуэль рыцарей")
clock = pygame.time.Clock()

running = True
while running:
    clock.tick(FPS)

    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    screen.fill((100, 140, 180))
    pygame.display.flip()

pygame.quit()
```

**Проверьте:** окно открывается и закрывается крестиком.

## Этап 2. Рисуем арену

Перед циклом объявите:

```python
GROUND_Y = 500
SKY_COLOR = (100, 140, 180)
GROUND_COLOR = (85, 65, 50)
```

В цикле замените `screen.fill(...)` на:

```python
screen.fill(SKY_COLOR)
pygame.draw.rect(screen, GROUND_COLOR, (0, GROUND_Y, WIDTH, HEIGHT - GROUND_Y))
```

**Задание:** нарисуйте облака и башни замка с помощью `pygame.draw.rect()` и `pygame.draw.circle()`.

## Этап 3. Добавляем двух рыцарей

Перед циклом создайте словари:

```python
KNIGHT_WIDTH = 60
KNIGHT_HEIGHT = 100

player1 = {
    "rect": pygame.Rect(150, GROUND_Y - KNIGHT_HEIGHT, KNIGHT_WIDTH, KNIGHT_HEIGHT),
    "color": (50, 100, 230),
    "speed": 5,
    "health": 100,
    "direction": 1,
    "blocking": False,
    "attack_timer": 0,
    "attack_flash": 0
}

player2 = {
    "rect": pygame.Rect(790, GROUND_Y - KNIGHT_HEIGHT, KNIGHT_WIDTH, KNIGHT_HEIGHT),
    "color": (220, 60, 60),
    "speed": 5,
    "health": 100,
    "direction": -1,
    "blocking": False,
    "attack_timer": 0,
    "attack_flash": 0
}
```

`rect` хранит положение персонажа; `direction` равен `1` при взгляде вправо и `-1` при взгляде влево.

Перед `pygame.display.flip()` нарисуйте рыцарей:

```python
pygame.draw.rect(screen, player1["color"], player1["rect"], border_radius=8)
pygame.draw.rect(screen, player2["color"], player2["rect"], border_radius=8)
```

**Проверьте:** на земле стоят два прямоугольника разных цветов.

## Этап 4. Управление

Внутри цикла, после обработки событий и перед рисованием, добавьте:

```python
keys = pygame.key.get_pressed()

if keys[pygame.K_a]:
    player1["rect"].x -= player1["speed"]
    player1["direction"] = -1
if keys[pygame.K_d]:
    player1["rect"].x += player1["speed"]
    player1["direction"] = 1

if keys[pygame.K_LEFT]:
    player2["rect"].x -= player2["speed"]
    player2["direction"] = -1
if keys[pygame.K_RIGHT]:
    player2["rect"].x += player2["speed"]
    player2["direction"] = 1

player1["rect"].clamp_ip(screen.get_rect())
player2["rect"].clamp_ip(screen.get_rect())
```

`clamp_ip()` не даёт персонажу уйти за границы окна. Поскольку движение только горизонтальное, оба рыцаря остаются на земле.

**Задание:** попробуйте поменять скорость второго рыцаря на 7.

## Этап 5. Область удара

Перед игровым циклом объявите:

```python
ATTACK_RANGE = 55
ATTACK_DAMAGE = 15
ATTACK_COOLDOWN = 40
BLOCKED_DAMAGE = 3
```

После словарей игроков, но до игрового цикла, создайте функцию:

```python
def get_attack_rect(player):
    rect = player["rect"]
    if player["direction"] == 1:
        return pygame.Rect(rect.right, rect.y + 20, ATTACK_RANGE, 40)
    return pygame.Rect(rect.left - ATTACK_RANGE, rect.y + 20, ATTACK_RANGE, 40)
```

Функция создаёт невидимый прямоугольник перед рыцарем. Если этот прямоугольник пересечётся с соперником, удар попадёт в цель.

Для проверки временно добавьте в часть отрисовки:

```python
pygame.draw.rect(screen, (255, 230, 0), get_attack_rect(player1), 2)
pygame.draw.rect(screen, (255, 230, 0), get_attack_rect(player2), 2)
```

**Проверьте:** при развороте рыцаря область удара перемещается на другую сторону.

## Этап 6. Удар и перезарядка

Добавьте функцию после `get_attack_rect()`:

```python
def attack(attacker, defender):
    if attacker["attack_timer"] > 0:
        return

    attacker["attack_timer"] = ATTACK_COOLDOWN
    attacker["attack_flash"] = 10

    if get_attack_rect(attacker).colliderect(defender["rect"]):
        damage = BLOCKED_DAMAGE if defender["blocking"] else ATTACK_DAMAGE
        defender["health"] = max(0, defender["health"] - damage)
```

В обработчике событий внутри `for event in pygame.event.get()` добавьте:

```python
if event.type == pygame.KEYDOWN:
    if event.key == pygame.K_f:
        attack(player1, player2)
    if event.key == pygame.K_k:
        attack(player2, player1)
```

Внутри основного цикла, после обработки событий, уменьшайте таймеры:

```python
for player in (player1, player2):
    if player["attack_timer"] > 0:
        player["attack_timer"] -= 1
    if player["attack_flash"] > 0:
        player["attack_flash"] -= 1
```

**Почему используем `KEYDOWN`, а не `get_pressed()`?** Событие возникает при нажатии клавиши, поэтому одна попытка удара не повторяется каждый кадр. Таймер не позволяет атаковать слишком часто.

## Этап 7. Защита щитом

После строки `keys = pygame.key.get_pressed()` добавьте:

```python
player1["blocking"] = keys[pygame.K_g]
player2["blocking"] = keys[pygame.K_l]
```

В функции `attack()` защита уже учитывается: вместо 15 единиц здоровья блокированный удар снимает только 3.

Чтобы щит был виден, добавьте в часть отрисовки после рыцарей:

```python
for player in (player1, player2):
    if player["blocking"]:
        rect = player["rect"]
        shield_x = rect.right if player["direction"] == 1 else rect.left - 12
        pygame.draw.rect(screen, (180, 190, 210), (shield_x, rect.y + 25, 12, 55), border_radius=5)
```

**Задание:** сделайте так, чтобы защищающийся рыцарь двигался вдвое медленнее. Для этого при расчёте движения используйте уменьшенную скорость, если `blocking == True`.

## Этап 8. Показываем взмах меча

Вместо постоянно видимых жёлтых прямоугольников из этапа 5 добавьте:

```python
for player in (player1, player2):
    if player["attack_flash"] > 0:
        pygame.draw.rect(screen, (255, 230, 0), get_attack_rect(player), 3)
```

Теперь рамка области удара будет появляться только на короткое время после атаки. Позже её можно заменить изображением меча.

## Этап 9. Полоски здоровья

После создания окна добавьте шрифт:

```python
font = pygame.font.SysFont("Arial", 32)
```

Перед игровым циклом создайте функцию:

```python
def draw_health(x, y, health, color):
    pygame.draw.rect(screen, (60, 60, 60), (x, y, 300, 30))
    pygame.draw.rect(screen, color, (x, y, 3 * health, 30))
    pygame.draw.rect(screen, (255, 255, 255), (x, y, 300, 30), 2)
```

В части отрисовки:

```python
draw_health(40, 30, player1["health"], (50, 100, 230))
draw_health(WIDTH - 340, 30, player2["health"], (220, 60, 60))
```

У каждого рыцаря 100 здоровья. Поэтому ширина заполненной полоски равна `health * 3`.

**Задание:** выведите над полосками надписи «Игрок 1» и «Игрок 2».

## Этап 10. Победа и остановка боя

Перед игровым циклом:

```python
winner = None
```

После обновления таймеров и перемещения игроков проверяйте:

```python
if player1["health"] <= 0:
    winner = "Победил игрок 2!"
elif player2["health"] <= 0:
    winner = "Победил игрок 1!"
```

Чтобы после победы бой остановился, **оберните игровую логику** (движение, защиту, атаки и таймеры) условием `if winner is None:`. В обработчике `KEYDOWN` атаки также должны выполняться только при `winner is None`:

```python
if event.type == pygame.KEYDOWN and winner is None:
    if event.key == pygame.K_f:
        attack(player1, player2)
    if event.key == pygame.K_k:
        attack(player2, player1)
```

Отрисовка должна выполняться всегда, даже когда бой окончен. В конце отрисовки добавьте:

```python
if winner is not None:
    message = font.render(winner, True, (255, 255, 255))
    screen.blit(message, message.get_rect(center=(WIDTH // 2, 220)))

    restart_message = font.render("Нажмите R для нового боя", True, (255, 255, 255))
    screen.blit(restart_message, restart_message.get_rect(center=(WIDTH // 2, 270)))
```

**Проверьте:** после того как здоровье одного рыцаря достигнет нуля, управление и атаки перестают работать.

## Этап 11. Перезапуск игры

Чтобы не копировать начальные значения в разных местах, создайте функцию **до основного цикла**:

```python
def reset_players():
    player1["rect"].topleft = (150, GROUND_Y - KNIGHT_HEIGHT)
    player2["rect"].topleft = (790, GROUND_Y - KNIGHT_HEIGHT)

    for player in (player1, player2):
        player["health"] = 100
        player["blocking"] = False
        player["attack_timer"] = 0
        player["attack_flash"] = 0

    player1["direction"] = 1
    player2["direction"] = -1
```

В обработчик событий добавьте отдельную проверку:

```python
if event.type == pygame.KEYDOWN:
    if event.key == pygame.K_r and winner is not None:
        reset_players()
        winner = None
```

**Проверьте:** после победы нажмите R. Рыцари должны вернуться на начальные позиции с полным здоровьем.

## Этап 12. Улучшаем внешний вид

Когда игра работает, можно заменить прямоугольники PNG-спрайтами:

```python
knight1_image = pygame.image.load("assets/knight_blue.png").convert_alpha()
knight1_image = pygame.transform.smoothscale(knight1_image, (KNIGHT_WIDTH, KNIGHT_HEIGHT))
```

Для отрисовки вместо `pygame.draw.rect()` используйте:

```python
screen.blit(knight1_image, player1["rect"])
```

Для второго рыцаря загрузите отдельный спрайт. Если изображение направлено не в ту сторону, можно использовать `pygame.transform.flip(image, True, False)`. Хитбоксы при этом по-прежнему хранятся в `player["rect"]`.

## Итоговая проверка

- [ ] Окно игры открывается и закрывается.
- [ ] На арене стоят два рыцаря.
- [ ] Оба игрока могут двигаться независимо.
- [ ] Рыцари не выходят за края окна.
- [ ] Удары работают только перед персонажем.
- [ ] Удар снимает здоровье только при попадании.
- [ ] Щит уменьшает получаемый урон.
- [ ] Между атаками есть перезарядка.
- [ ] Полоски здоровья обновляются.
- [ ] После победы бой останавливается.
- [ ] Клавиша R запускает новый раунд.

## Дополнительные задания

1. **Выносливость:** блокирование постепенно расходует запас энергии; когда энергия заканчивается, щит перестаёт работать.
2. **Отталкивание:** успешный удар немного сдвигает противника назад.
3. **Критический удар:** с небольшой вероятностью атака наносит удвоенный урон.
4. **Раунды:** матч продолжается до трёх побед одного игрока.
5. **Звуки:** добавьте отдельные звуки удара, блока и победы.
6. **Анимация:** замените прямоугольники последовательностями кадров для движения и атаки.
7. **ИИ:** создайте режим игры против компьютера, который подходит к игроку и атакует на близком расстоянии.

**Совет:** выполняйте этапы по порядку. После каждого этапа запускайте программу и проверяйте результат. Не переходите к следующей механике, пока предыдущая не заработала.
