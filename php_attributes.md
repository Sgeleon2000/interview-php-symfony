# PHP Attributes — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [PHP Attributes](https://www.php.net/manual/en/language.attributes.php), с пояснениями в комментариях.

**Атрибуты (PHP 8.0+)** — структурированные машиночитаемые метаданные для классов, методов, функций, параметров, свойств и констант класса. Читаются через Reflection API во время выполнения, позволяют добавлять декларативные метаданные без изменения структуры кода.

**Зачем нужны:**
- Отделить *что* делает фича от *как* она вызывается
- Альтернатива опциональным методам интерфейса (отметить метод как `#[SetUp]`, вместо того чтобы заставлять все классы реализовать `setUp()`)
- Декларативная конфигурация (роутинг, валидация, ORM-маппинг и т.д.)

---

## 1. Синтаксис

```php
// Атрибут открывается #[ и закрывается ]
#[MyAttribute]              // без аргументов
#[MyAttribute(1234)]        // позиционный аргумент
#[MyAttribute(value: 1234)] // именованный аргумент

// Несколько атрибутов:
#[First]
#[Second]
class Foo {}

// Или через запятую внутри одного блока:
#[First, Second]
class Bar {}
```

Полный пример всех форм:

```php
<?php
// a.php
namespace MyExample;

use Attribute;

#[Attribute]
class MyAttribute
{
    const VALUE = 'value';

    public function __construct(private mixed $value = null) {}
}

// b.php
namespace Another;

use MyExample\MyAttribute;

#[MyAttribute]                                  // 1) без аргументов
#[\MyExample\MyAttribute]                       // 2) fully-qualified имя
#[MyAttribute(1234)]                            // 3) позиционный аргумент
#[MyAttribute(value: 1234)]                     // 4) именованный аргумент
#[MyAttribute(MyAttribute::VALUE)]              // 5) константа класса
#[MyAttribute(['key' => 'value'])]              // 6) массив
#[MyAttribute(100 + 200)]                       // 7) константное выражение
class Thing {}

// Два одинаковых атрибута на одном классе — нужен IS_REPEATABLE (см. ниже)
#[MyAttribute(1234), MyAttribute(5678)]
class AnotherThing {}
```

**Правила синтаксиса:**
- Имя атрибута может быть unqualified / qualified / fully-qualified — как обычный класс
- Аргументы — только **литералы** и **константные выражения** (нельзя `new Foo()`, нельзя переменные)
- Поддерживаются и **позиционные**, и **именованные** аргументы
- Имя атрибута резолвится в класс, аргументы передаются в его конструктор — **но только когда атрибут запрашивается через Reflection API**

---

## 2. Объявление класса-атрибута

### 2.1. Минимальный случай — пустой класс

```php
<?php
namespace App\Attribute;

use Attribute;

// Без #[Attribute] класс работать как атрибут НЕ будет.
#[Attribute]
class Marker
{
}
```

### 2.2. С аргументами в конструкторе

```php
<?php
use Attribute;

#[Attribute]
class Route
{
    // Аргументы конструктора — это и есть аргументы атрибута.
    // public readonly свойства — современная идиоматичная форма.
    public function __construct(
        public readonly string $path,
        public readonly string $method = 'GET',
    ) {}
}

// Использование:
#[Route('/users/{id}', method: 'POST')]
class UserController {}
```

### 2.3. Ограничение целей применения (targets)

По умолчанию атрибут можно навешивать на что угодно. Чтобы ограничить — передаём bitmask:

```php
<?php
use Attribute;

// Этот атрибут можно навесить ТОЛЬКО на методы и функции.
// Попытка применить к классу/свойству вызовет исключение при newInstance().
#[Attribute(Attribute::TARGET_METHOD | Attribute::TARGET_FUNCTION)]
class Cached
{
    public function __construct(public readonly int $ttl = 3600) {}
}
```

**Все возможные targets:**

| Константа                       | Куда можно навесить                |
| ------------------------------- | ---------------------------------- |
| `Attribute::TARGET_CLASS`       | Классы                             |
| `Attribute::TARGET_FUNCTION`    | Функции                            |
| `Attribute::TARGET_METHOD`      | Методы класса                      |
| `Attribute::TARGET_PROPERTY`    | Свойства класса                    |
| `Attribute::TARGET_CLASS_CONSTANT` | Константы класса                |
| `Attribute::TARGET_PARAMETER`   | Параметры функций/методов          |
| `Attribute::TARGET_ALL`         | Везде (значение по умолчанию)      |

### 2.4. Повторяемые атрибуты (IS_REPEATABLE)

По умолчанию **один и тот же атрибут можно навесить только один раз**. Чтобы разрешить повторное применение — флаг `IS_REPEATABLE`:

```php
<?php
use Attribute;

#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
class Role
{
    public function __construct(public readonly string $name) {}
}

class Article
{
    // Тот же атрибут навешен дважды — разрешено благодаря IS_REPEATABLE
    #[Role('admin')]
    #[Role('editor')]
    public function delete(): void {}
}
```

### 2.5. Сводный пример со всеми флагами

