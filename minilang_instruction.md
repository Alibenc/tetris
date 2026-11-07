# Практическая работа: Создаём собственный язык программирования

**Сложность:** 5/10  
**Время:** 1–2 занятия  
**Результат:** собственный язык с переменными, математикой, выводом, вводом, условиями и циклами.

## Шаг 1. Создаём проект

Создай папку `minilang` и два файла:

```text
minilang/
├── main.py
└── program.txt
```

В `main.py` будет интерпретатор, который понимает и выполняет команды языка. В `program.txt` — программы на этом языке.

Например:

```text
SET x 10
ADD x 5
PRINT x
```

После запуска `main.py` должно появиться:

```text
15
```

## Шаг 2. Читаем программу из файла

В `main.py` напиши:

```python
with open("program.txt", "r", encoding="utf-8") as file:
    lines = file.readlines()

print(lines)
```

Обрати внимание на символы переноса строки `\n`. Уберём их:

```python
with open("program.txt", "r", encoding="utf-8") as file:
    lines = [line.strip() for line in file.readlines()]
```

Теперь `lines` содержит список строк программы.

## Шаг 3. Распознаём команды

Строку `SET x 10` можно разделить на части:

```python
line = "SET x 10"
parts = line.split()
print(parts)
```

Результат:

```python
["SET", "x", "10"]
```

- `parts[0]` — команда.
- `parts[1]` — первый аргумент.
- `parts[2]` — второй аргумент.

Пройди по всем строкам файла:

```python
for line in lines:
    if not line:
        continue

    parts = line.split()
    command = parts[0]
    print("Команда:", command)
```

Пока интерпретатор лишь распознаёт инструкции.

## Шаг 4. Добавляем переменные

Хранить переменные удобно в словаре:

```python
variables = {}
```

Например:

```python
variables["x"] = 10
variables["score"] = 100
```

Добавь обработчик `SET` внутрь цикла:

```python
if command == "SET":
    name = parts[1]
    value = int(parts[2])
    variables[name] = value
```

Проверь файл:

```text
SET x 10
SET y 25
```

После обработки выведи словарь:

```python
print(variables)
```

Ожидаемый результат:

```python
{"x": 10, "y": 25}
```

## Шаг 5. Реализуем PRINT

Команда `PRINT` выводит значение переменной:

```text
SET score 100
PRINT score
```

Добавь обработчик после `if command == "SET"`:

```python
elif command == "PRINT":
    name = parts[1]
    print(variables[name])
```

Попробуй `PRINT money`, не создавая `money`. Возникнет `KeyError` — позднее добавим собственные сообщения об ошибках.

## Шаг 6. Добавляем арифметику

Команда `ADD`:

```text
SET x 10
ADD x 5
PRINT x
```

Обработчик:

```python
elif command == "ADD":
    name = parts[1]
    value = int(parts[2])
    variables[name] += value
```

Самостоятельно добавь:

```text
SUB x 5
MUL x 3
DIV x 2
```

В первой версии `DIV` выполняет целочисленное деление.

Проверка:

```text
SET money 100
ADD money 50
MUL money 2
SUB money 40
DIV money 2
PRINT money
```

Результат:

```text
130
```

## Шаг 7. Используем переменные в операциях

Пока `ADD x 5` работает, а `ADD x y` — нет. Добавим функцию, которая возвращает либо число, либо значение переменной.

Размести её после объявления `variables = {}` и до цикла выполнения команд:

```python
def get_value(value):
    if value.lstrip("-").isdigit():
        return int(value)

    return variables[value]
```

Теперь `get_value("10")` возвращает 10, а `get_value("x")` — значение `x`.

Во всех арифметических обработчиках замени:

```python
value = int(parts[2])
```

на:

```python
value = get_value(parts[2])
```

Проверка:

```text
SET a 10
SET b 25
ADD a b
PRINT a
```

Результат: `35`.

**Дополнительно:** попробуй использовать `get_value` и в `SET`, чтобы поддержать `SET b a`.

## Шаг 8. Пользовательский ввод

Добавь команду `INPUT`:

```text
INPUT age
PRINT age
```

Обработчик:

