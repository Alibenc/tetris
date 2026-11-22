# Делаем Hill Rider — свою игру в стиле Hill Climb Racing на Pygame

**Цель:** создать 2D-игру, в которой автомобиль едет по холмам, подпрыгивает, вращается в воздухе и собирает предметы. По мере продвижения холмы становятся выше и сложнее.

**Сложность:** средняя/выше средней. Здесь много интересной математики, но готовый проект уже разбит на небольшие файлы. Лучший подход — собирать его **по этапам**, запускать после каждого этапа и разбираться, почему машина едет именно так.

> В архиве находится готовый результат. Используй его как эталон: сначала попробуй написать нужную часть самостоятельно, затем сравни со своей реализацией. Не нужно переписывать весь код разом.

## 0. Подготовка проекта

Установи Python версии 3.10 или новее. Создай папку проекта `hill_rider` и открой её в редакторе.

```text
hill_rider/
├── main.py
├── requirements.txt
└── src/
    ├── __init__.py
    ├── settings.py
    ├── terrain.py
    ├── car.py
    └── game.py
```

В `requirements.txt`:

```text
pygame>=2.5,<3
```

В терминале редактора:

```bash
python -m pip install -r requirements.txt
```

На Windows, если `python` не работает, попробуй `py`.

**Проверка:** запусти `python -c "import pygame; print(pygame.version.ver)"`. Должна появиться версия Pygame.

---

## 1. Создаем окно и игровой цикл

Начни с одного файла `main.py`:

```python
import pygame

pygame.init()
screen = pygame.display.set_mode((1280, 720))
pygame.display.set_caption('Hill Rider')
clock = pygame.time.Clock()

running = True
while running:
    dt = min(clock.tick(60) / 1000, 1 / 30)
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False

    screen.fill((141, 208, 232))
    pygame.display.flip()

pygame.quit()
```

`clock.tick(60)` ограничивает FPS, а `dt` — продолжительность кадра **в секундах**. Ограничение `1 / 30` не позволяет физике резко прыгнуть при подвисании окна.

**Задание:** сделай заголовок окна и нарисуй солнце с помощью `pygame.draw.circle`.

---

## 2. Рисуем бесконечную дорогу

Нам не нужна большая картинка трассы. Достаточно функции, которая возвращает высоту земли для любой координаты `x`.

В Pygame координата `y` увеличивается **вниз**: меньшая `y` означает, что точка выше.

Создай `src/terrain.py`:

```python
import math
import pygame

class Terrain:
    def height(self, x):
        return 545 + math.sin(x * 0.006) * 40

    def draw(self, screen, camera_x):
        points = []
        for screen_x in range(0, 1281, 10):
            world_x = screen_x + camera_x
            points.append((screen_x, self.height(world_x)))
        pygame.draw.polygon(
            screen, (74, 111, 70),
            points + [(1280, 720), (0, 720)]
        )
        pygame.draw.lines(screen, (158, 196, 107), False, points, 6)
```

В игровом цикле вместо простой заливки добавь:

```python
terrain = Terrain()  # создать ОДИН раз, до игрового цикла
# внутри цикла:
terrain.draw(screen, camera_x=0)
```

**Идея:** `sin` дает плавную волну, поэтому поверхность не обрывается на случайных высотах. Позже мы смешаем несколько волн с разными частотами.

**Проверь:** земля рисуется плавно, без разрывов.

---

## 3. Добавляем камеру и мировые координаты

До этого мы рисовали трассу с `camera_x=0`. Теперь сделаем так, чтобы камера следовала за машиной. **С этого раздела используем ту же архитектуру, что и в готовом проекте**: `Game` управляет игрой, `Car` хранит положение и физику автомобиля, `Terrain` отвечает за землю.

### 3.1. Создаём класс `Car`

В `src/car.py` запиши:

```python
class Car:
    def __init__(self, x, terrain):
        self.x = float(x)                         # мировая координата машины
        self.y = terrain.height(x) - 54           # машина чуть выше поверхности
        self.vx = 0.0                             # скорость по горизонтали
        self.vy = 0.0                             # скорость по вертикали
        self.angle = 0.0                          # наклон кузова, радианы
        self.angular_velocity = 0.0               # скорость вращения
        self.on_ground = False
        self.crashed = False
        self.air_time = 0.0
```