```php
<?php
use Attribute;

// #[Foo] — навешивается куда угодно, только один раз
#[Attribute]
class Foo {}

// #[Bar] — только на классы, можно вешать несколько раз, принимает variadic-аргументы
#[Attribute(Attribute::TARGET_CLASS | Attribute::IS_REPEATABLE)]
class Bar
{
    public function __construct(?string ...$args) {}
}

// #[Baz] — куда угодно (TARGET_ALL), но только один раз
#[Attribute(Attribute::TARGET_ALL)]
class Baz
{
    public function __construct(public readonly string $parameter) {}
}

// Применение:
#[Foo]                                          // [0]
#[Bar]                                          // [1] без аргументов
#[Bar('Banana')]                                // [2]
#[Bar('Banana', 'Apple', 'Lemon')]              // [3] variadic
#[Baz('The Only One')]                          // [4]
class Qux {}
```

---

## 3. Чтение атрибутов через Reflection API

Атрибут **не вызывается автоматически**. Чтобы получить его — нужно явно запросить через Reflection.

### 3.1. Базовое чтение

```php
<?php
use Attribute;

#[Attribute]
class MyAttribute
{
    public function __construct(public mixed $value) {}
}

#[MyAttribute(value: 1234)]
class Thing {}

// Создаём ReflectionClass
$reflection = new ReflectionClass(Thing::class);

// getAttributes() возвращает массив ReflectionAttribute
foreach ($reflection->getAttributes() as $attribute) {
    var_dump($attribute->getName());        // string(11) "MyAttribute"  — имя класса атрибута
    var_dump($attribute->getArguments());   // ['value' => 1234]          — сырые аргументы
    var_dump($attribute->newInstance());    // object(MyAttribute) { ... }— реальный объект
}
```

### 3.2. Фильтр по конкретному классу атрибута

```php
// Получим только MyAttribute, остальные проигнорируем
foreach ($reflection->getAttributes(MyAttribute::class) as $attribute) {
    $instance = $attribute->newInstance();  // → MyAttribute object
    echo $instance->value;                  // → 1234
}
```

### 3.3. `getAttributes()` для всех Reflection-классов

`getAttributes()` есть у всех типов рефлексии:

```php
<?php
// Класс
$ref = new ReflectionClass(MyClass::class);
$ref->getAttributes();

// Метод
$ref = new ReflectionMethod(MyClass::class, 'myMethod');
$ref->getAttributes();

// Свойство
$ref = new ReflectionProperty(MyClass::class, 'myProp');
$ref->getAttributes();

// Константа класса
$ref = new ReflectionClassConstant(MyClass::class, 'MY_CONST');
$ref->getAttributes();

// Функция
$ref = new ReflectionFunction('myFunction');
$ref->getAttributes();

// Параметр
$ref = new ReflectionParameter(['MyClass', 'myMethod'], 'myParam');
$ref->getAttributes();
```

### 3.4. Почему `newInstance()` отдельно от `getAttributes()`

```text
ReflectionAttribute — это ОПИСАНИЕ атрибута (имя + аргументы), а не сам объект.
newInstance() создаёт реальный объект уже в момент вызова.

Это даёт контроль над ошибками:
  • Класс атрибута не существует → видно через getName() БЕЗ падения
  • Аргументы не подходят к конструктору → исключение только при newInstance()
  • Target не совпадает (например, IS_REPEATABLE нарушен) → исключение при newInstance()

Можно перебрать список, проверить getName() и решить, создавать ли инстанс.
```

---

## 4. Большой практический пример: optional методы через атрибут

Из официальной документации — паттерн "опциональные методы интерфейса через атрибуты":

```php
<?php
use Attribute;

interface ActionHandler
{
    public function execute();
}

// Маркерный атрибут: "этот метод нужно вызвать до execute()"
#[Attribute(Attribute::TARGET_METHOD | Attribute::IS_REPEATABLE)]
class SetUp {}

class CopyFile implements ActionHandler
{
    public string $fileName;
    public string $targetDirectory;

    #[SetUp]
    public function fileExists(): void
    {
        if (!file_exists($this->fileName)) {
            throw new RuntimeException("File does not exist");
        }
    }

    #[SetUp]
    public function targetDirectoryExists(): void
    {
        if (!file_exists($this->targetDirectory)) {
            mkdir($this->targetDirectory);
        } elseif (!is_dir($this->targetDirectory)) {
            throw new RuntimeException("Target directory is not a directory");
        }
    }

    public function execute(): void
    {
        copy($this->fileName, $this->targetDirectory . '/' . basename($this->fileName));
    }
}

function executeAction(ActionHandler $actionHandler): void
{
    $reflection = new ReflectionObject($actionHandler);

    // Ищем ВСЕ методы, помеченные #[SetUp], и зовём их
    foreach ($reflection->getMethods() as $method) {
        if (count($method->getAttributes(SetUp::class)) > 0) {
            $actionHandler->{$method->getName()}();
        }
    }

    $actionHandler->execute();
}

$copyAction = new CopyFile();
$copyAction->fileName = "/tmp/foo.jpg";
$copyAction->targetDirectory = "/home/user";

executeAction($copyAction);
// → fileExists() и targetDirectoryExists() вызвались автоматически
// → потом execute()
```