```python
elif command == "INPUT":
    name = parts[1]
    variables[name] = int(input(f"{name}: "))
```

Проверка:

```text
INPUT a
INPUT b
ADD a b
PRINT a
```

Программа должна запросить два числа и вывести сумму.

## Шаг 9. Реализуем GOTO

Чтобы переходить между строками, заменим `for` на `while` с индексом текущей строки.

```python
current_line = 0

while current_line < len(lines):
    line = lines[current_line]

    if not line:
        current_line += 1
        continue

    parts = line.split()
    command = parts[0]

    # Здесь расположи уже написанные обработчики SET, PRINT, ADD и остальных команд.

    current_line += 1
```

`current_line` — индекс текущей инструкции. Добавь ещё одну ветку к обработчикам:

```python
elif command == "GOTO":
    current_line = int(parts[1]) - 1
    continue
```

В языке номера строк начинаются с 1, в списке Python — с 0. `continue` не даёт выполнить обычное увеличение индекса после перехода.

Проверка:

```text
SET x 10
GOTO 4
PRINT x
ADD x 5
PRINT x
```

Результат: `15`.

## Шаг 10. Добавляем условия

Синтаксис:

```text
IF x > 10 GOTO 5
```

Для начала реализуй `>`, `<`, `==`:

```python
elif command == "IF":
    left = get_value(parts[1])
    operator = parts[2]
    right = get_value(parts[3])
    target = int(parts[5])

    result = False

    if operator == ">":
        result = left > right
    elif operator == "<":
        result = left < right
    elif operator == "==":
        result = left == right

    if result:
        current_line = target - 1
        continue
```

Самостоятельно добавь `>=`, `<=`, `!=`.

Проверка:

```text
SET age 18
IF age >= 18 GOTO 5
PRINT 0
GOTO 6
PRINT 1
PRINT age
```

Ожидаемый результат:

```text
1
18
```

**Обрати внимание:** сейчас `PRINT` умеет выводить только существующие переменные. Чтобы тест выше работал, доработай обработчик `PRINT` так, чтобы он использовал `get_value(parts[1])`. Тогда он сможет выводить и числа, и переменные.

## Шаг 11. Первый цикл на своём языке

В `program.txt`:

```text
SET x 1
PRINT x
ADD x 1
IF x < 11 GOTO 2
```

Программа должна вывести числа от 1 до 10. Отдельной команды `WHILE` нет, но циклы уже работают через условные переходы.

## Шаг 12. Комментарии

Пусть строки, начинающиеся с `#`, игнорируются:

```text
# Создаём счётчик
SET x 1

# Выводим числа
PRINT x
ADD x 1
IF x < 11 GOTO 5
```

В начале цикла выполнения замени проверку пустой строки:

```python
if not line or line.startswith("#"):
    current_line += 1
    continue
```

Комментарии и пустые строки остаются в `lines`, поэтому номера для `GOTO` считаются с их учётом.

## Шаг 13. Обрабатываем ошибки

Добавь понятные сообщения для случаев:

- Неизвестная команда.
- Несуществующая переменная.
- Деление на ноль.
- Неверное количество аргументов.
- Переход на несуществующую строку.

Например:

```text
Error at line 4: Unknown command 'HELLO'
```

Начни с неизвестной команды, затем добавляй проверки постепенно.

## Шаг 14. Финальное задание

Напиши **на MiniLang**, а не на Python, программу, которая:

1. Запрашивает число `N`.
2. Вычисляет сумму чисел от 1 до `N`.
3. Выводит результат.

Например:

```text
N: 5
15
```

Затем реализуй вычисление факториала. Дополнительные команды в интерпретатор добавлять не требуется.

## Итог

| Возможность | Команда |
|---|---|
| Переменные | `SET` |
| Вывод | `PRINT` |
| Ввод | `INPUT` |
| Сложение | `ADD` |
| Вычитание | `SUB` |
| Умножение | `MUL` |
| Деление | `DIV` |
| Переходы | `GOTO` |
| Условия | `IF` |
| Циклы | Через `IF` и `GOTO` |
| Комментарии | `#` |

**Бонус на следующее занятие:** строки, `PRINT "Hello World"` и метки вместо номеров строк (`GOTO start`).
