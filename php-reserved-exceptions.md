# Предопределённые исключения в PHP

> На основе [php.net/manual/ru/reserved.exceptions.php](https://www.php.net/manual/ru/reserved.exceptions.php). Примеры на PHP 8.x.

PHP предоставляет набор **встроенных классов исключений и ошибок**, составляющих основу иерархии `Throwable`. Знание этой иерархии нужно, чтобы правильно ловить и бросать исключения. Дополнительные классы (`LogicException`, `RuntimeException` и их потомки) определены в расширении **SPL** — см. отдельный раздел.

## Содержание

- [Интерфейс Throwable](#интерфейс-throwable)
- [Класс Exception](#класс-exception)
- [Класс Error](#класс-error)
- [Потомки Error](#потомки-error)
- [ErrorException](#errorexception)
- [Иерархия целиком](#иерархия-целиком)
- [Методы исключений](#методы-исключений)
- [Создание и бросание исключений](#создание-и-бросание-исключений)
- [Цепочка исключений (previous)](#цепочка-исключений-previous)
- [Кастомные исключения](#кастомные-исключения)

---

## Интерфейс Throwable

Базовый интерфейс для всего, что можно бросить через `throw` и поймать через `catch`. Его реализуют **оба** корневых класса — `Exception` и `Error`.

```php
<?php
interface Throwable extends Stringable
{
    public function getMessage(): string;
    public function getCode(): int;
    public function getFile(): string;
    public function getLine(): int;
    public function getTrace(): array;
    public function getTraceAsString(): string;
    public function getPrevious(): ?Throwable;
    public function __toString(): string;
}
```

> Нельзя реализовать `Throwable` напрямую в пользовательском классе — нужно наследоваться от `Exception` или `Error`.

```php
<?php
function handle(\Throwable $t): void {   // ловит и Exception, и Error
    error_log($t->getMessage());
}
```

---

## Класс Exception

`Exception` — базовый класс для всех **пользовательских и большинства прикладных** исключений.

```php
<?php
class Exception implements Throwable
{
    public function __construct(
        string $message = '',
        int $code = 0,
        ?Throwable $previous = null    // для цепочки исключений
    ) {}
    // + методы из Throwable
}
```

```php
<?php
throw new Exception('Что-то пошло не так', 42);
```

---

## Класс Error

`Error` — базовый класс для **внутренних ошибок PHP-движка** (появился в PHP 7). Обычно сигнализирует о баге в коде, а не о штатной ситуации.

```php
<?php
// Раньше многие из них были фатальными ошибками; теперь — перехватываемы:
try {
    undefinedFunction();          // Error
} catch (\Error $e) {
    echo 'Поймали ошибку движка: ' . $e->getMessage();
}
```

> `Error` и `Exception` — **параллельные** ветви, обе реализуют `Throwable`, но `Error` **не** наследуется от `Exception`. Поэтому `catch (\Exception)` не поймает `TypeError` — нужен `catch (\Error)` или `catch (\Throwable)`.

---

## Потомки Error

| Класс | Когда бросается |
|-------|-----------------|
| **`TypeError`** | несоответствие типа аргумента, свойства или возвращаемого значения |
| **`ValueError`** | аргумент верного типа, но с недопустимым значением (PHP 8.0+) |
| **`ArgumentCountError`** | передано слишком мало аргументов в функцию |
| **`ArithmeticError`** | ошибка при математической операции (напр. сдвиг на отрицательное число) |
| **`DivisionByZeroError`** | деление на ноль (`intdiv`, `%`, `/` в 8.0+) — потомок `ArithmeticError` |
| **`AssertionError`** | проваленный `assert()` |
| **`UnhandledMatchError`** | `match` не нашёл совпадения и нет `default` (PHP 8.0+) |
| **`FiberError`** | неверная операция с Fiber (PHP 8.1+) |
| **`ParseError`** | синтаксическая ошибка (напр. в `eval()`) — потомок `CompileError` |
| **`CompileError`** | ошибка компиляции |

```php
<?php
declare(strict_types=1);

// TypeError
try { strlen([]); } catch (\TypeError $e) { echo 'TypeError'; }

// ValueError (тип верный, значение недопустимо)
try { array_chunk([1,2], -1); } catch (\ValueError $e) { echo 'ValueError'; }

// DivisionByZeroError
try { $x = 1 % 0; } catch (\DivisionByZeroError $e) { echo 'Деление на ноль'; }

// UnhandledMatchError
try {
    echo match (5) { 1 => 'a', 2 => 'b' };
} catch (\UnhandledMatchError $e) { echo 'Нет совпадения в match'; }

// ArgumentCountError
function need(int $a, int $b) {}
try { need(1); } catch (\ArgumentCountError $e) { echo 'Мало аргументов'; }
```

---

## ErrorException

Особый класс: позволяет превратить **классическую ошибку** PHP (`E_WARNING`, `E_NOTICE` и т.п.) в перехватываемое исключение. Несмотря на имя, наследуется от `Exception`, а не от `Error`.

```php
<?php
// Типичный приём: обработчик ошибок бросает ErrorException
set_error_handler(function (int $severity, string $message, string $file, int $line): bool {
    throw new \ErrorException($message, 0, $severity, $file, $line);
});

try {
    $x = $undefined;   // обычно Warning → теперь ErrorException
} catch (\ErrorException $e) {
    echo 'Поймали как исключение: ' . $e->getMessage();
}
```

---

## Иерархия целиком

```
Throwable (interface)
│
├── Error                          ← ошибки движка PHP
│   ├── ArgumentCountError
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   ├── CompileError
│   │   └── ParseError
│   ├── FiberError
│   ├── TypeError
│   ├── UnhandledMatchError
│   └── ValueError
│
└── Exception                      ← прикладные исключения
    ├── ErrorException
    └── (из SPL):
        ├── LogicException         ← ошибки логики (видны при разработке)
        │   ├── BadFunctionCallException
        │   │   └── BadMethodCallException
        │   ├── DomainException
        │   ├── InvalidArgumentException
        │   ├── LengthException
        │   └── OutOfRangeException
        └── RuntimeException       ← ошибки времени выполнения
            ├── OutOfBoundsException
            ├── OverflowException
            ├── RangeException
            ├── UnderflowException
            └── UnexpectedValueException
```

> **LogicException** — ошибки, которые должны выявляться при разработке (неверное использование API). **RuntimeException** — ошибки, возникающие только во время выполнения (недоступный файл, неверные данные). Это соглашение SPL помогает выбирать правильный класс.

---

## Методы исключений

```php
<?php
try {
    throw new RuntimeException('Ошибка', 500);
} catch (RuntimeException $e) {
    echo $e->getMessage();        // 'Ошибка'
    echo $e->getCode();           // 500
    echo $e->getFile();           // путь к файлу
    echo $e->getLine();           // номер строки
    echo $e->getTraceAsString();  // стек вызовов строкой
    print_r($e->getTrace());      // стек массивом
    $prev = $e->getPrevious();    // предыдущее исключение или null
    echo (string) $e;             // __toString — полное описание
}
```

---

## Создание и бросание исключений

```php
<?php
function divide(int $a, int $b): float
{
    if ($b === 0) {
        throw new InvalidArgumentException('Делитель не может быть нулём');
    }
    return $a / $b;
}

// throw как выражение (PHP 8.0+) — можно в ??, тернарнике, стрелочной функции
$value = $config['key'] ?? throw new RuntimeException('Нет ключа');
$fn = fn($x) => $x > 0 ? $x : throw new ValueError('Неположительное');
```

Выбор класса:
- неверный аргумент → `InvalidArgumentException`;
- значение вне диапазона → `OutOfRangeException` / `RangeException`;
- недоступный ресурс/файл → `RuntimeException`;
- вызов несуществующего метода → `BadMethodCallException`.

---

## Цепочка исключений (previous)

Третий аргумент конструктора — «предыдущее» исключение. Позволяет оборачивать низкоуровневые ошибки, не теряя контекст.

```php
<?php
try {
    try {
        throw new PDOException('Сбой соединения с БД');
    } catch (PDOException $e) {
        // оборачиваем, сохраняя исходное через $previous
        throw new RuntimeException('Не удалось загрузить пользователя', 0, $e);
    }
} catch (RuntimeException $e) {
    echo $e->getMessage();              // Не удалось загрузить пользователя
    echo $e->getPrevious()->getMessage(); // Сбой соединения с БД
}
```

---

## Кастомные исключения

Свои исключения наследуют от подходящего базового класса.

```php
<?php
// Базовое исключение домена
class UserException extends RuntimeException {}

// Конкретные
class UserNotFoundException extends UserException
{
    public function __construct(
        public readonly int $userId,    // дополнительные данные
        ?Throwable $previous = null,
    ) {
        parent::__construct("Пользователь #$userId не найден", 404, $previous);
    }
}

try {
    throw new UserNotFoundException(42);
} catch (UserException $e) {            // ловим по базовому классу
    echo $e->getMessage();              // Пользователь #42 не найден
    echo $e->userId;                    // 42
}
```

> Создавайте иерархию своих исключений с общим базовым классом — это позволяет ловить целые группы ошибок одним `catch`.

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое `Throwable`? | Общий интерфейс для `Error` и `Exception`. |
| `Error` наследуется от `Exception`? | Нет — это параллельные ветви, обе реализуют `Throwable`. |
| Поймает ли `catch (\Exception)` `TypeError`? | Нет — нужен `catch (\Error)` или `catch (\Throwable)`. |
| Когда бросается `TypeError`? | При несоответствии типа аргумента/возврата. |
| `TypeError` vs `ValueError`? | Тип неверный vs тип верный, но недопустимое значение. |
| Что такое `ErrorException`? | Обёртка классической ошибки PHP в исключение (наследует `Exception`). |
| `LogicException` vs `RuntimeException`? | Первое — ошибки в коде (при разработке); второе — ошибки времени выполнения. |
| Какой класс для неверного аргумента? | `InvalidArgumentException` (потомок `LogicException`). |
| Зачем `getPrevious()`? | Доступ к обёрнутому исходному исключению (цепочка). |
| Можно ли `throw` как выражение? | Да, с PHP 8.0 — в `??`, тернарнике, `fn`. |
| Откуда `RuntimeException`, `LogicException`? | Из расширения SPL. |

---

*Источник: [php.net/manual/ru/reserved.exceptions.php](https://www.php.net/manual/ru/reserved.exceptions.php). Примеры на PHP 8.x. См. также `php-errors.md`, `php-oop.md`, `php-fibers.md`.*
