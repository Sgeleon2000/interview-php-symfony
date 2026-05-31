# Пространства имён в PHP (Namespaces)

> На основе [php.net/manual/en/language.namespaces.php](https://www.php.net/manual/en/language.namespaces.php). Примеры на PHP 8.x.

**Пространство имён** — это способ группировки логически связанных классов, интерфейсов, функций и констант, а также защиты от конфликтов имён. Они решают две проблемы: **столкновение имён** (две библиотеки определяют класс `Logger`) и **длинные имена** (возможность давать короткие псевдонимы).

## Содержание

- [Зачем нужны пространства имён](#зачем-нужны-пространства-имён)
- [Объявление namespace](#объявление-namespace)
- [Вложенные (иерархические) пространства](#вложенные-иерархические-пространства)
- [Использование: три вида имён](#использование-три-вида-имён)
- [Ключевое слово namespace и __NAMESPACE__](#ключевое-слово-namespace-и-__namespace__)
- [Импорт и псевдонимы (use)](#импорт-и-псевдонимы-use)
- [Глобальное пространство](#глобальное-пространство)
- [Правила разрешения имён](#правила-разрешения-имён)
- [Namespace и автозагрузка (PSR-4)](#namespace-и-автозагрузка-psr-4)

---

## Зачем нужны пространства имён

Без namespace два класса с одинаковым именем из разных библиотек конфликтуют. Namespace — как «фамилия» для имени: позволяет `App\Logger` и `Vendor\Logger` сосуществовать.

```php
<?php
// Аналогия с файловой системой:
// файл foo.txt может быть в /home/greg и в /home/other —
// полный путь их различает. Namespace — то же для кода.

\App\Service\Logger        // ≈ /App/Service/Logger
```

Пространствами имён можно инкапсулировать **только** классы, интерфейсы, трейты, enum, функции и константы.

---

## Объявление namespace

Объявляется ключевым словом `namespace` — оно **должно быть самой первой инструкцией** в файле (до него допускается только `declare`).

```php
<?php

declare(strict_types=1);     // единственное, что может идти раньше

namespace App\Service;        // объявление пространства

class UserService {}          // полное имя: \App\Service\UserService
function helper() {}          // \App\Service\helper
const VERSION = '1.0';        // \App\Service\VERSION
```

> ⚠️ Любой вывод (даже пробел или BOM) перед `namespace` вызовет фатальную ошибку. Поэтому файлы с классами обычно **не** содержат закрывающего тега `?>`.

Можно объявить несколько пространств в одном файле (не рекомендуется), в т.ч. блоками с фигурными скобками:

```php
<?php
namespace App\First {
    class A {}
}
namespace App\Second {
    class B {}
}
namespace { // глобальный код
    $a = new \App\First\A();
}
```

---

## Вложенные (иерархические) пространства

Пространства образуют иерархию через разделитель `\`.

```php
<?php
namespace App\Repository\Doctrine;

class UserRepository {}   // \App\Repository\Doctrine\UserRepository
```

Это похоже на каталоги: `App` → `Repository` → `Doctrine`. При PSR-4 эта иерархия обычно отражает структуру папок.

---

## Использование: три вида имён

Как и в файловой системе, имена бывают трёх видов:

| Вид имени | Пример | Аналогия с файлами | Разрешается относительно |
|-----------|--------|--------------------|--------------------------|
| **Неполное (unqualified)** | `Logger` | `file.txt` | текущего namespace |
| **Квалифицированное (qualified)** | `Sub\Logger` | `subdir/file.txt` | текущего namespace |
| **Полностью квалифицированное (fully qualified)** | `\App\Sub\Logger` | `/dir/file.txt` | корня (абсолютное) |

```php
<?php
namespace App\Service;

// Неполное имя — ищется в текущем namespace
$x = new Logger();              // → \App\Service\Logger

// Квалифицированное — относительно текущего
$y = new Sub\Logger();          // → \App\Service\Sub\Logger

// Полностью квалифицированное — абсолютный путь от корня
$z = new \Monolog\Logger();     // → именно \Monolog\Logger
```

---

## Ключевое слово namespace и __NAMESPACE__

`namespace\` явно указывает на **текущее** пространство (аналог `self::` для классов). Магическая константа `__NAMESPACE__` содержит имя текущего пространства строкой.

```php
<?php
namespace App\Service;

function foo() { echo 'foo'; }

namespace\foo();                 // явно вызвать \App\Service\foo
echo __NAMESPACE__;              // "App\Service"

// Полезно для динамического построения имён
$class = __NAMESPACE__ . '\\Logger';
$obj = new $class();
```

---

## Импорт и псевдонимы (use)

`use` импортирует имя из другого пространства, чтобы обращаться к нему кратко. Размещается **после** `namespace`, до объявлений.

```php
<?php
namespace App\Controller;

use App\Service\UserService;              // импорт класса
use App\Repository\UserRepository as Repo; // с псевдонимом (as)

use function App\Helpers\format;          // импорт функции (PHP 5.6+)
use const App\Config\TIMEOUT;             // импорт константы (PHP 5.6+)

class HomeController
{
    public function __construct(
        private UserService $service,      // вместо \App\Service\UserService
        private Repo $repo,                // псевдоним
    ) {}
}

format('x');   // вызов импортированной функции
echo TIMEOUT;  // импортированная константа
```

**Групповой импорт** (PHP 7.0+) — несколько имён из одного пространства:

```php
<?php
use App\Models\{User, Post, Comment};
use App\Service\{
    UserService,
    MailService as Mailer,
};
```

> `use` влияет только на текущий файл. Импорт — это псевдоним на этапе компиляции, а не подключение файла (за подключение отвечает автозагрузка).

---

## Глобальное пространство

Встроенные классы и функции PHP (`DateTime`, `Exception`, `strlen`, `Closure`) находятся в **глобальном** пространстве. Внутри namespace к ним обращаются через ведущий `\`.

```php
<?php
namespace App\Service;

// Классы: ОБЯЗАТЕЛЬНО \ — иначе PHP ищет \App\Service\DateTime
$date = new \DateTime();
try {
    // ...
} catch (\Exception $e) {            // \Exception, не App\Service\Exception
}

// Функции и константы: PHP делает fallback в глобальное пространство автоматически
echo strlen('hi');                   // найдёт \strlen, если App\Service\strlen нет
echo \strlen('hi');                  // явный вызов (чуть быстрее, без fallback)
```

> **Важное различие:** для **классов** fallback в глобальное пространство **не** работает — нужен ведущий `\`. Для **функций и констант** fallback есть. Поэтому всегда пишите `new \DateTime()`, а не `new DateTime()` внутри namespace.

---

## Правила разрешения имён

Алгоритм PHP при разрешении имени (упрощённо):

1. **Полностью квалифицированные** имена (`\Foo\Bar`) — всегда абсолютны, без преобразований.
2. **Квалифицированные** (`Sub\Foo`) и **неполные** имена сначала проверяются по таблице `use`-импортов.
3. Если совпадения нет — имя приписывается к текущему namespace.
4. Для **функций/констант** при отсутствии в текущем namespace — fallback в глобальное пространство. Для **классов** fallback нет.

```php
<?php
namespace App;

use App\Service\Mailer;

new Mailer();        // → \App\Service\Mailer (из use)
new Logger();        // → \App\Logger (текущий namespace)
new \Logger();       // → \Logger (глобальный)
strlen('x');         // → \strlen (fallback функции)
```

---

## Namespace и автозагрузка (PSR-4)

На практике пространства имён жёстко связаны с автозагрузкой: структура namespace соответствует структуре каталогов.

```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

- `App\Service\UserService` → `src/Service/UserService.php`
- `App\Controller\HomeController` → `src/Controller/HomeController.php`

```php
<?php
// src/Service/UserService.php
namespace App\Service;        // namespace ДОЛЖЕН совпадать с путём по PSR-4

class UserService {}
```

> Подробно об автозагрузке PSR-4 — см. `psr-standards.md`. Об ООП-возможностях классов — `php-oop.md`.

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Зачем нужны namespace? | Избежать конфликтов имён и давать короткие псевдонимы длинным именам. |
| Где должно стоять объявление `namespace`? | Первой инструкцией файла (раньше — только `declare`). |
| Какие три вида имён? | Неполное, квалифицированное, полностью квалифицированное (с ведущим `\`). |
| Что делает `use`? | Импортирует имя в файл (псевдоним на этапе компиляции), не подключает файл. |
| Как задать псевдоним? | `use App\Long\Name as Short;`. |
| Можно ли импортировать функции/константы? | Да: `use function ...`, `use const ...` (с 5.6). |
| Почему `new DateTime()` в namespace падает? | Для классов нет fallback в глобальное пространство — нужен `\DateTime`. |
| Есть ли fallback для функций? | Да — для функций и констант PHP ищет в глобальном пространстве. |
| Что такое `__NAMESPACE__`? | Магическая константа с именем текущего пространства. |
| Как namespace связан с PSR-4? | Иерархия namespace отражает структуру каталогов для автозагрузки. |

---

*Источник: [php.net/manual/en/language.namespaces.php](https://www.php.net/manual/en/language.namespaces.php). Примеры на PHP 8.x.*