Здесь **`self.x` — это то же самое, что `car_x` в простых примерах**. Отдельную глобальную переменную `car_x` мы не создаём. Аналогично `vx` и `vy` существуют как поля машины: `self.vx` и `self.vy`.

В другом классе доступ к ним выглядит как `self.car.x`, `self.car.vx`, `self.car.vy`.

### 3.2. Создаём `Game` и камеру

Добавь в `src/game.py`:

```python
import pygame
from .terrain import Terrain
from .car import Car

WIDTH, HEIGHT = 1280, 720
FPS = 60

class Game:
    def __init__(self):
        pygame.init()
        self.screen = pygame.display.set_mode((WIDTH, HEIGHT))
        pygame.display.set_caption('Hill Rider')
        self.clock = pygame.time.Clock()
        self.terrain = Terrain()
        self.car = Car(105, self.terrain)
        self.camera_x = 0.0
        self.camera_y = 0.0

    def update(self, dt, keys):
        # Временное управление, только для проверки камеры.
        if keys[pygame.K_RIGHT]:
            self.car.x += 250 * dt
        if keys[pygame.K_LEFT]:
            self.car.x = max(0, self.car.x - 250 * dt)
        self.car.y = self.terrain.height(self.car.x) - 54

        desired_camera_x = max(0, self.car.x - WIDTH * 0.30)
        self.camera_x += (desired_camera_x - self.camera_x) * min(1, dt * 5)

    def draw(self):
        self.screen.fill((141, 208, 232))
        self.terrain.draw(self.screen, self.camera_x)
        x_on_screen = int(self.car.x - self.camera_x)
        y_on_screen = int(self.car.y - self.camera_y)
        pygame.draw.rect(self.screen, (230, 95, 60),
                         (x_on_screen - 35, y_on_screen - 15, 70, 30))
        pygame.display.flip()

    def run(self):
        running = True
        while running:
            dt = min(self.clock.tick(FPS) / 1000, 1 / 30)
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    running = False
            self.update(dt, pygame.key.get_pressed())
            self.draw()
        pygame.quit()
```

Замени `main.py`:

```python
from src.game import Game

if __name__ == '__main__':
    Game().run()
```

**Как работает камера:** у машины есть мировая координата `self.car.x`, у камеры — `self.camera_x`. Чтобы нарисовать объект на экране, вычитаем координату камеры:

```python
screen_x = world_x - self.camera_x
```

**Проверка:** стрелками перемещай прямоугольник по трассе. Он должен оставаться примерно в левой трети экрана, когда камера догонит его. Это пока *тестовое перемещение*, позже заменим его физикой.

> Если в готовой игре встретишь `desired_camera_y` и `self.camera_y`, это дополнительное вертикальное слежение. Пока оно не требуется, но в финале мы его добавим.

---

## 4. Делаем машину физическим объектом

Теперь меняем `src/car.py`. Важно не путаться: внутри класса `Car` поля называются **`self.x`, `self.y`, `self.vx`, `self.vy`, `self.angle`, `self.angular_velocity`**. В `Game` доступ к ним осуществляется через `self.car`.

### 4.1. Гравитация и скорость

Добавь в `src/car.py` импорт:

```python
import math
import pygame
from .settings import GRAVITY, MASS, INERTIA
```

Параметры `GRAVITY`, `MASS`, `INERTIA` вынеси в `src/settings.py`:

```python
GRAVITY = 1250.0
MASS = 430.0
INERTIA = 155000.0
```

Внутри класса `Car` напиши первый вариант метода:

```python
    def update(self, dt, terrain, gas, brake):
        self.vy += GRAVITY * dt
        self.x += self.vx * dt
        self.y += self.vy * dt
```

Обрати внимание: **`update` — метод `Car`, а не `Game`**. В `Game.update` нужно вызвать его:

```python
self.car.update(dt, self.terrain, gas=False, brake=False)
```

Удали оттуда временный блок, который напрямую менял `self.car.x` и выставлял `self.car.y = terrain.height(...)`. Иначе машина будет «телепортироваться» обратно на землю каждый кадр.

**Проверка:** машина падает вниз под действием гравитации. Пока у неё нет столкновений с дорогой — это нормально.

### 4.2. Поворот точки вокруг центра кузова

