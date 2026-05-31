# Предопределённые интерфейсы и классы в PHP

> На основе [php.net/manual/ru/reserved.interfaces.php](https://www.php.net/manual/ru/reserved.interfaces.php). Примеры на PHP 8.x.

PHP предоставляет набор **встроенных интерфейсов**, которые позволяют объектам интегрироваться со встроенными конструкциями языка: `foreach`, `count()`, доступ по индексу `[]`, приведение к строке и т.д. Реализуя их, вы делаете свои классы «родными» для синтаксиса PHP.

## Содержание

- [Traversable, Iterator, IteratorAggregate](#traversable-iterator-iteratoraggregate)
- [ArrayAccess](#arrayaccess)
- [Countable](#countable)
- [Stringable](#stringable)
- [Throwable](#throwable)
- [UnitEnum и BackedEnum](#unitenum-и-backedenum)
- [Serializable (устарел)](#serializable-устарел)
- [Встроенные классы](#встроенные-классы)
  - [Closure](#closure)
  - [Generator](#generator)
  - [stdClass](#stdclass)
  - [WeakReference и WeakMap](#weakreference-и-weakmap)
  - [Fiber](#fiber)
  - [SensitiveParameterValue](#sensitiveparametervalue)
- [Сводная таблица](#сводная-таблица)

---

## Traversable, Iterator, IteratorAggregate

Группа интерфейсов для обхода объектов в `foreach`.

- **`Traversable`** — базовый «маркерный» интерфейс. Сам по себе нереализуем напрямую; служит для проверки `instanceof Traversable`. Его реализуют `Iterator` и `IteratorAggregate`.
- **`Iterator`** — объект сам управляет обходом (5 методов).
- **`IteratorAggregate`** — объект делегирует обход внешнему итератору (1 метод) — проще.

```php
<?php
// Iterator — полный контроль над обходом
class NumberCollection implements Iterator
{
    private int $pos = 0;
    public function __construct(private array $items = []) {}

    public function current(): mixed { return $this->items[$this->pos]; }
    public function key(): mixed     { return $this->pos; }
    public function next(): void     { $this->pos++; }
    public function rewind(): void   { $this->pos = 0; }
    public function valid(): bool    { return isset($this->items[$this->pos]); }
}

foreach (new NumberCollection([10, 20, 30]) as $k => $v) {
    echo "$k => $v\n";
}
```

```php
<?php
// IteratorAggregate — проще: вернуть готовый итератор
class UserList implements IteratorAggregate
{
    private array $users = [];
    public function add(string $u): void { $this->users[] = $u; }

    public function getIterator(): Iterator
    {
        return new ArrayIterator($this->users);   // или yield из генератора
    }
}

foreach (new UserList() as $user) { /* ... */ }
```

> Если объект просто оборачивает массив — берите `IteratorAggregate`. Если логика обхода сложная (БД-курсор, поток) — `Iterator`. Подробнее о паттерне — `design-patterns.md` (Iterator).

---

## ArrayAccess

Позволяет обращаться к объекту как к массиву — через `[]`.

```php
<?php
class Config implements ArrayAccess
{
    private array $data = [];

    public function offsetExists(mixed $offset): bool { return isset($this->data[$offset]); }
    public function offsetGet(mixed $offset): mixed { return $this->data[$offset] ?? null; }
    public function offsetSet(mixed $offset, mixed $value): void
    {
        if ($offset === null) { $this->data[] = $value; }  // $config[] = ...
        else { $this->data[$offset] = $value; }
    }
    public function offsetUnset(mixed $offset): void { unset($this->data[$offset]); }
}

$config = new Config();
$config['db'] = 'mysql';            // offsetSet
echo $config['db'];                 // offsetGet → mysql
var_dump(isset($config['db']));     // offsetExists → true
unset($config['db']);               // offsetUnset
```

---

## Countable

Позволяет применять к объекту функцию `count()`.

```php
<?php
class Cart implements Countable
{
    private array $items = [];
    public function add(string $item): void { $this->items[] = $item; }

    public function count(): int { return count($this->items); }
}

$cart = new Cart();
$cart->add('книга');
$cart->add('ручка');
echo count($cart);   // 2 — вызывает Countable::count()
```

---

## Stringable

Объект, который можно привести к строке. Реализуется через метод `__toString()` (PHP 8.0+ автоматически считает класс с `__toString()` реализующим `Stringable`).

```php
<?php
class Money implements Stringable
{
    public function __construct(private int $cents) {}

    public function __toString(): string
    {
        return number_format($this->cents / 100, 2) . ' ₽';
    }
}

$m = new Money(12345);
echo $m;                        // 123.45 ₽
$s = (string) $m;               // явное приведение
function render(string|Stringable $x): string { return (string) $x; }
```

---

## Throwable

Базовый интерфейс для всего, что можно бросить (`throw`) и поймать (`catch`) — реализуют `Exception` и `Error`. Подробно — см. `php-reserved-exceptions.md`.

```php
<?php
try { /* ... */ } catch (\Throwable $e) {
    // ловит и Exception, и Error
}
```

---

## UnitEnum и BackedEnum

Интерфейсы перечислений (PHP 8.1+):

- **`UnitEnum`** — реализуют **все** enum. Даёт статический метод `cases()` и свойство `name`.
- **`BackedEnum`** — реализуют **backed enum** (со значением). Добавляет `from()`, `tryFrom()` и свойство `value`.

```php
<?php
enum Status: string { case Active = 'active'; case Banned = 'banned'; }

var_dump(Status::Active instanceof UnitEnum);   // true
var_dump(Status::Active instanceof BackedEnum);  // true

// Типизация по интерфейсу — принять любой backed enum
function store(BackedEnum $e): string { return (string) $e->value; }
```

> Подробно об enum — `php-enumerations.md`.

---

## Serializable (устарел)

Старый интерфейс кастомной сериализации (`serialize()`/`unserialize()`). **Устарел** с PHP 8.1 — вместо него используют магические методы **`__serialize()`** и **`__unserialize()`**.

```php
<?php
// ✅ Современный способ (вместо Serializable)
class Session
{
    private array $data = ['user' => 1];

    public function __serialize(): array
    {
        return $this->data;                  // что сохранить
    }

    public function __unserialize(array $data): void
    {
        $this->data = $data;                 // как восстановить
    }
}

$s = serialize(new Session());
$restored = unserialize($s);
```

---

## Встроенные классы

### Closure

Класс, представляющий анонимные функции. Все `function () {}` и `fn() =>` — экземпляры `Closure`.

```php
<?php
$fn = fn($x) => $x * 2;
var_dump($fn instanceof Closure);   // true

// Привязка $this к другому объекту
$closure = function () { return $this->value; };
$bound = Closure::bind($closure, $someObject, SomeClass::class);

// fromCallable — превратить любой callable в Closure
$strlen = Closure::fromCallable('strlen');
```

### Generator

Объект, возвращаемый функцией-генератором (с `yield`). Реализует `Iterator`. Даёт ленивые последовательности без хранения всего в памяти.

```php
<?php
function range_lazy(int $start, int $end): Generator
{
    for ($i = $start; $i <= $end; $i++) {
        yield $i;                    // значения выдаются по требованию
    }
}

foreach (range_lazy(1, 1_000_000) as $n) {
    if ($n > 3) break;               // не создаёт миллион элементов в памяти
    echo $n;
}

// yield с ключом и приём значения
function gen(): Generator {
    $received = yield 'key' => 'value';
}
```

> Generator vs Fiber — см. `php-fibers.md`.

### stdClass

Пустой «универсальный» класс. Создаётся при приведении массива к объекту или для динамических свойств.

```php
<?php
$obj = new stdClass();
$obj->name = 'Анна';                // динамические свойства

$obj = (object) ['a' => 1, 'b' => 2];   // из массива
echo $obj->a;                       // 1

$data = json_decode('{"x": 1}');    // json_decode без true → stdClass
echo $data->x;                      // 1
```

### WeakReference и WeakMap

Слабые ссылки — не мешают сборщику мусора удалить объект (PHP 7.4+ / WeakMap 8.0+).

```php
<?php
// WeakReference — слабая ссылка на объект
$obj = new stdClass();
$weak = WeakReference::create($obj);
var_dump($weak->get());    // объект
unset($obj);
var_dump($weak->get());    // null — объект собран GC

// WeakMap — ключи-объекты без удержания их от сборки (для кэшей/метаданных)
$map = new WeakMap();
$key = new stdClass();
$map[$key] = 'метаданные';
echo count($map);          // 1
unset($key);
echo count($map);          // 0 — запись удалилась автоматически
```

> `WeakMap` идеален для кэширования данных, привязанных к объектам, без утечек памяти.

### Fiber

Приостанавливаемый стек выполнения для кооперативной многозадачности (PHP 8.1+). Подробно — `php-fibers.md`.

### SensitiveParameterValue

Обёртка для значения параметра, помеченного атрибутом `#[\SensitiveParameter]` — скрывает чувствительные данные (пароли, токены) в стек-трейсах.

```php
<?php
function login(#[\SensitiveParameter] string $password): void
{
    throw new Exception('fail');     // в трейсе $password будет скрыт
}
// В стек-трейсе: SensitiveParameterValue вместо реального значения
```

---

## Сводная таблица

| Интерфейс/класс | Назначение | Ключевые члены |
|-----------------|-----------|----------------|
| `Traversable` | маркер «обходимый» | — (база для Iterator/IteratorAggregate) |
| `Iterator` | обход в `foreach` | `current/key/next/rewind/valid` |
| `IteratorAggregate` | внешний итератор | `getIterator()` |
| `ArrayAccess` | доступ через `[]` | `offsetGet/Set/Exists/Unset` |
| `Countable` | работа с `count()` | `count()` |
| `Stringable` | приведение к строке | `__toString()` |
| `Throwable` | бросаемые объекты | методы исключений |
| `UnitEnum` / `BackedEnum` | перечисления | `cases()` / `from()`,`tryFrom()` |
| `Closure` | анонимные функции | `bind()`, `fromCallable()` |
| `Generator` | ленивые последовательности | `yield`, реализует `Iterator` |
| `stdClass` | универсальный объект | динамические свойства |
| `WeakReference`/`WeakMap` | слабые ссылки | не держат объект от GC |

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Как сделать объект обходимым в `foreach`? | Реализовать `Iterator` или (проще) `IteratorAggregate`. |
| `Iterator` vs `IteratorAggregate`? | Первый сам управляет обходом (5 методов); второй возвращает готовый итератор (1 метод). |
| Что такое `Traversable`? | Маркерный интерфейс-база; напрямую не реализуется. |
| Как обращаться к объекту через `[]`? | Реализовать `ArrayAccess`. |
| Как заставить работать `count($obj)`? | Реализовать `Countable`. |
| Что даёт `Stringable`? | Приведение к строке через `__toString()` (авто с 8.0). |
| Чем заменили `Serializable`? | Магическими `__serialize()` / `__unserialize()` (8.1). |
| Что такое `Closure`? | Класс анонимных функций; `bind`, `fromCallable`. |
| Что такое `Generator`? | Объект с `yield`, реализует `Iterator`, ленивые последовательности. |
| Зачем `WeakMap`? | Хранить данные по объектам-ключам без удержания их от сборки мусора. |
| Что такое `stdClass`? | Пустой класс для динамических свойств и `(object)`/`json_decode`. |

---

*Источник: [php.net/manual/ru/reserved.interfaces.php](https://www.php.net/manual/ru/reserved.interfaces.php). Примеры на PHP 8.x. См. также `php-oop.md`, `php-enumerations.md`, `php-fibers.md`, `php-reserved-exceptions.md`, `design-patterns.md`.*
