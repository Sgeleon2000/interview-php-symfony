# Атрибуты в PHP (Attributes)

> На основе [php.net/manual/en/language.attributes.php](https://www.php.net/manual/en/language.attributes.php). Доступны с **PHP 8.0+**.

**Атрибуты** — это структурированные, машиночитаемые метаданные, которые можно добавлять к классам, методам, свойствам, параметрам, константам и функциям. Они заменяют практику хранения метаданных в **PHPDoc-аннотациях** (`@Route`, `@ORM\Column`) настоящим синтаксисом языка, который проверяется на этапе компиляции.

## Содержание

- [Зачем нужны атрибуты](#зачем-нужны-атрибуты)
- [Синтаксис](#синтаксис)
- [Объявление собственного атрибута](#объявление-собственного-атрибута)
- [Цели атрибута (#[Attribute] flags)](#цели-атрибута-attribute-flags)
- [Чтение атрибутов через Reflection](#чтение-атрибутов-через-reflection)
- [Аргументы атрибутов](#аргументы-атрибутов)
- [Атрибуты vs аннотации PHPDoc](#атрибуты-vs-аннотации-phpdoc)
- [Встроенные атрибуты PHP](#встроенные-атрибуты-php)
- [Практический пример](#практический-пример)

---

## Зачем нужны атрибуты

До PHP 8 метаданные писали в комментариях-аннотациях, а фреймворки (Doctrine, Symfony) парсили их строкой:

```php
<?php
/**
 * @Route("/users", methods={"GET"})       ← аннотация в комментарии
 * @ORM\Column(type="string")
 */
```

Проблемы аннотаций: это просто строки в комментариях — нет проверки синтаксиса, нужны сторонние парсеры, опечатки не отлавливаются. **Атрибуты** решают это: они часть языка, проверяются компилятором, читаются через нативный Reflection.

```php
<?php
#[Route('/users', methods: ['GET'])]       // ← настоящий атрибут
public function list(): Response {}
```

---

## Синтаксис

Атрибут пишется в синтаксисе `#[...]` перед объявлением, к которому относится.

```php
<?php
#[Attribute1]
#[Attribute2, Attribute3]                 // несколько в одной группе
#[Attribute4(value: 'x', flag: true)]     // с аргументами
class Example
{
    #[Inject]
    private Service $service;

    #[Route('/path')]
    public function action(#[Sensitive] string $password): void {}
}
```

Особенности синтаксиса:
- Начинается с `#[` и заканчивается `]` (не путать с комментарием `#`).
- Несколько атрибутов — отдельными группами или через запятую внутри одной.
- Имя атрибута — это **имя класса**; аргументы — как у конструктора.
- Поддерживают пространства имён и `use`.

```php
<?php
use App\Attributes\Route;

#[Route('/home')]              // короткое имя через use
#[\App\Attributes\Cache(60)]   // полное имя
class Controller {}
```

---

## Объявление собственного атрибута

Атрибут — это обычный класс, помеченный встроенным атрибутом `#[Attribute]`.

```php
<?php
namespace App\Attributes;

use Attribute;

#[Attribute]                              // помечает класс как атрибут
class Route
{
    public function __construct(
        public string $path,
        public array $methods = ['GET'],
    ) {}
}
```

Теперь его можно применять:

```php
<?php
use App\Attributes\Route;

class UserController
{
    #[Route('/users', methods: ['GET', 'POST'])]
    public function index(): void {}
}
```

> Сам по себе атрибут ничего не делает — это просто данные. Логику применяет код, который **читает** атрибуты через Reflection (обычно фреймворк).

---

## Цели атрибута (#[Attribute] flags)

Можно ограничить, к чему применим атрибут, и разрешить ли повторное применение — флагами в `#[Attribute(...)]`.

```php
<?php
use Attribute;

#[Attribute(Attribute::TARGET_METHOD | Attribute::TARGET_FUNCTION)]
class Route { /* ... */ }
```

Доступные цели (битовые флаги):

| Флаг | Где применим |
|------|--------------|
| `Attribute::TARGET_CLASS` | классы |
| `Attribute::TARGET_FUNCTION` | функции |
| `Attribute::TARGET_METHOD` | методы |
| `Attribute::TARGET_PROPERTY` | свойства |
| `Attribute::TARGET_CLASS_CONSTANT` | константы класса |
| `Attribute::TARGET_PARAMETER` | параметры |
| `Attribute::TARGET_ALL` | всё (по умолчанию) |
| `Attribute::IS_REPEATABLE` | разрешить несколько одинаковых атрибутов на одной цели |

```php
<?php
// Атрибут только для свойств, можно повторять
#[Attribute(Attribute::TARGET_PROPERTY | Attribute::IS_REPEATABLE)]
class Validate
{
    public function __construct(public string $rule) {}
}

class User
{
    #[Validate('required')]
    #[Validate('email')]              // повторно — благодаря IS_REPEATABLE
    public string $email;
}
```

> Если применить атрибут не к разрешённой цели, при чтении через Reflection (`newInstance()`) будет `Error`.

---

## Чтение атрибутов через Reflection

Атрибуты считываются во время выполнения через Reflection API. Метод `getAttributes()` возвращает объекты `ReflectionAttribute`, а `newInstance()` создаёт сам объект атрибута.

```php
<?php
$reflection = new ReflectionClass(UserController::class);

foreach ($reflection->getMethods() as $method) {
    // Получить атрибуты конкретного типа
    $attributes = $method->getAttributes(Route::class);

    foreach ($attributes as $attribute) {
        echo $attribute->getName();        // App\Attributes\Route
        var_dump($attribute->getArguments()); // сырые аргументы

        $route = $attribute->newInstance(); // создать объект Route
        echo $route->path;                   // /users
        var_dump($route->methods);           // ['GET', 'POST']
    }
}
```

Reflection-классы с методом `getAttributes()`: `ReflectionClass`, `ReflectionMethod`, `ReflectionProperty`, `ReflectionFunction`, `ReflectionParameter`, `ReflectionClassConstant`, `ReflectionEnum`.

```php
<?php
// Фильтрация: получить только атрибуты, наследующие интерфейс/класс
$attrs = $reflection->getAttributes(
    SomeInterface::class,
    ReflectionAttribute::IS_INSTANCEOF   // учесть наследников
);
```

---

## Аргументы атрибутов

Аргументы атрибута — это аргументы конструктора его класса. Допустимы только **константные выражения** (литералы, константы, enum, массивы, `new` других объектов с 8.1).

```php
<?php
#[Attribute]
class Column
{
    public function __construct(
        public string $type = 'string',
        public bool $nullable = false,
        public ?int $length = null,
    ) {}
}

class Product
{
    #[Column(type: 'string', length: 255)]   // именованные аргументы — удобно
    public string $name;

    #[Column('integer', nullable: true)]      // позиционные + именованные
    public ?int $stock;
}

// Можно использовать enum и константы как аргументы
#[Column(type: ColumnType::String)]
public string $title;
```

---

## Атрибуты vs аннотации PHPDoc

| | PHPDoc-аннотации | Атрибуты |
|---|------------------|----------|
| Где живут | в комментариях `/** */` | в коде, синтаксис `#[...]` |
| Проверка синтаксиса | нет (просто строка) | да (компилятор PHP) |
| Парсер | сторонний (doctrine/annotations) | нативный Reflection |
| Опечатки/ошибки имён | не ловятся | ловятся (класс должен существовать) |
| Автодополнение IDE | ограниченно | полноценно |
| Доступно с | всегда (через библиотеки) | PHP 8.0+ |

> Современные фреймворки (Symfony, Doctrine) перешли с аннотаций на атрибуты. Аннотации в комментариях считаются устаревшим подходом.

---

## Встроенные атрибуты PHP

PHP предоставляет несколько встроенных атрибутов:

```php
<?php
// #[Attribute] — помечает класс как атрибут (см. выше)

// #[\Deprecated] (PHP 8.4+) — помечает функцию/метод/константу как устаревшие
#[\Deprecated(message: 'Используйте newMethod()', since: '2.0')]
function oldMethod(): void {}

// #[\Override] (PHP 8.3+) — гарантирует, что метод переопределяет родительский
class Child extends Parent {
    #[\Override]
    public function handle(): void {}   // ошибка компиляции, если у предка нет handle()
}

// #[\ReturnTypeWillChange] — подавляет deprecation о смене типа возврата
// #[\AllowDynamicProperties] (8.2+) — разрешает динамические свойства классу
#[\AllowDynamicProperties]
class Flexible {}

// #[\SensitiveParameter] (8.2+) — скрывает значение параметра в стек-трейсах
function login(#[\SensitiveParameter] string $password): void {}
```

---

## Практический пример

Простой контейнер, читающий атрибут `#[Inject]` для автоподстановки зависимостей:

```php
<?php
use Attribute;

#[Attribute(Attribute::TARGET_PROPERTY)]
class Inject
{
    public function __construct(public string $service) {}
}

class Controller
{
    #[Inject('logger')]
    public object $logger;
}

// «Контейнер», который применяет атрибуты:
function injectDependencies(object $obj, array $services): void
{
    $reflection = new ReflectionObject($obj);

    foreach ($reflection->getProperties() as $property) {
        $attrs = $property->getAttributes(Inject::class);
        if ($attrs === []) {
            continue;
        }
        $inject = $attrs[0]->newInstance();
        $property->setValue($obj, $services[$inject->service]);
    }
}

$ctrl = new Controller();
injectDependencies($ctrl, ['logger' => new SomeLogger()]);
// теперь $ctrl->logger заполнен
```

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое атрибуты? | Машиночитаемые метаданные в синтаксисе `#[...]` (PHP 8.0+). |
| Чем заменяют аннотации PHPDoc? | Нативным синтаксисом с проверкой компилятором и Reflection. |
| Как объявить свой атрибут? | Обычный класс, помеченный `#[Attribute]`. |
| Делает ли атрибут что-то сам? | Нет — это данные; логику применяет код, читающий их через Reflection. |
| Как прочитать атрибут? | `$reflection->getAttributes()` → `->newInstance()`. |
| Как ограничить, куда применять атрибут? | Флаги `Attribute::TARGET_*` в `#[Attribute(...)]`. |
| Как разрешить повторение? | Флаг `Attribute::IS_REPEATABLE`. |
| Какие аргументы допустимы? | Константные выражения: литералы, константы, enum, массивы. |
| Назови встроенные атрибуты. | `#[Attribute]`, `#[\Override]` (8.3), `#[\SensitiveParameter]` (8.2), `#[\Deprecated]` (8.4). |
| Где используются на практике? | Роутинг, ORM-маппинг, DI, валидация (Symfony, Doctrine). |

---

*Источник: [php.net/manual/en/language.attributes.php](https://www.php.net/manual/en/language.attributes.php). Доступно с PHP 8.0+. См. также `php-oop.md` (`#[\Override]`), `php-namespaces.md`.*