Добавь два метода в класс `Car` (они есть в готовой версии):

```python
    def rotate(self, x, y):
        c, s = math.cos(self.angle), math.sin(self.angle)
        return x * c - y * s, x * s + y * c

    def point(self, local_x, local_y):
        dx, dy = self.rotate(local_x, local_y)
        return self.x + dx, self.y + dy
```

`rotate` поворачивает локальное смещение, а `point` возвращает **мировые координаты** точки автомобиля. К примеру, `self.point(44, 15)` — точка переднего колеса.

Угол в Pygame работает так: `self.angle > 0` вращает нос автомобиля по часовой стрелке, то есть **вниз**, если машина смотрит вправо. `self.angle < 0` поднимает нос **вверх**.

### 4.3. Отрисовка автомобиля

Добавь в `Car` простой вариант метода `draw`. Пока кузов будет прямоугольным:

```python
    def draw(self, surface, camera_x, camera_y=0):
        def p(x, y):
            wx, wy = self.point(x, y)
            return (int(wx - camera_x), int(wy - camera_y))

        body = [(-60, -20), (60, -20), (60, 10), (-60, 10)]
        pygame.draw.polygon(surface, (224, 90, 54), [p(*v) for v in body])
        for lx in (-44, 44):
            pygame.draw.circle(surface, (35, 35, 40), p(lx, 15), 19)
```

В `Game.draw()` замени тестовый прямоугольник на:

```python
self.car.draw(self.screen, self.camera_x, self.camera_y)
```

В готовом `src/car.py` этот же метод рисует более подробный автомобиль, но принцип координат тот же.

---

## 5. Колёса, наклон поверхности и подвеска

Одного простого `self.y = terrain.height(self.x)` недостаточно: машина не сможет прыгать или опираться на склон двумя колёсами. Поэтому вычислим силы **отдельно в двух точках кузова**.

### 5.1. Параметры колёс и подвески

Допиши в `src/settings.py`:

```python
SPRING = 27000.0
DAMPING = 3200.0
WHEEL_RADIUS = 19
WHEEL_BASE = 88
```

И добавь эти имена к импорту из `.settings` в `car.py`. Центры колёс находятся в локальных точках `(-WHEEL_BASE / 2, 15)` и `(WHEEL_BASE / 2, 15)`.

### 5.2. Угол поверхности

Добавь в класс `Terrain` (в `src/terrain.py`) два метода:

```python
    def slope(self, x):
        return (self.height(x + 4) - self.height(x - 4)) / 8

    def normal(self, x):
        s = self.slope(x)
        length = math.hypot(s, 1)
        return (-s / length, -1 / length)
```

**Нормаль** — вектор, направленный от поверхности вверх. На ровной дороге он равен `(0, -1)`. `math.hypot` вычисляет длину вектора, чтобы нормаль имела длину 1.

### 5.3. Сила пружины и поворот от колёс

Вместо простого падения из раздела 4.1 теперь используем полный расчёт сил. Следующий блок — **замена метода `Car.update()` целиком**, а не отдельные строки для вставки в случайное место.

Сначала допиши в `src/settings.py`:

```python
ENGINE_FORCE = 260000.0
BRAKE_FORCE = 190000.0
AIR_TORQUE = 335000.0
MAX_SPEED = 740.0
```

В `src/car.py` удобно импортировать все параметры явно:

```python
from .settings import (
    GRAVITY, MASS, INERTIA, SPRING, DAMPING,
    ENGINE_FORCE, BRAKE_FORCE, AIR_TORQUE,
    MAX_SPEED, WHEEL_RADIUS, WHEEL_BASE,
)
```

А теперь сам метод:

