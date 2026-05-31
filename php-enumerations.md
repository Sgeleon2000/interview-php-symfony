# Перечисления в PHP (Enumerations)

> На основе [php.net/manual/en/language.enumerations.php](https://www.php.net/manual/en/language.enumerations.php). Доступны с **PHP 8.1+**.

**Перечисление (enum)** — это особый вид класса, определяющий замкнутый набор именованных значений (вариантов). Enum гарантирует, что переменная может принимать только одно из заранее заданных значений — это типобезопасная замена набору констант.

## Содержание

- [Зачем нужны enum](#зачем-нужны-enum)
- [Чистые перечисления (Pure Enums)](#чистые-перечисления-pure-enums)
- [Типизированные перечисления (Backed Enums)](#типизированные-перечисления-backed-enums)
- [Методы в enum](#методы-в-enum)
- [Константы и интерфейсы](#константы-и-интерфейсы)
- [Перечисление как объект](#перечисление-как-объект)
- [Методы cases(), from(), tryFrom()](#методы-cases-from-tryfrom)
- [Трейты в enum](#трейты-в-enum)
- [Ограничения](#ограничения)
- [Enum vs константы](#enum-vs-константы)

---

## Зачем нужны enum

Раньше фиксированный набор значений моделировали константами — но это небезопасно: в функцию можно передать любую строку/число.

```php
<?php
// ❌ Старый подход: набор констант
class Status {
    const DRAFT = 'draft';
    const PUBLISHED = 'published';
}
function setStatus(string $status) { /* можно передать любую строку! */ }
setStatus('whatever');   // ошибки не будет

// ✅ Enum: только допустимые значения
enum Status {
    case Draft;
    case Published;
}
function setStatus(Status $status) {}   // нельзя передать что-то постороннее
setStatus(Status::Draft);
```

---

## Чистые перечисления (Pure Enums)

Перечисление без привязанных значений. Каждый `case` — самостоятельный экземпляр-синглтон.

```php
<?php
enum Suit
{
    case Hearts;
    case Diamonds;
    case Clubs;
    case Spades;
}

$card = Suit::Hearts;

var_dump($card === Suit::Hearts);   // true
var_dump($card instanceof Suit);    // true
echo $card->name;                   // "Hearts" — встроенное свойство name
```

> У каждого case есть только свойство **`name`** (строка с именем варианта). Значения нет.

---

## Типизированные перечисления (Backed Enums)

Если каждому варианту нужно скалярное значение (для хранения в БД, передачи в API), используют backed enum — с типом `int` или `string`.

```php
<?php
enum Status: string          // backing-тип: string (или int)
{
    case Draft = 'draft';
    case Published = 'published';
    case Archived = 'archived';
}

$s = Status::Published;
echo $s->value;              // "published" — встроенное свойство value
echo $s->name;               // "Published"

enum Priority: int
{
    case Low = 1;
    case Medium = 2;
    case High = 3;
}
echo Priority::High->value;  // 3
```

Правила backed enum:
- Тип указывается после имени: `enum X: string`.
- Допустимые backing-типы — только `int` и `string`.
- Каждый `case` **обязан** иметь уникальное значение.
- Backed enum реализует встроенный интерфейс `BackedEnum`.

---

## Методы в enum

Перечисления могут содержать методы (как обычные классы). Внутри `$this` — текущий вариант.

```php
<?php
enum Suit: string
{
    case Hearts = 'H';
    case Diamonds = 'D';
    case Clubs = 'C';
    case Spades = 'S';

    public function color(): string
    {
        return match ($this) {
            Suit::Hearts, Suit::Diamonds => 'Красная',
            Suit::Clubs, Suit::Spades   => 'Чёрная',
        };
    }

    public function label(): string
    {
        return ucfirst(strtolower($this->name));
    }

    // Статический метод (например, фабрика)
    public static function default(): self
    {
        return self::Spades;
    }
}

echo Suit::Hearts->color();    // Красная
echo Suit::default()->name;    // Spades
```

> `match ($this)` — идиоматичный способ ветвления по варианту enum.

---

## Константы и интерфейсы

Enum может объявлять константы и реализовывать интерфейсы.

```php
<?php
interface HasLabel
{
    public function label(): string;
}

enum Status: string implements HasLabel
{
    case Active = 'active';
    case Banned = 'banned';

    const DEFAULT = self::Active;   // константа может ссылаться на case

    public function label(): string
    {
        return match ($this) {
            Status::Active => 'Активен',
            Status::Banned => 'Заблокирован',
        };
    }
}

function render(HasLabel $item): string { return $item->label(); }
echo render(Status::Active);        // Активен
echo Status::DEFAULT->value;        // active
```

---

## Перечисление как объект

Каждый case — это объект-синглтон типа enum. Поэтому:

```php
<?php
enum Suit { case Hearts; case Spades; }

$a = Suit::Hearts;
$b = Suit::Hearts;

var_dump($a === $b);             // true — один и тот же экземпляр
var_dump($a instanceof Suit);    // true
var_dump($a instanceof UnitEnum);// true — все enum реализуют UnitEnum

// Можно использовать в match/switch, как ключи нельзя (но как значения — да)
```

Встроенные интерфейсы:
- **`UnitEnum`** — реализуют все enum (метод `cases()`, свойство `name`).
- **`BackedEnum`** — реализуют backed enum (методы `from()`, `tryFrom()`, свойство `value`).

---

## Методы cases(), from(), tryFrom()

```php
<?php
enum Status: string
{
    case Active = 'active';
    case Banned = 'banned';
}

// cases() — массив всех вариантов (есть у любого enum)
foreach (Status::cases() as $case) {
    echo "{$case->name} = {$case->value}\n";
}

// from() — получить вариант по значению; бросает ValueError, если нет
$s = Status::from('active');          // Status::Active
// Status::from('xxx');               // ValueError

// tryFrom() — то же, но вернёт null вместо исключения
$s = Status::tryFrom('xxx');          // null
$s = Status::tryFrom('banned');       // Status::Banned

// Типичный приём при разборе ввода
$status = Status::tryFrom($_GET['status'] ?? '') ?? Status::Active;
```

> `from()` / `tryFrom()` доступны **только** у backed enum (через `BackedEnum`). У чистых enum их нет.

---

## Трейты в enum

Enum может использовать трейты — но трейт не может содержать свойства (только методы и статические методы).

```php
<?php
trait HasLabel
{
    public function label(): string
    {
        return ucfirst(strtolower($this->name));
    }
}

enum Role
{
    use HasLabel;

    case Admin;
    case Editor;
}

echo Role::Admin->label();   // Admin
```

---

## Ограничения

Enum — не полноценный класс. Нельзя:

- **Иметь состояние / свойства** (`$this` неизменяем, объявлять свойства запрещено).
- **Создавать экземпляры** через `new` — только обращение к `case`.
- **Наследовать** enum или наследоваться от него (`extends` запрещён).
- Использовать `static`-свойства.
- Реализовывать магические методы изменения состояния (`__construct`, `__set` и т.п.).

```php
<?php
// ❌ new Status();           — Error
// ❌ enum X extends Y {}     — Error
// ✅ Можно: методы, константы, интерфейсы, трейты (без свойств)
```

---

## Enum vs константы

| | Набор констант | Enum |
|---|----------------|------|
| Типобезопасность | ❌ можно передать любое значение | ✅ только допустимые варианты |
| Методы и поведение | ❌ | ✅ |
| Реализация интерфейсов | ❌ | ✅ |
| Перебор всех значений | вручную | `cases()` |
| Разбор из скаляра | вручную | `from()` / `tryFrom()` |
| Доступно с | всегда | PHP 8.1+ |

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| С какой версии появились enum? | PHP 8.1. |
| Pure enum vs backed enum? | Pure — без значений (только `name`); backed — со скалярным `value` (int/string). |
| Какие backing-типы допустимы? | Только `int` и `string`. |
| Как получить все варианты? | `Enum::cases()`. |
| `from()` vs `tryFrom()`? | `from()` бросает `ValueError`, `tryFrom()` возвращает `null`. |
| У какого enum есть `from()`? | Только у backed (интерфейс `BackedEnum`). |
| Как ветвиться по enum? | `match ($this)` / `match ($enum)`. |
| Могут ли enum иметь методы? | Да — обычные, статические, из трейтов. |
| Какие интерфейсы реализуют enum? | `UnitEnum` (все), `BackedEnum` (backed). |
| Что нельзя делать с enum? | Создавать через `new`, наследовать, хранить состояние/свойства. |
| Чем enum лучше констант? | Типобезопасность, методы, перебор, разбор из скаляра. |

---

*Источник: [php.net/manual/en/language.enumerations.php](https://www.php.net/manual/en/language.enumerations.php). Доступно с PHP 8.1+. См. также `php-types.md`, `php-oop.md`, `php-constants.md`.*
