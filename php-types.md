# Типы данных в PHP

> На основе [php.net/manual/en/language.types.php](https://www.php.net/manual/en/language.types.php). Примеры на PHP 8.x.

PHP — язык с **динамической** и **слабой** типизацией. Тип переменной определяется значением, которое в неё записано, и может меняться во время выполнения. При этом начиная с PHP 7 можно объявлять типы аргументов, возвращаемых значений и свойств, а с `declare(strict_types=1)` — включить строгую проверку.

## Содержание

- [Система типов: обзор](#система-типов-обзор)
- [Скалярные типы](#скалярные-типы)
  - [null](#null)
  - [bool](#bool-boolean)
  - [int](#int-integer)
  - [float](#float)
  - [string](#string)
  - [Числовые строки](#числовые-строки-numeric-strings)
- [Составные типы](#составные-типы)
  - [array](#array)
  - [object](#object)
- [Специальные типы](#специальные-типы)
  - [resource](#resource)
  - [callable](#callable)
  - [enum](#enum-перечисления)
- [Объявления типов](#объявления-типов-type-declarations)
- [Особые объявления типов](#особые-объявления-типов)
- [Жонглирование типами](#жонглирование-типами-type-juggling)
- [Проверка и приведение типов](#проверка-и-приведение-типов)

---

## Система типов: обзор

PHP различает **10 базовых типов**:

| Категория | Типы |
|-----------|------|
| **Скалярные** | `bool`, `int`, `float`, `string` |
| **Составные** | `array`, `object` |
| **Специальные** | `resource`, `null`, `callable`, `enum` (перечисления) |

```php
<?php
$x = 42;            // int
$x = 3.14;          // float
$x = "hello";       // string
$x = true;          // bool
$x = [1, 2, 3];     // array
$x = new stdClass();// object
$x = null;          // null

echo gettype($x);   // узнать тип во время выполнения
```

---

## Скалярные типы

### null

Тип с единственным значением `null` — означает «отсутствие значения».

```php
<?php
$a = null;
var_dump(is_null($a));   // bool(true)
var_dump(isset($a));     // bool(false) — null считается «не установленным»

$b;                      // необъявленная переменная также null (с Warning при чтении)
```

### bool (boolean)

Логический тип: `true` или `false`.

**Что приводится к `false`:** `false`, `0`, `0.0`, `""`, `"0"`, пустой массив `[]`, `null`.
**Всё остальное — `true`** (в т.ч. `"0.0"`, `"false"`, `-1`, строка с пробелом).

```php
<?php
var_dump((bool) "");      // false
var_dump((bool) "0");     // false
var_dump((bool) "0.0");   // true  (внимание!)
var_dump((bool) []);      // false
var_dump((bool) -1);      // true
```

### int (integer)

Целое число. Размер зависит от платформы (обычно 64 бита). Литералы:

```php
<?php
$dec = 1234;        // десятичный
$oct = 0o755;       // восьмеричный (PHP 8.1+; старый синтаксис 0755)
$hex = 0x1A;        // шестнадцатеричный
$bin = 0b1010;      // двоичный
$big = 1_000_000;   // разделитель разрядов (PHP 7.4+)

echo PHP_INT_MAX;   // максимум; при переполнении int → float
```

### float

Число с плавающей точкой (двойной точности). Не хранит значения точно — нельзя сравнивать через `==`.

```php
<?php
$a = 1.5;
$b = 1.2e3;          // 1200.0 (экспоненциальная запись)
$c = 0.1 + 0.2;
var_dump($c == 0.3);                       // false! (погрешность)
var_dump(abs($c - 0.3) < PHP_FLOAT_EPSILON); // true — правильное сравнение

var_dump(is_nan(sqrt(-1)));  // bool(true)  — NAN
var_dump(is_infinite(1/0 ?? INF)); // INF
```

### string

Строка — последовательность байтов. Четыре способа задания:

```php
<?php
$name = 'Мир';

// 1) Одинарные кавычки — без интерполяции
echo 'Привет, $name';        // Привет, $name

// 2) Двойные кавычки — с интерполяцией и escape-последовательностями
echo "Привет, $name\n";      // Привет, Мир
echo "Сумма: {$name}!";      // фигурные скобки для сложных выражений

// 3) Heredoc — как двойные кавычки
$text = <<<EOT
Привет, $name
EOT;

// 4) Nowdoc — как одинарные кавычки (без интерполяции)
$raw = <<<'EOT'
Привет, $name
EOT;

// Доступ к символу по индексу
echo $name[0];               // первый байт
```

### Числовые строки (numeric strings)

Строка, содержащая число (`"123"`, `"  12.5"`, `"1e3"`), в арифметике ведёт себя как число.

```php
<?php
var_dump(is_numeric("123"));    // true
var_dump(is_numeric("12.5e3")); // true
var_dump(is_numeric("0x1A"));   // false (hex-строки больше не numeric)

var_dump("10" + 5);             // int(15)
// В PHP 8+ "abc" == 0 → false (раньше было true); сравнение строки и числа умнее
var_dump("abc" == 0);           // bool(false)
var_dump("1abc" + 1);           // int(2) + Warning (нечисловая строка)
```

---

## Составные типы

### array

Упорядоченная карта (ключ → значение). Ключи — `int` или `string`, значения — любого типа. Массив может быть индексным, ассоциативным или смешанным.

```php
<?php
$list  = [1, 2, 3];                      // индексный
$assoc = ['name' => 'Анна', 'age' => 30]; // ассоциативный
$nested = ['users' => [['id' => 1]]];    // многомерный

$list[] = 4;                  // добавить в конец
echo $assoc['name'];          // Анна
echo count($list);            // 4

// Деструктуризация
[$a, $b] = [10, 20];
['name' => $n] = $assoc;

// Spread (PHP 7.4+ / строковые ключи — 8.1+)
$merged = [...$list, 5, 6];
```

Ключевые особенности: дублирующиеся ключи перезаписываются; `bool`/`float`-ключи приводятся к `int`; `"8"` и `8` — один ключ, а `"08"` — строковый.

### object

Экземпляр класса. Создаётся через `new`. Объекты передаются **по ссылке-handle** (копируется ссылка, не данные).

```php
<?php
class Point {
    public function __construct(
        public float $x = 0,
        public float $y = 0,
    ) {}
}

$p = new Point(1.0, 2.0);
echo $p->x;                   // 1

// Быстрый объект «на лету»
$o = new stdClass();
$o->prop = 'value';

// Приведение массива к объекту
$obj = (object) ['a' => 1];
echo $obj->a;                 // 1
```

---

## Специальные типы

### resource

Специальная переменная — ссылка на внешний ресурс (открытый файл, соединение с БД, поток). Создаётся и освобождается специальными функциями.

```php
<?php
$handle = fopen('php://temp', 'r+');  // resource
var_dump(get_resource_type($handle)); // string(6) "stream"
fwrite($handle, 'data');
fclose($handle);
```

> Многие расширения постепенно переходят с `resource` на объекты (например, `CurlHandle`, `GdImage` стали объектами в PHP 8).

### callable

Тип для «вызываемого» — функции, метода, замыкания.

```php
<?php
// Имя функции в строке
$f1 = 'strlen';

// Метод объекта: [объект, 'метод'] или 'Class::method'
$f2 = [$obj, 'method'];
$f3 = 'MyClass::staticMethod';

// Замыкание (Closure) и стрелочная функция
$f4 = function (int $x): int { return $x * 2; };
$f5 = fn (int $x): int => $x * 2;     // PHP 7.4+

// First-class callable syntax (PHP 8.1+)
$f6 = strlen(...);

echo $f4(21);                          // 42
function apply(callable $fn, $arg) { return $fn($arg); }
```

### enum (Перечисления)

Перечисления — особый вид классов с фиксированным набором значений (PHP 8.1+).

```php
<?php
// Чистый enum
enum Status {
    case Draft;
    case Published;
    case Archived;
}

// Backed enum — со скалярным значением
enum Suit: string {
    case Hearts = 'H';
    case Spades = 'S';

    // У enum могут быть методы
    public function color(): string {
        return match ($this) {
            Suit::Hearts => 'red',
            Suit::Spades => 'black',
        };
    }
}

$s = Suit::Hearts;
echo $s->value;             // H
echo $s->color();           // red
var_dump(Suit::from('S'));  // Suit::Spades
var_dump(Suit::tryFrom('X')); // null
var_dump(Suit::cases());    // массив всех вариантов
```

---

## Объявления типов (Type Declarations)

Типы можно указывать у аргументов, возвращаемых значений и свойств.

```php
<?php

declare(strict_types=1);

class Repository
{
    // Типизированное свойство
    private array $items = [];

    // Типы аргументов и возврата
    public function add(string $key, int|float $value): void
    {
        $this->items[$key] = $value;
    }

    // Nullable-тип (?int = int|null)
    public function find(string $key): ?int
    {
        return $this->items[$key] ?? null;
    }

    // Union-тип (PHP 8.0+)
    public function get(string $key): int|float|null
    {
        return $this->items[$key] ?? null;
    }
}
```

**Виды объявлений:**
- **Nullable:** `?Type` — допускает `null`.
- **Union (PHP 8.0+):** `int|string` — один из перечисленных типов.
- **Intersection (PHP 8.1+):** `Countable&Iterator` — реализует все перечисленные интерфейсы.
- **`strict_types`:** при `declare(strict_types=1)` PHP не приводит типы автоматически (кроме `int`→`float`), а кидает `TypeError`.

---

## Особые объявления типов

| Тип | Где используется | Значение |
|-----|------------------|----------|
| **`mixed`** | аргумент/возврат/свойство | любой тип (эквивалент `object\|resource\|array\|string\|int\|float\|bool\|null`) |
| **`void`** | только возврат | функция ничего не возвращает |
| **`never`** | только возврат (8.1+) | функция никогда не возвращает (всегда `throw`/`exit`) |
| **`iterable`** | аргумент/возврат | `array` или `Traversable` |
| **`self` / `parent` / `static`** | внутри класса | относительные типы; `static` — позднее статическое связывание |
| **`false` / `true`** | union/самостоятельно (8.2+) | литеральные (singleton) типы |

```php
<?php
function dump(mixed $value): void {
    var_dump($value);                 // ничего не возвращает
}

function fail(string $msg): never {
    throw new \RuntimeException($msg); // никогда не вернётся
}

function numbers(): iterable {
    yield 1; yield 2;                  // генератор — тоже iterable
}
```

---

## Жонглирование типами (Type Juggling)

PHP автоматически приводит типы в зависимости от контекста (арифметика, сравнение, конкатенация).

```php
<?php
var_dump(5 + "5");        // int(10)   — строка → число
var_dump(5 . "5");        // string(2) "55" — число → строка
var_dump(true + 1);       // int(2)    — bool → int
var_dump("10" == 10);     // true      — нестрогое сравнение

// PHP 8 изменил сравнение строки и числа:
var_dump(0 == "foo");     // false (в PHP 7 было true!)
var_dump("1" == "01");    // true  — обе numeric, сравниваются как числа
var_dump("10" == "1e1");  // true
var_dump(100 == "1e2");   // true
```

> **Совет:** для предсказуемости используйте строгое сравнение `===` (сравнивает и значение, и тип) и включайте `declare(strict_types=1)`.

---

## Проверка и приведение типов

**Функции проверки типа:**

```php
<?php
is_int($x); is_float($x); is_string($x); is_bool($x);
is_array($x); is_object($x); is_callable($x); is_null($x);
is_numeric($x);            // число или числовая строка
is_iterable($x);           // array или Traversable
gettype($x);               // строковое имя типа
get_debug_type($x);        // точное имя типа/класса (PHP 8.0+) — лучше gettype
$x instanceof SomeClass;   // проверка класса/интерфейса
```

**Явное приведение (cast):**

```php
<?php
$i = (int) "123abc";      // 123
$f = (float) "3.14";      // 3.14
$s = (string) 42;         // "42"
$b = (bool) 0;            // false
$a = (array) "x";         // ["x"]
$o = (object) ['k' => 1]; // stdClass

// settype — приведение «на месте»
$v = "100";
settype($v, 'integer');   // $v стал int(100)
```

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Какая типизация в PHP? | Динамическая и слабая; со строгим режимом через `strict_types=1`. |
| Сколько базовых типов? | 10: bool, int, float, string, array, object, resource, null, callable, enum. |
| Разница `==` и `===`? | `==` приводит типы, `===` сравнивает значение **и** тип. |
| Что приводится к `false`? | `false`, `0`, `0.0`, `""`, `"0"`, `[]`, `null`. |
| Почему `0.1 + 0.2 != 0.3`? | Погрешность float; сравнивать через `PHP_FLOAT_EPSILON`. |
| Чем `mixed` отличается от `void`? | `mixed` — любой тип; `void` — функция ничего не возвращает. |
| Что такое `never`? | Возврат для функций, которые всегда бросают исключение или завершают выполнение. |
| Union vs Intersection? | `A\|B` — один из типов; `A&B` — реализует оба интерфейса. |
| Backed enum vs чистый enum? | Backed имеет скалярное `->value`; чистый — только `case`. |
| Как объекты передаются? | По ссылке-handle (копируется идентификатор объекта, не данные). |

---

*Источник: [php.net/manual/en/language.types.php](https://www.php.net/manual/en/language.types.php). Примеры на PHP 8.x.*