```python
    def update(self, dt, terrain, gas, brake):
        if self.crashed:
            return

        # Трение воздуха, гравитация и сопротивление вращению.
        force_x = -self.vx * 60
        force_y = MASS * GRAVITY - self.vy * 13
        torque = -self.angular_velocity * 12500
        contacts = 0

        for lx in (-WHEEL_BASE / 2, WHEEL_BASE / 2):
            # Положение каждого колеса с учетом наклона кузова.
            rx, ry = self.rotate(lx, 15)
            wx, wy = self.x + rx, self.y + ry
            nx, ny = terrain.normal(wx)

            # Колесо ниже поверхности? Значит, пружина сжата.
            penetration = (
                (wy + WHEEL_RADIUS - terrain.height(wx))
                / max(0.4, -ny)
            )
            if penetration <= 0:
                continue

            contacts += 1
            # Скорость точки крепления колеса — с учетом вращения.
            point_vx = self.vx - self.angular_velocity * ry
            point_vy = self.vy + self.angular_velocity * rx
            normal_speed = point_vx * nx + point_vy * ny

            spring_force = max(
                0,
                SPRING * min(penetration, 65) - DAMPING * normal_speed,
            )
            fx, fy = nx * spring_force, ny * spring_force

            # Сила двигателя направлена по касательной к склону.
            tangent_x, tangent_y = -ny, nx
            tangential_speed = point_vx * tangent_x + point_vy * tangent_y
            drive = (int(gas) * ENGINE_FORCE - int(brake) * BRAKE_FORCE) / 2
            drive -= tangential_speed * (95 if not (gas or brake) else 30)
            fx += tangent_x * drive
            fy += tangent_y * drive

            force_x += fx
            force_y += fy
            torque += rx * fy - ry * fx

        self.on_ground = contacts > 0
        if self.on_ground:
            self.air_time = 0
        else:
            self.air_time += dt
            # В воздухе: газ = назад, тормоз = вперед.
            torque += (int(brake) - int(gas)) * AIR_TORQUE

        # Обновляем скорости и положение машины.
        self.vx += force_x / MASS * dt
        self.vy += force_y / MASS * dt
        self.angular_velocity += (
            torque / (INERTIA * (4.5 if self.on_ground else 1)) * dt
        )
        self.angular_velocity = max(-4.7, min(4.7, self.angular_velocity))
        self.vx = max(-300, min(MAX_SPEED, self.vx))
        self.vy = max(-950, min(1200, self.vy))
        self.x += self.vx * dt
        self.y += self.vy * dt
        self.angle += self.angular_velocity * dt

        # Проверяем столкновение крыши с землей.
        for lx, ly in ((-34, -26), (20, -38), (47, -18)):
            px, py = self.point(lx, ly)
            if py > terrain.height(px) - 2 and math.cos(self.angle) < 0.0:
                self.crashed = True
        if self.x < 0:
            self.x, self.vx = 0, max(0, self.vx)
        if self.y > 1300:
            self.crashed = True
```

Тут **не появляется ни одной отдельной переменной `vx` или `car_x`**: все состояния принадлежат `Car` и называются `self.vx`, `self.x` и так далее.

**Что проверить:** на ровной земле машина опирается сразу на два колеса; на подъёме переднее и заднее колёса передают кузову разные силы. Измени `DAMPING` на `0`, посмотри на раскачку, затем верни `3200`.

---

## 6. Газ, тормоз и перевороты в воздухе

Механика уже включена в `Car.update()` из предыдущего раздела. Теперь подключаем кнопки в `Game.update()`.

Заменяй прежний временный вариант метода `Game.update()` следующим:

```python
    def update(self, dt, keys):
        gas = keys[pygame.K_RIGHT] or keys[pygame.K_d] or keys[pygame.K_UP]
        brake = keys[pygame.K_LEFT] or keys[pygame.K_a] or keys[pygame.K_DOWN]

        # Четыре маленьких шага физики за один кадр.
        for _ in range(4):
            self.car.update(dt / 4, self.terrain, gas, brake)
            if self.car.crashed:
                break

        desired_camera_x = max(0, self.car.x - WIDTH * 0.30)
        self.camera_x += (desired_camera_x - self.camera_x) * min(1, dt * 5)
```

**На земле:** двигатель создаёт силу вдоль склона (`tangent_x`, `tangent_y`). Благодаря этому машина разгоняется по холмам. **В воздухе:** колёса не касаются трассы, поэтому тяги вперёд нет, зато добавляется момент вращения:

```python
torque += (int(brake) - int(gas)) * AIR_TORQUE
```

- Газ (`→`, `D`, `↑`) — отрицательный момент, нос машины идёт **вверх**, переворот **назад**.
- Тормоз (`←`, `A`, `↓`) — положительный момент, нос машины идёт **вниз**, переворот **вперёд**.

`torque` не меняет угол напрямую: он сначала меняет `self.angular_velocity`, а скорость вращения — `self.angle`. Благодаря этому у автомобиля есть инерция.

