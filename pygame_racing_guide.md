# Практическая работа: «Гонки с препятствиями» на Pygame

**Сложность:** средняя (6/10)  
**Цель:** создать гоночную игру с видом сверху: движение по дороге, встречные автомобили, столкновения, очки, рекорд, ускорение и перезапуск.

## 1. Подготовка

Установите библиотеку командой `pip install pygame`. Создайте файл `main.py`.

```python
import pygame
import random

pygame.init()
WIDTH, HEIGHT = 800, 700
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption('Гонки с препятствиями')
clock = pygame.time.Clock()
running = True

while running:
    clock.tick(60)
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
    screen.fill((40, 150, 40))
    pygame.display.flip()

pygame.quit()
```

Запустите программу. Убедитесь, что окно открывается и закрывается.

## 2. Дорога

Перед циклом задайте размеры и цвета:

```python
ROAD_WIDTH = 400
ROAD_X = (WIDTH - ROAD_WIDTH) // 2
GRASS = (40, 150, 40)
ROAD = (65, 65, 65)
WHITE = (255, 255, 255)
```

Вместо старой заливки внутри цикла нарисуйте дорогу:

```python
screen.fill(GRASS)
pygame.draw.rect(screen, ROAD, (ROAD_X, 0, ROAD_WIDTH, HEIGHT))
pygame.draw.line(screen, WHITE, (ROAD_X, 0), (ROAD_X, HEIGHT), 5)
pygame.draw.line(screen, WHITE, (ROAD_X + ROAD_WIDTH, 0),
                 (ROAD_X + ROAD_WIDTH, HEIGHT), 5)
```

**Задание:** поменяйте ширину дороги, сохранив её по центру.

## 3. Движущаяся разметка

Перед циклом:

```python
LINE_HEIGHT = 60
LINE_GAP = 40
line_offset = 0
road_speed = 5
```

Внутри цикла, после рисования дороги:

```python
line_offset = (line_offset + road_speed) % (LINE_HEIGHT + LINE_GAP)
for lane in range(1, 3):
    line_x = ROAD_X + lane * ROAD_WIDTH // 3
    for y in range(-LINE_HEIGHT - LINE_GAP, HEIGHT,
                   LINE_HEIGHT + LINE_GAP):
        pygame.draw.rect(screen, WHITE,
                         (line_x - 3, y + line_offset, 6, LINE_HEIGHT))
```

**Объяснение:** разметка повторяется через равные промежутки; изменение `line_offset` создаёт иллюзию движения.

**Задание:** измените скорость и длину линий.

## 4. Машина игрока

Перед циклом:

```python
CAR_WIDTH, CAR_HEIGHT = 55, 100
player_x = WIDTH // 2 - CAR_WIDTH // 2
player_y = HEIGHT - CAR_HEIGHT - 30
player_speed = 7
PLAYER_COLOR = (30, 120, 255)
```

Внутри цикла перед отрисовкой создайте прямоугольник и нарисуйте его:

```python
player_rect = pygame.Rect(player_x, player_y, CAR_WIDTH, CAR_HEIGHT)
pygame.draw.rect(screen, PLAYER_COLOR, player_rect, border_radius=10)
```

**Задание:** измените цвет машины.

## 5. Управление

Перед созданием `player_rect` внутри цикла:

```python
keys = pygame.key.get_pressed()
if keys[pygame.K_LEFT] or keys[pygame.K_a]:
    player_x -= player_speed
if keys[pygame.K_RIGHT] or keys[pygame.K_d]:
    player_x += player_speed

player_x = max(ROAD_X + 10,
               min(player_x, ROAD_X + ROAD_WIDTH - CAR_WIDTH - 10))
```

**Задание:** проверьте, что автомобиль не может выехать за границы дороги.

## 6. Встречные машины

Перед циклом:

```python
enemies = []
enemy_width, enemy_height = 55, 100
enemy_speed = 5
spawn_timer = 0
spawn_delay = 90
ENEMY_COLOR = (230, 50, 50)
```

Внутри цикла, до отрисовки машин:

```python
spawn_timer += 1
if spawn_timer >= spawn_delay:
    spawn_timer = 0
    lane = random.randint(0, 2)
    lane_width = ROAD_WIDTH // 3
    enemy_x = ROAD_X + lane * lane_width + (lane_width - enemy_width) // 2
    enemies.append(pygame.Rect(enemy_x, -enemy_height,
                               enemy_width, enemy_height))

for enemy in enemies[:]:
    enemy.y += enemy_speed
    if enemy.top > HEIGHT:
        enemies.remove(enemy)
```

В части отрисовки:

```python
for enemy in enemies:
    pygame.draw.rect(screen, ENEMY_COLOR, enemy, border_radius=10)
```