**Преимущество над интерфейсом**: классам не нужно реализовывать обязательный `setUp()`, можно навесить `#[SetUp]` на любое количество методов с любыми именами.

---

## 5. Наследование атрибутов

Класс-атрибут можно унаследовать — это нормально (документация это поддерживает):

```php
<?php
use Attribute;

#[Attribute(Attribute::TARGET_PROPERTY)]
class PropertyMeta
{
    public function __construct(
        public readonly ?string $name = null,
        public readonly ?string $label = null,
    ) {}
}

// Дочерний атрибут — добавляет специфичные для int поля
#[Attribute(Attribute::TARGET_PROPERTY)]
class IntegerPropertyMeta extends PropertyMeta
{
    public function __construct(
        ?string $name = null,
        ?string $label = null,
        public readonly ?int $min = null,
        public readonly ?int $max = null,
    ) {
        parent::__construct($name, $label);
    }
}

class MyClass
{
    #[IntegerPropertyMeta('prop', 'property: ', min: 0, max: 10)]
    public int $prop = 5;
}
```

---

## 6. Полезный паттерн: JSON-сериализация через атрибуты

```php
<?php
use Attribute;

#[Attribute(Attribute::TARGET_PROPERTY | Attribute::TARGET_CLASS_CONSTANT)]
class JsonSerialize
{
    public function __construct(public readonly ?string $fieldName = null) {}
}

class User
{
    #[JsonSerialize]                           // ключ = имя свойства ('name')
    public string $name = '';

    #[JsonSerialize('email_address')]          // кастомное имя ключа
    public string $email = '';

    // Без атрибута — поле НЕ сериализуется
    protected string $passwordHash = '';
}

function serialize(object $obj): string
{
    $data = [];
    $ref = new ReflectionObject($obj);

    foreach ($ref->getProperties() as $prop) {
        $attrs = $prop->getAttributes(JsonSerialize::class);
        if (!$attrs) {
            continue; // нет атрибута — пропускаем
        }

        $meta = $attrs[0]->newInstance();
        $key  = $meta->fieldName ?? $prop->getName();
        $prop->setAccessible(true);
        $data[$key] = $prop->getValue($obj);
    }

    return json_encode($data, JSON_THROW_ON_ERROR);
}

$user = new User();
$user->name = 'Alice';
$user->email = 'alice@example.com';

echo serialize($user);
// → {"name":"Alice","email_address":"alice@example.com"}
```

---

## 7. Сводная таблица

| Концепт                                | Синтаксис / API                                          |
| -------------------------------------- | -------------------------------------------------------- |
| Объявить класс-атрибут                 | `#[Attribute]` над `class Foo`                           |
| Ограничить targets                     | `#[Attribute(Attribute::TARGET_METHOD \| ...)]`          |
| Разрешить повторное применение         | `\| Attribute::IS_REPEATABLE`                            |
| Применить атрибут                      | `#[Foo]`, `#[Foo(1)]`, `#[Foo(value: 1)]`                |
| Несколько атрибутов сразу              | `#[A]\n#[B]` или `#[A, B]`                               |
| Прочитать все атрибуты                 | `$reflection->getAttributes()`                           |
| Прочитать конкретный класс             | `$reflection->getAttributes(Foo::class)`                 |
| Имя атрибута без инстанцирования       | `$attr->getName()`                                       |
| Сырые аргументы                        | `$attr->getArguments()`                                  |
| Создать объект атрибута                | `$attr->newInstance()`                                   |

---

## 8. Главные правила

```text
✓ Один класс — один атрибут (рекомендация документации).
✓ Аргументы атрибута = аргументы конструктора класса.
✓ Используй public readonly свойства в конструкторе — современно и удобно.
✓ Ограничивай targets — ловишь ошибки на этапе newInstance().
✓ Атрибуты НЕ вызываются автоматически — нужен Reflection API.

✗ Не передавай в аргументы атрибута объекты — только литералы и константы.
✗ Не используй #[Attribute] на интерфейсе или абстрактном классе — нет смысла.
✗ Не забывай IS_REPEATABLE, если планируешь вешать один атрибут несколько раз.
✗ Не путай ReflectionAttribute (описание) и инстанс класса (newInstance()).
```

---

## 9. Где это используется в экосистеме

```text
• Symfony Routing:    #[Route('/path')]
• Symfony DI:         #[Autowire(...)], #[AsEventListener], #[AsCommand]
• Doctrine ORM:       #[Entity], #[Column], #[ManyToOne], #[Id]
• PHPUnit:            #[Test], #[DataProvider('...')], #[CoversClass(...)]
• Validation:         #[Assert\NotBlank], #[Assert\Length(min: 3)]
• Serializer:         #[Groups(['public'])], #[Ignore]
```

До PHP 8 для всего этого использовались **PHPDoc-аннотации** (как `@ORM\Column`) — это был хак на основе парсинга комментариев. Сейчас атрибуты — нативный, типобезопасный механизм.