**Проверка:** разгонись до холма. В полёте зажми газ, потом повтори прыжок с тормозом. Нос должен поворачиваться в противоположные стороны. Не вставляй дополнительное `self.angle += ...` для кнопок — в готовом проекте этого нет.

---

## 7. Усложняем трассу с расстоянием

В разделе 2 у `Terrain.height()` была одна синусоида. Теперь заменим её процедурным рельефом из готового проекта. Всё относится к **классу `Terrain` в `src/terrain.py`**, не к `Game` и не к `Car`.

Добавь импорты `math`, `random` и `HEIGHT` из настроек:

```python
import math
import random
from .settings import HEIGHT
```

Добавь в `Terrain` конструктор и сложность:

```python
    def __init__(self):
        self.seed = 7319
        rng = random.Random(self.seed)
        self.waves = [
            (rng.uniform(.003, .008), rng.uniform(0, 6.28)),
            (rng.uniform(.011, .020), rng.uniform(0, 6.28)),
            (rng.uniform(.025, .038), rng.uniform(0, 6.28)),
        ]
        self.bumps = []
        for i in range(1, 260):
            x = i * 340 + rng.randint(-90, 90)
            self.bumps.append((
                x, rng.uniform(45, 110),
                rng.choice([-1, 1]) * rng.uniform(12, 75),
            ))

    def difficulty(self, x):
        return min(6, max(0, int(max(x, 0) / 5000)))
```

**Важный момент:** `rng = random.Random(self.seed)` гарантирует одинаковый рельеф при каждом запуске. Если генерировать новые случайные холмы прямо внутри `height`, земля будет меняться при каждом обращении — так нельзя.

Полностью замени `Terrain.height()`:

```python
    def height(self, x):
        if x < 0:
            return 565.0

        ramp = min(1.0, max(0.0, (x - 260) / 3400))
        d = self.difficulty(x)
        amplitude = ramp * (24 + min(d, 6) * 12)
        y = 545.0

        for (frequency, phase), factor in zip(self.waves, (1.3, .60, .28)):
            y += math.sin(x * frequency + phase) * amplitude * factor

        for cx, width, bump_h in self.bumps:
            if abs(x - cx) < width * 3:
                y += (
                    bump_h * ramp * (0.32 + d * .15)
                    * math.exp(-((x - cx) / width) ** 2 * 1.5)
                )

        return min(HEIGHT - 95, max(325, y))
```

`ramp` — плавное появление холмов после стартовой площадки. `d` увеличивается каждые 5000 пикселей. Три синусоиды образуют большие, средние и мелкие неровности. `bumps` добавляет отдельные бугры и впадины.

Методы `slope()` и `normal()` из раздела 5 **оставь без изменений**: они автоматически начнут работать с новым `height()`.

**Задание:** доедь до следующих уровней и сравни первые холмы с более поздними. Попробуй изменить `24 + min(d, 6) * 12` на `24 + min(d, 6) * 18` и посмотри, станет ли трасса проходимой.

---

## 8. Стабилизируем физику и улучшаем камеру

Четыре подшага уже есть в `Game.update()` из раздела 6:

```python
for _ in range(4):
    self.car.update(dt / 4, self.terrain, gas, brake)
```

Это важно, потому что пружины создают большие силы. Если один кадр занимает 1/60 секунды, каждый подшаг длится около 1/240 секунды. Так подвеска ведёт себя стабильнее.

В `Car.update()` мы также ограничили `self.vx`, `self.vy` и `self.angular_velocity`: машина не должна бесконечно ускоряться из-за сильного удара.

Для прыжков добавим вертикальное слежение камеры. В конец `Game.update()`:

```python
desired_camera_y = max(-150, min(180, self.car.y - 415))
self.camera_y += (desired_camera_y - self.camera_y) * min(1, dt * 2)
```

Чтобы камера работала и для трассы, измени `Terrain.draw()`, добавив необязательный аргумент `camera_y=0` и вычитая его из координат всех нарисованных точек. В готовой версии это уже реализовано. Затем вызывай:

```python
self.terrain.draw(self.screen, self.camera_x, self.camera_y)
self.car.draw(self.screen, self.camera_x, self.camera_y)
```