**Объяснение:** случайно выбирается одна из трёх полос. Машины хранятся в списке и перемещаются вниз.

**Задание:** измените частоту появления автомобилей.

## 7. Столкновения и разделение логики

Перед циклом:

```python
game_over = False
```

После перемещения врагов:

```python
for enemy in enemies:
    if player_rect.colliderect(enemy):
        game_over = True
```

**Важно:** с этого момента обновление движения игрока, разметки, врагов, таймеров и проверку столкновений поместите внутрь `if not game_over:`. Создавайте `player_rect` после изменения `player_x`, но до проверки столкновений. Рисование дороги, разметки, игрока, врагов и интерфейса выполняйте каждый кадр, вне этого условия. Разметку рисуйте по текущему `line_offset`, а обновляйте `line_offset` только во время игры.

## 8. Очки и рекорд

Перед циклом:

```python
score = 0
record = 0
font = pygame.font.SysFont('Arial', 32)
```

В месте удаления машины, выехавшей за нижний край, добавьте:

```python
if enemy.top > HEIGHT:
    enemies.remove(enemy)
    score += 1
    record = max(record, score)
```

Это **заменяет** старую проверку `if enemy.top > HEIGHT`, а не дублирует её.

В части отрисовки:

```python
screen.blit(font.render(f'Очки: {score}', True, WHITE), (20, 20))
screen.blit(font.render(f'Рекорд: {record}', True, WHITE), (20, 60))
```

**Задание:** начисляйте 10 очков за машину.

## 9. Увеличение сложности

Перед циклом:

```python
level = 1
```

После увеличения `score` при удалении встречной машины:

```python
new_level = score // 5 + 1
if new_level > level:
    level = new_level
    enemy_speed += 0.5
    road_speed += 0.5
    spawn_delay = max(35, spawn_delay - 5)
```

**Почему так:** уровень увеличивается только при достижении нового порога, а не каждый кадр.

В отрисовке:

```python
screen.blit(font.render(f'Уровень: {level}', True, WHITE), (20, 100))
```

## 10. Экран проигрыша

После рисования всех объектов и интерфейса, но перед `pygame.display.flip()`:

```python
if game_over:
    overlay = pygame.Surface((WIDTH, HEIGHT))
    overlay.set_alpha(180)
    overlay.fill((0, 0, 0))
    screen.blit(overlay, (0, 0))

    message = font.render('Игра окончена!', True, WHITE)
    restart = font.render('Нажмите R для перезапуска', True, WHITE)
    screen.blit(message, message.get_rect(center=(WIDTH // 2, HEIGHT // 2 - 30)))
    screen.blit(restart, restart.get_rect(center=(WIDTH // 2, HEIGHT // 2 + 20)))
```

## 11. Перезапуск

Внутри цикла `for event in pygame.event.get():` добавьте обработчик:

```python
if event.type == pygame.KEYDOWN:
    if event.key == pygame.K_r and game_over:
        game_over = False
        score = 0
        level = 1
        enemy_speed = 5
        road_speed = 5
        spawn_timer = 0
        spawn_delay = 90
        line_offset = 0
        enemies.clear()
        player_x = WIDTH // 2 - CAR_WIDTH // 2
```

Рекорд не сбрасывайте.

## 12. Проверка готовой игры

- [ ] Окно запускается и закрывается.
- [ ] Дорога находится посередине.
- [ ] Разметка движется вниз.
- [ ] Игрок двигается влево и вправо, не выезжая за дорогу.
- [ ] Встречные машины появляются в случайных полосах.
- [ ] За успешно пропущенную машину начисляются очки.
- [ ] При столкновении движение останавливается.
- [ ] Каждые пять очков сложность повышается.
- [ ] После проигрыша можно нажать R.
- [ ] Рекорд сохраняется между перезапусками в пределах одного запуска программы.

## Дополнительные задания

1. **Легко:** добавьте разные цвета встречных автомобилей.
2. **Легко:** добавьте жёлтые полосы по краям дороги.
3. **Средне:** сделайте три жизни вместо мгновенного проигрыша; после удара добавьте временную неуязвимость.
4. **Средне:** добавьте меню старта с клавишей Enter.
5. **Средне:** замените прямоугольники на изображения машин через `pygame.image.load()` и `pygame.transform.scale()`.
6. **Сложно:** добавьте звук столкновения и музыку.
7. **Сложно:** сохраняйте рекорд в текстовом файле, чтобы он оставался после закрытия игры.
8. **Сложно:** запретите создание новой машины в полосе, если вверху слишком близко уже есть другая.

**Подсказка:** сначала добейтесь работы простой версии с прямоугольниками. Украшения добавляйте только после того, как столкновения, очки и перезапуск работают правильно.
