# ООП в PHP (Классы и объекты)

> На основе [php.net/manual/en/language.oop5.php](https://www.php.net/manual/en/language.oop5.php). Примеры на PHP 8.x.

PHP поддерживает полноценную объектно-ориентированную модель: классы, наследование, интерфейсы, абстрактные классы, трейты, видимость, статические члены, магические методы, перечисления и многое другое.

## Содержание

- [Классы и объекты](#классы-и-объекты)
- [Свойства и константы](#свойства-и-константы)
- [Конструктор и деструктор](#конструктор-и-деструктор)
- [Видимость (модификаторы доступа)](#видимость-модификаторы-доступа)
- [Наследование (extends)](#наследование-extends)
- [Абстрактные классы](#абстрактные-классы)
- [Интерфейсы](#интерфейсы)
- [Трейты (traits)](#трейты-traits)
- [Статические свойства и методы](#статические-свойства-и-методы)
- [Константы класса](#константы-класса)
- [Поздним статическим связыванием (static::)](#поздним-статическим-связыванием-static)
- [Магические методы](#магические-методы)
- [Перегрузка (overloading)](#перегрузка-overloading)
- [Клонирование объектов](#клонирование-объектов)
- [Сравнение объектов](#сравнение-объектов)
- [final, readonly и другие модификаторы](#final-readonly-и-другие-модификаторы)
- [Анонимные классы](#анонимные-классы)
- [Перечисления (enum)](#перечисления-enum)
- [Namespace и автозагрузка](#namespace-и-автозагрузка)

---

## Классы и объекты

**Класс** — шаблон; **объект** — его экземпляр, созданный через `new`.

```php
<?php
class User
{
    public string $name;

    public function greet(): string
    {
        return "Привет, {$this->name}";   // $this — текущий объект
    }
}

$user = new User();           // создание экземпляра
$user->name = 'Анна';
echo $user->greet();          // Привет, Анна

// Проверка типа
var_dump($user instanceof User);   // true
echo $user::class;                 // "User" (PHP 8.0+)
echo get_class($user);             // "User"
```

---

## Свойства и константы

Свойства (поля) объявляются с указанием видимости и (опционально) типа.

```php
<?php
class Product
{
    public string $name;
    public float $price = 0.0;       // значение по умолчанию
    protected ?string $sku = null;   // nullable
    private array $tags = [];
    public const CURRENCY = 'USD';   // константа класса
}
```

> Типизированные свойства (PHP 7.4+) без значения по умолчанию находятся в состоянии «uninitialized» — обращение до инициализации даёт ошибку.

---

## Конструктор и деструктор

`__construct()` вызывается при создании объекта, `__destruct()` — при уничтожении.

```php
<?php
class Connection
{
    public function __construct(
        private string $host,        // Constructor Property Promotion (8.0+)
        private int $port = 5432,
    ) {
        echo "Подключение к {$this->host}\n";
    }

    public function __destruct()
    {
        echo "Соединение закрыто\n";
    }
}

$conn = new Connection('localhost');
```

> **Constructor Property Promotion (8.0+)** — свойства объявляются прямо в параметрах конструктора с указанием видимости, что убирает шаблонный код `$this->x = $x;`.

---

## Видимость (модификаторы доступа)

| Модификатор | Доступ |
|-------------|--------|
| `public` | отовсюду |
| `protected` | внутри класса и его наследников |
| `private` | только внутри объявившего класса |

```php
<?php
class Account
{
    public string $owner;
    protected float $balance = 0;
    private string $pin = '0000';

    public function deposit(float $amount): void
    {
        $this->balance += $amount;   // доступ к protected внутри класса
    }
}
```

---

## Наследование (extends)

Класс наследует свойства и методы родителя. PHP поддерживает только **одиночное** наследование (один родитель).

```php
<?php
class Animal
{
    public function __construct(protected string $name) {}

    public function speak(): string
    {
        return '...';
    }
}

class Dog extends Animal
{
    public function speak(): string      // переопределение метода
    {
        return 'Гав!';
    }

    public function describe(): string
    {
        return parent::speak() . " {$this->name}";   // вызов метода родителя
    }
}

$dog = new Dog('Рекс');
echo $dog->speak();   // Гав!
```

> Атрибут `#[\Override]` (PHP 8.3+) помечает метод как переопределяющий родительский — компилятор проверит, что метод действительно существует у предка.

---

## Абстрактные классы

Нельзя инстанцировать; задают «шаблон» с обязательными к реализации абстрактными методами.

```php
<?php
abstract class Shape
{
    abstract public function area(): float;   // без тела — реализуют наследники

    public function describe(): string         // обычный метод тоже можно
    {
        return 'Площадь: ' . $this->area();
    }
}

class Circle extends Shape
{
    public function __construct(private float $r) {}

    public function area(): float
    {
        return 3.14159 * $this->r ** 2;
    }
}

// new Shape();  // Fatal error
echo (new Circle(2))->describe();
```

---

## Интерфейсы

Контракт без реализации (только сигнатуры методов и константы). Класс может реализовать **несколько** интерфейсов.

```php
<?php
interface Comparable
{
    public function compareTo(self $other): int;
}

interface Printable
{
    public function toString(): string;
}

class Money implements Comparable, Printable
{
    public function __construct(private int $cents) {}

    public function compareTo(self $other): int
    {
        return $this->cents <=> $other->cents;
    }

    public function toString(): string
    {
        return number_format($this->cents / 100, 2);
    }
}
```

**Интерфейс vs абстрактный класс:**

| | Интерфейс | Абстрактный класс |
|---|-----------|-------------------|
| Реализация методов | нет (с 8.0 нет даже частичной) | может быть |
| Свойства | нет (только константы) | да |
| Множественность | реализовать можно несколько | наследовать только один |
| Назначение | контракт «что умеет» | общая база «что есть» |

---

## Трейты (traits)

Механизм **горизонтального переиспользования** кода — «копируют» методы в класс. Решают проблему отсутствия множественного наследования.

```php
<?php
trait Timestampable
{
    private ?string $createdAt = null;

    public function touch(): void
    {
        $this->createdAt = date('Y-m-d H:i:s');
    }
}

trait Loggable
{
    public function log(string $msg): void { echo "[LOG] $msg\n"; }
}

class Post
{
    use Timestampable, Loggable;   // подключение нескольких трейтов
}

$post = new Post();
$post->touch();
$post->log('создан');
```

> При конфликте методов из разных трейтов используют `insteadof` и `as` для разрешения. Трейты могут содержать абстрактные и статические методы.

---

## Статические свойства и методы

Принадлежат классу, а не экземпляру. Доступ через `::`.

```php
<?php
class Counter
{
    public static int $count = 0;

    public static function increment(): void
    {
        self::$count++;        // обращение к статике через self::
    }
}

Counter::increment();
Counter::increment();
echo Counter::$count;   // 2
```

> В статических методах нет `$this`. Используются для фабрик, утилит, счётчиков.

---

## Константы класса

```php
<?php
class Config
{
    public const VERSION = '1.0';
    protected const SECRET = 'xxx';        // видимость (7.1+)
    final public const LOCKED = true;       // final (8.1+) — нельзя переопределить

    const int MAX = 100;                    // типизированные константы (8.3+)
}

echo Config::VERSION;
```

Подробнее — см. `php-constants.md` (раздел «Константы классов»).

---

## Поздним статическим связыванием (static::)

`self::` ссылается на класс, где написан код; `static::` — на класс, который реально вызван (учитывает наследника). Это **Late Static Binding**.

```php
<?php
class Base
{
    public static function create(): static   // вернёт тип фактического класса
    {
        return new static();   // НЕ new self()
    }

    public function whoSelf(): string { return self::class; }
    public function whoStatic(): string { return static::class; }
}

class Derived extends Base {}

$obj = Derived::create();
var_dump($obj instanceof Derived);   // true (благодаря new static)
echo (new Derived())->whoSelf();     // Base
echo (new Derived())->whoStatic();   // Derived
```

---

## Магические методы

Специальные методы с префиксом `__`, вызываемые автоматически в определённых ситуациях:

| Метод | Когда вызывается |
|-------|------------------|
| `__construct` / `__destruct` | создание / уничтожение объекта |
| `__get($name)` | чтение недоступного/несуществующего свойства |
| `__set($name, $value)` | запись в недоступное свойство |
| `__isset` / `__unset` | `isset()` / `unset()` на недоступном свойстве |
| `__call($name, $args)` | вызов недоступного метода объекта |
| `__callStatic($name, $args)` | вызов недоступного статического метода |
| `__toString()` | приведение объекта к строке |
| `__invoke(...)` | вызов объекта как функции |
| `__clone()` | при `clone` |
| `__sleep` / `__wakeup` | сериализация / десериализация |
| `__serialize` / `__unserialize` | сериализация (новый механизм, 7.4+) |
| `__debugInfo()` | при `var_dump()` |

```php
<?php
class Magic
{
    private array $data = [];

    public function __get($name) { return $this->data[$name] ?? null; }
    public function __set($name, $value) { $this->data[$name] = $value; }
    public function __isset($name) { return isset($this->data[$name]); }

    public function __call($name, $args) { return "вызван $name"; }
    public static function __callStatic($name, $args) { return "статический $name"; }

    public function __toString(): string { return 'Magic object'; }
    public function __invoke($x) { return $x * 2; }
}

$m = new Magic();
$m->foo = 'bar';           // __set
echo $m->foo;              // __get → bar
echo $m->anything();       // __call → вызван anything
echo Magic::stat();        // __callStatic → статический stat
echo "$m";                 // __toString → Magic object
echo $m(21);               // __invoke → 42
```

---

## Перегрузка (overloading)

В PHP «overloading» означает **динамическое создание** свойств и методов через магические методы `__get`/`__set`/`__call` (см. выше), а **не** перегрузку сигнатур, как в Java/C++. Несколько методов с одним именем PHP не поддерживает.

---

## Клонирование объектов

`clone` создаёт **поверхностную** копию. Для глубокого копирования вложенных объектов реализуют `__clone()`.

```php
<?php
class Order
{
    public function __construct(public Customer $customer, public array $items = []) {}

    public function __clone()
    {
        $this->customer = clone $this->customer;   // глубокое копирование
    }
}

$copy = clone $order;   // $copy->customer — независимая копия
```

> См. также паттерн Prototype в `design-patterns.md`.

---

## Сравнение объектов

```php
<?php
$a = new stdClass(); $a->x = 1;
$b = new stdClass(); $b->x = 1;
$c = $a;

var_dump($a == $b);    // true  — одинаковый класс и равные свойства
var_dump($a === $b);   // false — разные экземпляры
var_dump($a === $c);   // true  — одна и та же ссылка-handle
```

---

## final, readonly и другие модификаторы

```php
<?php
final class Singleton {}        // нельзя наследовать

class Base
{
    final public function lock(): void {}   // нельзя переопределить в наследнике
}

class Point
{
    public function __construct(
        public readonly int $x,   // readonly свойство (8.1+) — задаётся 1 раз
        public readonly int $y,
    ) {}
}

$p = new Point(1, 2);
// $p->x = 5;   // Error: нельзя изменить readonly

// readonly class целиком (8.2+)
final readonly class Money
{
    public function __construct(public int $cents, public string $currency) {}
}
```

---

## Анонимные классы

Класс без имени — создаётся «на месте» (PHP 7.0+). Удобно для одноразовых реализаций интерфейсов.

```php
<?php
interface Logger { public function log(string $m): void; }

function run(Logger $logger): void { $logger->log('go'); }

run(new class implements Logger {
    public function log(string $m): void { echo $m; }
});
```

---

## Перечисления (enum)

Особый тип класса с фиксированным набором экземпляров (PHP 8.1+). Могут иметь методы, константы, реализовывать интерфейсы.

```php
<?php
enum Status: string
{
    case Active = 'active';
    case Banned = 'banned';

    public function label(): string
    {
        return match ($this) {
            Status::Active => 'Активен',
            Status::Banned => 'Заблокирован',
        };
    }
}

echo Status::Active->value;       // active
echo Status::Active->label();     // Активен
var_dump(Status::cases());        // все варианты
```

> Подробно об enum — `php-types.md`.

---

## Namespace и автозагрузка

Пространства имён группируют классы и избегают конфликтов имён. Автозагрузка (PSR-4) подключает файлы по имени класса.

```php
<?php
namespace App\Service;

use App\Repository\UserRepository;   // импорт
use Psr\Log\LoggerInterface as Log;  // псевдоним

class UserService
{
    public function __construct(
        private UserRepository $repo,
        private Log $logger,
    ) {}
}

// Полное имя: \App\Service\UserService
```

> Об автозагрузке PSR-4 через Composer — см. `psr-standards.md`.

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Поддерживает ли PHP множественное наследование? | Нет, только одиночное; для переиспользования — трейты и интерфейсы. |
| Интерфейс vs абстрактный класс? | Интерфейс — контракт без реализации (можно несколько); абстрактный класс — частичная реализация (один). |
| Что такое трейт? | Механизм горизонтального переиспользования кода между классами. |
| `self::` vs `static::`? | `self` — класс объявления; `static` — фактический класс (late static binding). |
| Что делает `new static()`? | Создаёт экземпляр класса, на котором вызван метод (учитывает наследника). |
| Назови магические методы. | `__construct`, `__get`, `__set`, `__call`, `__toString`, `__invoke`, `__clone` и др. |
| Что такое constructor property promotion? | Объявление свойств в параметрах конструктора (8.0+). |
| Что такое readonly свойство? | Свойство, которое можно присвоить лишь однажды (8.1+). |
| `clone` — глубокая или поверхностная копия? | Поверхностная; глубокую делают через `__clone()`. |
| `==` vs `===` для объектов? | `==` — тот же класс и равные свойства; `===` — тот же экземпляр. |
| Что такое анонимный класс? | Класс без имени, создаваемый «на месте». |
| Зачем `enum` вместо констант? | Типобезопасный фиксированный набор значений (8.1+). |

---

*Источник: [php.net/manual/en/language.oop5.php](https://www.php.net/manual/en/language.oop5.php). Примеры на PHP 8.x.*