**Проверь:** когда машина высоко подпрыгивает, камера немного следует за ней вверх. Когда приземляется, возвращается плавно.

---

## 9. Монеты, топливо и окончание заезда

Эти данные относятся **к `Game`, не к `Car`**: автомобиль отвечает за движение, а игра — за количество монет, топливо и проигрыш.

### 9.1. Состояние заезда

В готовой версии `Game.__init__()` в конце вызывает `self.reset()`, а метод `reset()` создаёт всё для нового заезда. Когда будешь переносить архитектуру, замени первоначальную инициализацию трассы, машины и камер на вызов `self.reset()`.

```python
    def reset(self):
        self.terrain = Terrain()
        self.car = Car(105, self.terrain)
        self.camera_x = 0.0
        self.camera_y = 0.0
        self.coins = 0
        self.fuel = 100.0
        self.best_x = 105.0
        self.game_over = False
        self.paused = False
        self.pickups = []
        self.next_pickup_x = 650
        self.spawn_pickups(2500)
```

`best_x` — максимальная достигнутая координата, чтобы расстояние не уменьшалось, если автомобиль едет назад.

### 9.2. Генерация предметов

Добавь в `Game`:

```python
    def spawn_pickups(self, right_edge):
        while self.next_pickup_x < right_edge:
            x = self.next_pickup_x
            kind = ('fuel' if int(x // 1150) > int((x - 300) // 1150)
                    else 'coin')
            offset = 85 if kind == 'fuel' else 68
            self.pickups.append({
                'x': x,
                'y': self.terrain.height(x) - offset,
                'type': kind,
                'taken': False,
            })
            self.next_pickup_x += 280
```

Это список словарей. Каждый словарь хранит мировые координаты предмета, его тип и флаг `taken`.

### 9.3. Подбор предметов и расход бензина

Добавь в `Game.update()` **после цикла физики**, но перед обновлением камеры:

```python
self.best_x = max(self.best_x, self.car.x)
self.fuel -= dt * (0.47 + (0.55 if gas else 0))
if self.fuel <= 0:
    self.fuel = 0
    self.game_over = True

self.spawn_pickups(self.car.x + WIDTH * 2)
for item in self.pickups:
    if item['taken']:
        continue
    if math.hypot(self.car.x - item['x'], self.car.y - item['y']) < 62:
        item['taken'] = True
        if item['type'] == 'fuel':
            self.fuel = min(100, self.fuel + 42)
        else:
            self.coins += 1
self.pickups = [
    p for p in self.pickups
    if not p['taken'] and p['x'] > self.car.x - 2000
]
```

Не забудь `import math` вверху `game.py`. В самом начале `Game.update()` добавь:

```python
if self.game_over or self.paused:
    return
```

А внутри цикла физики при `self.car.crashed` выставляй `self.game_over = True` и прерывай цикл. Код полного `Game.update()` можно сверить с готовым файлом `src/game.py`.

### 9.4. Перезапуск и пауза

В методе `Game.run()`, внутри перебора `pygame.event.get()`, добавь обработку `KEYDOWN`:

```python
if event.type == pygame.KEYDOWN:
    if event.key in (pygame.K_r, pygame.K_SPACE):
        self.reset()
    elif event.key == pygame.K_p and not self.game_over:
        self.paused = not self.paused
    elif event.key == pygame.K_ESCAPE:
        running = False
```

**Проверка:** канистры должны восстанавливать запас топлива, монеты увеличивать счёт, а `R` начинать новый заезд.

---

## 10. Отрисовка предметов и интерфейса

В `Game` создай отдельные методы `draw_pickups()` и `draw_hud()`. Эти методы **рисуют**, но не изменяют физику и не собирают предметы.

### 10.1. Предметы

Для каждого несобранного предмета переводим его координаты из мировых в экранные:

```python
x = int(item['x'] - self.camera_x)
y = int(item['y'] - self.camera_y)
```

Монету удобно нарисовать несколькими окружностями, канистру — прямоугольниками. Например, внутри `Game.draw_pickups()`:

