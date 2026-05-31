# Ошибки в PHP (Errors)

> На основе [php.net/manual/en/language.errors.php](https://www.php.net/manual/en/language.errors.php). Примеры на PHP 8.x.

Раздел описывает базовую концепцию **ошибок** в PHP и их связь с механизмом исключений. Начиная с PHP 7 большинство фатальных ошибок превратились в объекты-исключения, которые можно перехватывать через `try/catch`.

## Содержание

- [Основы обработки ошибок](#основы-обработки-ошибок)
- [Иерархия Throwable](#иерархия-throwable)
- [Error vs Exception](#error-vs-exception)
- [Уровни ошибок (E_*)](#уровни-ошибок-e_)
- [Перехват ошибок](#перехват-ошибок)
- [Настройка отображения ошибок](#настройка-отображения-ошибок)
- [Оператор подавления @](#оператор-подавления-)
- [Кастомный обработчик ошибок](#кастомный-обработчик-ошибок)
- [Поведение в production и development](#поведение-в-production-и-development)

---

## Основы обработки ошибок

В PHP исторически было два механизма:

1. **Классические ошибки** (`E_WARNING`, `E_NOTICE`, …) — сообщения разных уровней серьёзности, которые по умолчанию выводятся, но не прерывают скрипт (кроме фатальных).
2. **Исключения** (`Exception`) — объектный механизм с `try/catch`.

С **PHP 7** их объединили: фатальные и recoverable-ошибки бросаются как объекты `Error`, и оба типа (`Error` и `Exception`) реализуют общий интерфейс `Throwable`.

```php
<?php
// Раньше это был Fatal Error, останавливающий скрипт.
// Теперь — перехватываемый Error:
try {
    $result = 10 % 0;            // DivisionByZeroError
} catch (\DivisionByZeroError $e) {
    echo 'Поймали: ' . $e->getMessage();
}
```

---

## Иерархия Throwable

Всё, что можно «бросить» (`throw`) и «поймать» (`catch`), реализует интерфейс `Throwable`.

```
Throwable (interface)
├── Error                    — внутренние ошибки PHP-движка
│   ├── TypeError            — несоответствие типа аргумента/возврата
│   ├── ValueError           — недопустимое значение аргумента (8.0+)
│   ├── ArithmeticError
│   │   └── DivisionByZeroError
│   ├── AssertionError
│   └── ...
└── Exception                — ошибки уровня приложения
    ├── ErrorException        — оборачивает классическую ошибку в исключение
    ├── RuntimeException
    │   ├── OutOfBoundsException
    │   └── ...
    ├── LogicException
    │   ├── InvalidArgumentException
    │   └── ...
    └── ...
```

> Нельзя реализовать `Throwable` напрямую в своём классе — нужно наследоваться от `Exception` или `Error`.

```php
<?php
function handle(\Throwable $t): void {   // ловит и Error, и Exception
    echo get_class($t) . ': ' . $t->getMessage();
}
```

---

## Error vs Exception

| | `Error` | `Exception` |
|---|---------|-------------|
| Источник | внутренние ошибки PHP-движка | ошибки логики приложения |
| Примеры | `TypeError`, `ParseError`, `DivisionByZeroError` | `RuntimeException`, `InvalidArgumentException` |
| Когда возникают | нарушение контракта языка | бизнес-условия, валидация |
| Кого ловить | обычно не ловят (это баг кода) | штатно обрабатывают |
| Общий предок | `Throwable` | `Throwable` |

```php
<?php
declare(strict_types=1);

function add(int $a, int $b): int { return $a + $b; }

try {
    add('not', 'numbers');           // TypeError (это Error, не Exception)
} catch (\TypeError $e) {
    echo 'Ошибка типа: ' . $e->getMessage();
}
```

> Практическое правило: `Error` обычно сигнализирует о **баге в коде**, который надо чинить, а не «обрабатывать». `Exception` — ожидаемая нештатная ситуация.

---

## Уровни ошибок (E_*)

Классические ошибки имеют уровни — битовые константы:

| Константа | Значение |
|-----------|----------|
| `E_ERROR` | фатальная ошибка времени выполнения (скрипт останавливается) |
| `E_WARNING` | предупреждение (не останавливает) |
| `E_NOTICE` | уведомление (вероятная ошибка, напр. чтение неопределённой переменной) |
| `E_DEPRECATED` | использование устаревшего функционала |
| `E_PARSE` | синтаксическая ошибка |
| `E_STRICT` | рекомендации по совместимости (устарело) |
| `E_USER_ERROR` / `E_USER_WARNING` / `E_USER_NOTICE` | пользовательские (через `trigger_error()`) |
| `E_ALL` | все ошибки и предупреждения |

```php
<?php
// Сгенерировать собственную ошибку
trigger_error('Что-то не так', E_USER_WARNING);

// В PHP 8.0 многие бывшие Warning/Notice стали более строгими:
// например, обращение к несуществующему ключу массива — Warning,
// а вызов несуществующего метода — Error.
```

---

## Перехват ошибок

```php
<?php
try {
    // код, который может бросить исключение/ошибку
    riskyOperation();
} catch (\InvalidArgumentException $e) {
    // конкретный тип
    echo 'Неверный аргумент';
} catch (\RuntimeException | \LogicException $e) {
    // несколько типов сразу (PHP 7.1+)
    echo 'Runtime или Logic';
} catch (\Throwable $e) {
    // перехват вообще всего (и Error, и Exception)
    echo 'Любая ошибка: ' . $e->getMessage();
} finally {
    // выполняется всегда — для очистки ресурсов
    cleanup();
}

// catch без переменной (PHP 8.0+), если объект не нужен
try { /* ... */ } catch (\Exception) { echo 'упс'; }
```

Полезные методы `Throwable`:

```php
<?php
$e->getMessage();    // текст
$e->getCode();       // код
$e->getFile();       // файл
$e->getLine();       // строка
$e->getTrace();      // стек вызовов (массив)
$e->getTraceAsString();
$e->getPrevious();   // предыдущее исключение (цепочка)
```

---

## Настройка отображения ошибок

Управляется директивами `php.ini` или функциями во время выполнения.

```php
<?php
// Какие уровни СООБЩАТЬ (репортить)
error_reporting(E_ALL);                  // все
error_reporting(E_ALL & ~E_DEPRECATED);  // все, кроме deprecated

// ВЫВОДИТЬ ли ошибки на экран
ini_set('display_errors', '1');          // dev: показывать
ini_set('display_errors', '0');          // prod: скрывать от пользователя

// ЛОГИРОВАТЬ ли в файл
ini_set('log_errors', '1');
ini_set('error_log', '/var/log/php_errors.log');
```

> Ключевое различие: `error_reporting` — *какие* ошибки учитывать; `display_errors` — *показывать* ли их в выводе; `log_errors` — *писать* ли в лог.

---

## Оператор подавления @

`@` подавляет сообщения об ошибках для одного выражения. Использовать **крайне осторожно** — скрывает проблемы.

```php
<?php
$content = @file_get_contents('maybe-missing.txt');   // не выведет Warning
if ($content === false) {
    // обработать отсутствие файла
}
```

> ⚠️ В PHP 8.0 `@` больше **не** подавляет фатальные ошибки. Предпочтительнее проверять условия явно (`file_exists`, `isset`) или ловить исключения, а не глушить через `@`.

---

## Кастомный обработчик ошибок

Можно перехватывать классические ошибки и превращать их в исключения — частый приём.

```php
<?php
// Обработчик ошибок (классические E_*)
set_error_handler(function (int $severity, string $message, string $file, int $line): bool {
    // Превратить ошибку в исключение
    throw new \ErrorException($message, 0, $severity, $file, $line);
});

// Обработчик НЕперехваченных исключений
set_exception_handler(function (\Throwable $e): void {
    error_log($e->getMessage());
    http_response_code(500);
    echo 'Внутренняя ошибка';
});

// Обработчик фатальных ошибок (на завершении скрипта)
register_shutdown_function(function (): void {
    $error = error_get_last();
    if ($error !== null && $error['type'] === E_ERROR) {
        // залогировать фатальную ошибку
    }
});
```

---

## Поведение в production и development

| Настройка | Development | Production |
|-----------|-------------|------------|
| `display_errors` | `On` (видеть сразу) | `Off` (не светить детали пользователю) |
| `error_reporting` | `E_ALL` | `E_ALL` (но не показывать) |
| `log_errors` | `On` | `On` (писать в лог) |
| Stack trace на экране | да | нет (только в логе) |

> В проде показ ошибок и стек-трейсов пользователю — риск безопасности (раскрытие путей, структуры кода). Ошибки логируют, а пользователю показывают нейтральную страницу.

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что изменилось в обработке ошибок в PHP 7? | Многие фатальные ошибки стали объектами `Error`, перехватываемыми через `try/catch`. |
| Что такое `Throwable`? | Общий интерфейс для `Error` и `Exception` — всё, что можно бросить/поймать. |
| `Error` vs `Exception`? | `Error` — ошибки движка (баги кода); `Exception` — ошибки уровня приложения. |
| Можно ли поймать `TypeError`? | Да, это `Error`; ловится через `catch (\TypeError)` или `\Throwable`. |
| Как поймать вообще всё? | `catch (\Throwable $e)`. |
| Зачем `finally`? | Выполняется всегда — для освобождения ресурсов. |
| Разница `error_reporting` и `display_errors`? | Первое — какие ошибки учитывать, второе — выводить ли их. |
| Чем опасен `@`? | Скрывает ошибки; в 8.0 не подавляет фатальные. |
| Как превратить ошибку в исключение? | `set_error_handler()` + `throw new ErrorException(...)`. |
| Настройки ошибок в проде? | `display_errors=Off`, `log_errors=On`. |

---

*Источник: [php.net/manual/en/language.errors.php](https://www.php.net/manual/en/language.errors.php). Примеры на PHP 8.x. Подробнее об исключениях — `language.exceptions.php`.*
