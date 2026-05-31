# Предопределённые атрибуты в PHP

> На основе [php.net/manual/ru/reserved.attributes.php](https://www.php.net/manual/ru/reserved.attributes.php). Примеры на PHP 8.x.

Помимо возможности объявлять собственные атрибуты (см. `php-attributes.md`), PHP предоставляет набор **встроенных (предопределённых) атрибутов**, которые влияют на поведение движка: помечают код устаревшим, проверяют переопределение, скрывают чувствительные данные и т.д.

## Содержание

- [#[Attribute]](#attribute)
- [#[AllowDynamicProperties]](#allowdynamicproperties)
- [#[Override]](#override)
- [#[SensitiveParameter]](#sensitiveparameter)
- [#[Deprecated]](#deprecated)
- [#[ReturnTypeWillChange]](#returntypewillchange)
- [#[NoDiscard]](#nodiscard)
- [Сводная таблица](#сводная-таблица)

---

## #[Attribute]

Помечает класс как **атрибут** — то есть позволяет использовать его в синтаксисе `#[...]`. Принимает битовую маску целей (`TARGET_*`) и флаг `IS_REPEATABLE`.

```php
<?php
use Attribute;

#[Attribute(Attribute::TARGET_METHOD | Attribute::TARGET_FUNCTION)]
class Route
{
    public function __construct(public string $path) {}
}
```

Доступно с PHP 8.0. Подробно об объявлении и чтении пользовательских атрибутов — `php-attributes.md`.

---

## #[AllowDynamicProperties]

**(PHP 8.2+)** Разрешает классу иметь **динамические свойства** (не объявленные заранее). С PHP 8.2 создание необъявленных свойств считается устаревшим (`E_DEPRECATED`); этот атрибут отключает предупреждение для конкретного класса.

```php
<?php
// Без атрибута — Deprecated при записи необъявленного свойства
class Strict {}
$s = new Strict();
$s->foo = 1;                      // ⚠️ Deprecated (8.2+), Error (в будущем)

// С атрибутом — динамические свойства разрешены
#[\AllowDynamicProperties]
class Flexible {}
$f = new Flexible();
$f->foo = 1;                      // ✅ ОК, без предупреждения
```

> Атрибут наследуется потомками. Использовать стоит редко — обычно лучше объявить свойства явно или применить `__get`/`__set`. `stdClass` динамические свойства разрешены всегда.

---

## #[Override]

**(PHP 8.3+)** Гарантирует, что помеченный метод **действительно переопределяет** метод родителя или интерфейса. Если у предка такого метода нет — ошибка на этапе компиляции.

```php
<?php
class BaseController
{
    public function handle(): void {}
}

class UserController extends BaseController
{
    #[\Override]
    public function handle(): void {}      // ✅ метод есть у родителя

    #[\Override]
    public function handel(): void {}      // ❌ Fatal error: опечатка, у предка нет handel()
}
```

> Защищает от двух частых ошибок: опечатки в имени метода и удаления/переименования метода в родителе (тогда «переопределение» молча перестаёт работать). Аналог `@Override` в Java.

---

## #[SensitiveParameter]

**(PHP 8.2+)** Помечает параметр функции как **чувствительный** — его значение скрывается в стек-трейсах и сообщениях об ошибках (вместо значения подставляется объект `SensitiveParameterValue`).

```php
<?php
function connect(string $user, #[\SensitiveParameter] string $password): void
{
    throw new RuntimeException('Сбой подключения');
}

try {
    connect('admin', 'super-secret-123');
} catch (RuntimeException $e) {
    echo $e->getTraceAsString();
    // В трейсе: connect('admin', Object(SensitiveParameterValue))
    // Реальный пароль НЕ попадёт в логи
}
```

> Применяйте к паролям, API-ключам, токенам, номерам карт — везде, где утечка значения в логи опасна. См. также класс `SensitiveParameterValue` в `php-reserved-interfaces.md`.

---

## #[Deprecated]

**(PHP 8.4+)** Помечает функцию, метод, константу класса или enum-case как **устаревшие**. При обращении к ним движок выдаёт `E_USER_DEPRECATED`. Раньше для этого использовали только PHPDoc-тег `@deprecated` (который движок игнорировал).

```php
<?php
class Api
{
    #[\Deprecated(message: 'используйте newMethod()', since: '2.0')]
    public function oldMethod(): void {}

    public function newMethod(): void {}
}

$api = new Api();
$api->oldMethod();   // Deprecated: Method Api::oldMethod() is deprecated, используйте newMethod()
```

Параметры (оба необязательны):
- `message` — текст с пояснением/заменой;
- `since` — версия, с которой элемент устарел.

> В отличие от `@deprecated` в комментарии, этот атрибут реально генерирует runtime-предупреждение и виден инструментам.

---

## #[ReturnTypeWillChange]

Подавляет уведомление об устаревании, которое возникает, когда метод, переопределяющий **встроенный** метод, имеет несовместимый тип возвращаемого значения. Нужен в основном для совместимости при реализации внутренних интерфейсов (`ArrayAccess`, `Iterator` и т.п.) без указания точных типов.

```php
<?php
class LegacyCollection implements ArrayAccess
{
    // Встроенный offsetGet объявлен как : mixed.
    // Если хотим вернуть конкретный тип без нарушения совместимости:
    #[\ReturnTypeWillChange]
    public function offsetGet($offset)         // без : mixed
    {
        return $this->data[$offset] ?? null;
    }
    // ... остальные методы ArrayAccess
    public function offsetExists($o): bool { return false; }
    public function offsetSet($o, $v): void {}
    public function offsetUnset($o): void {}
}
```

> Доступен с PHP 8.0. В новом коде лучше указывать корректные типы (`: mixed`) и не использовать этот атрибут — он нужен в основном для постепенной миграции старого кода.

---

## #[NoDiscard]

**(PHP 8.5+)** Помечает функцию/метод так, что **игнорирование возвращаемого значения** вызывает предупреждение. Полезно для функций, чей результат важно использовать (например, иммутабельные операции, возвращающие новый объект).

```php
<?php
#[\NoDiscard(message: 'результат with() — новый объект, исходный не меняется')]
function withHeader(string $name, string $value): Response { /* ... */ }

withHeader('X-Foo', 'bar');          // ⚠️ предупреждение: результат отброшен
$response = withHeader('X-Foo', 'bar'); // ✅ результат использован

// Подавить намеренно можно приведением к (void):
(void) withHeader('X-Foo', 'bar');
```

> Особенно ценно для иммутабельных API (как PSR-7), где забыть присвоить результат `with*()` — частая ошибка.

---

## Сводная таблица

| Атрибут | Версия | Применяется к | Назначение |
|---------|--------|---------------|-----------|
| `#[Attribute]` | 8.0 | класс | объявить класс как атрибут |
| `#[ReturnTypeWillChange]` | 8.0 | метод | подавить deprecation о смене типа возврата |
| `#[AllowDynamicProperties]` | 8.2 | класс | разрешить динамические свойства |
| `#[SensitiveParameter]` | 8.2 | параметр | скрыть значение в стек-трейсах |
| `#[Override]` | 8.3 | метод | проверить, что метод переопределяет родительский |
| `#[Deprecated]` | 8.4 | функция/метод/константа | пометить устаревшим (runtime-предупреждение) |
| `#[NoDiscard]` | 8.5 | функция/метод | предупреждать при игнорировании результата |

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что делает `#[Attribute]`? | Помечает класс, чтобы его можно было использовать как атрибут (8.0). |
| Зачем `#[Override]`? | Проверка на этапе компиляции, что метод реально переопределяет родительский (8.3). |
| Зачем `#[SensitiveParameter]`? | Скрывает значение параметра (пароль, токен) в стек-трейсах и логах (8.2). |
| Что делает `#[AllowDynamicProperties]`? | Разрешает классу необъявленные свойства без Deprecated (8.2). |
| Чем `#[Deprecated]` лучше `@deprecated`? | Генерирует реальное runtime-предупреждение, а не просто комментарий (8.4). |
| Когда нужен `#[ReturnTypeWillChange]`? | При переопределении встроенного метода с несовместимым типом возврата (миграция). |
| Что делает `#[NoDiscard]`? | Предупреждает, если возвращаемое значение проигнорировано (8.5). |

---

*Источник: [php.net/manual/ru/reserved.attributes.php](https://www.php.net/manual/ru/reserved.attributes.php). Примеры на PHP 8.x. См. также `php-attributes.md`, `php-oop.md`, `php-reserved-interfaces.md`.*