```python
    def draw_pickups(self):
        for item in self.pickups:
            if item['taken']:
                continue
            x = int(item['x'] - self.camera_x)
            y = int(item['y'] - self.camera_y)
            if not (-40 < x < WIDTH + 40):
                continue
            if item['type'] == 'coin':
                pygame.draw.circle(self.screen, (229, 153, 33), (x, y), 18)
                pygame.draw.circle(self.screen, (255, 219, 83), (x, y), 14)
            else:
                pygame.draw.rect(self.screen, (203, 58, 53),
                                 (x - 17, y - 19, 34, 39), border_radius=4)
```

### 10.2. Счётчики

В `Game.__init__()` до вызова `self.reset()` создай шрифт:

```python
self.font = pygame.font.SysFont('arial', 25, bold=True)
```

Простейший `Game.draw_hud()`:

```python
    def draw_hud(self):
        distance = max(0, int((self.best_x - 105) / 12))
        lines = [
            f'Расстояние: {distance} м',
            f'Монеты: {self.coins}',
            f'Топливо: {int(self.fuel)}%',
            f'Участок: {self.terrain.difficulty(self.best_x) + 1} / 7',
        ]
        for index, line in enumerate(lines):
            image = self.font.render(line, True, (30, 50, 60))
            self.screen.blit(image, (20, 20 + index * 32))
```

В готовой версии интерфейс красочнее: там есть шкала топлива, затенение экрана при аварии, облака и параллакс. Основной принцип при этом не меняется.

Обнови `Game.draw()`:

```python
    def draw(self):
        self.screen.fill((141, 208, 232))
        self.terrain.draw(self.screen, self.camera_x, self.camera_y)
        self.draw_pickups()
        self.car.draw(self.screen, self.camera_x, self.camera_y)
        self.draw_hud()
        pygame.display.flip()
```

**Задание:** сделай полосу топлива: сначала нарисуй серую подложку, затем зелёную полосу шириной, пропорциональной `self.fuel / 100`.

---

## 11. Сверяем архитектуру с готовой игрой

К этому моменту у тебя уже есть основные механики. Итоговые роли файлов должны совпадать с готовым проектом:

| Файл | Что в нём находится |
|---|---|
| `main.py` | Создание `Game` и запуск `run()` |
| `src/settings.py` | Параметры физики, размеры окна и FPS |
| `src/terrain.py` | `Terrain`: `height()`, `slope()`, `normal()`, `difficulty()`, `draw()` |
| `src/car.py` | `Car`: координаты и скорости, `rotate()`, `point()`, `update()`, `draw()` |
| `src/game.py` | `Game`: состояние заезда, камера, предметы, интерфейс и игровой цикл |

Не путай два метода `update`:

- `Game.update(dt, keys)` **читает клавиатуру, обновляет игровые данные и вызывает физику**.
- `Car.update(dt, terrain, gas, brake)` **рассчитывает силы, скорости, положение и столкновения автомобиля**.

В финальном проекте `main.py` остаётся коротким:

```python
from src.game import Game

if __name__ == '__main__':
    Game().run()
```

В разделе 1 мы временно писали игровой цикл прямо в `main.py`, но с раздела 3 перенесли его в `Game.run()`. **Не оставляй одновременно два игровых цикла.**

### Чек-лист готовности

- [ ] `Game` создаёт `self.terrain`, `self.car` и `self.camera_x`.
- [ ] У машины есть `self.x`, `self.y`, `self.vx`, `self.vy`, `self.angle` и `self.angular_velocity`.
- [ ] Газ и тормоз действуют по склону, а не просто по горизонтали.
- [ ] В воздухе газ вращает машину **назад**, тормоз — **вперёд**.
- [ ] Два колеса взаимодействуют с рельефом независимо.
- [ ] Камера двигается за машиной; трасса становится сложнее вдали от старта.
- [ ] Монеты, топливо, проигрыш, перезапуск и пауза работают.

### Дополнительные задания

**Легко:** сделай индикатор скорости автомобиля: `abs(self.car.vx) / 12 * 3.6` км/ч (приближённо, при 12 пикселях на метр).

**Средне:** добавь разные автомобили, меняя `MASS`, `INERTIA` и `ENGINE_FORCE`.

**Сложнее:** добавь награду за переворот на 360° в воздухе, но выдавай её лишь после удачного приземления.

**Эксперимент:** уменьши `INERTIA` и посмотри, насколько быстрее автомобиль вращается. Затем увеличь `DAMPING` и проверь, насколько спокойнее становится подвеска.
