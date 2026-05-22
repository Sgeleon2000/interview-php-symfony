# Doctrine ORM Basic Mapping — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [Basic Mapping](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/basic-mapping.html), с пояснениями в комментариях.

**Entity** — PHP-объект, который Doctrine может сохранить в БД. Mapping-метаданные говорят Doctrine, как объект превратить в строку таблицы и обратно. Они задаются через **PHP-атрибуты**, **XML** или **PHP-код** — все драйверы одинаково быстры (метаданные кэшируются).

---

## 1. Базовая entity-класс

`src/Entity/Message.php`

```php
<?php
namespace App\Entity;

use Doctrine\ORM\Mapping\Entity;
use Doctrine\ORM\Mapping\Table;

// #[Entity] — помечает класс как Doctrine-сущность
#[Entity]
// #[Table] опционально: по умолчанию имя таблицы = имя класса.
// Меняем явно — сущность Message будет жить в таблице 'message'.
#[Table(name: 'message')]
class Message
{
    private int $id;
    private string $text;
    private \DateTimeImmutable $postedAt;
}
```

Эквивалент в XML (`config/doctrine/Message.orm.xml`):

```xml
<doctrine-mapping>
    <entity name="App\Entity\Message" table="message">
        <!-- field-маппинги ниже -->
    </entity>
</doctrine-mapping>
```

---

## 2. Маппинг свойств — `#[Column]` со всеми опциями

`src/Entity/Article.php`

```php
<?php
namespace App\Entity;

use App\Enum\ArticleStatus;
use Doctrine\DBAL\Schema\DefaultExpression\CurrentTimestamp;
use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping\Column;
use Doctrine\ORM\Mapping\Entity;
use Doctrine\ORM\Mapping\GeneratedValue;
use Doctrine\ORM\Mapping\Id;

#[Entity]
class Article
{
    // ---------- ИДЕНТИФИКАТОР ----------
    #[Id]                                              // первичный ключ
    #[Column(type: Types::INTEGER)]
    #[GeneratedValue(strategy: 'AUTO')]                // стратегия генерации (см. ниже)
    private ?int $id = null;

    // ---------- Базовая строка ----------
    // type не указан → по умолчанию 'string' (VARCHAR), length не указан → 255
    #[Column]
    private string $shortTitle;

    // ---------- Все опции #[Column] ----------
    #[Column(
        name: 'full_title',           // имя колонки в БД (по умолч. = имя свойства)
        type: Types::STRING,          // Doctrine mapping type
        length: 500,                  // длина для string-колонок (default 255)
        unique: true,                 // UNIQUE constraint
        nullable: false,              // NOT NULL (default false)
        insertable: true,             // включать ли в INSERT (default true)
        updatable: true,              // включать ли в UPDATE (default true)
        // generated: 'INSERT',       // NEVER / INSERT / ALWAYS (для DB-generated колонок)
        columnDefinition: null,       // кастомный DDL (осторожно — путает SchemaTool)
        options: [
            'default'  => 'untitled', // DEFAULT-значение на уровне БД
            'comment'  => 'Полное название статьи',
            'collation' => 'utf8mb4_unicode_ci',
        ],
    )]
    private string $fullTitle;

    // ---------- Decimal с precision/scale ----------
    #[Column(
        type: Types::DECIMAL,
        precision: 10,                // всего цифр
        scale: 2,                     // из них после точки
    )]
    private string $price;            // decimal в PHP лучше держать как string!

    // ---------- Nullable + автоматический тип из PHP ----------
    // type не указан — Doctrine выведет из PHP-типа (см. таблицу ниже).
    // ВНИМАНИЕ: ?int НЕ делает колонку nullable автоматически —
    // nullable нужно указать ЯВНО.
    #[Column(nullable: true)]
    private ?int $views = null;

    // ---------- DEFAULT через options ----------
    #[Column(options: ['default' => 'Hello World!'])]
    private string $greeting = 'Hello World!';

    // ---------- CURRENT_TIMESTAMP на уровне БД ----------
    // insertable: false + updatable: false → колонкой управляет БД
    #[Column(
        options: ['default' => new CurrentTimestamp()],
        insertable: false,
        updatable: false,
    )]
    private \DateTimeImmutable $createdAt;

    // ---------- Кастомное имя колонки (snake_case) ----------
    #[Column(name: 'posted_at', type: Types::DATETIME_IMMUTABLE)]
    private \DateTimeImmutable $postedAt;

    // ---------- PHP-Enum как тип колонки ----------
    #[Column(enumType: ArticleStatus::class)]
    private ArticleStatus $status;
}
```

---

## 3. Автоматический маппинг типов из PHP-свойств (с Doctrine 2.9+)

Если опустить `type` в `#[Column]`, Doctrine выведет его из type-hint свойства:

| PHP-тип свойства      | Doctrine type               |
| --------------------- | --------------------------- |
| `int`                 | `Types::INTEGER`            |
| `float`               | `Types::FLOAT`              |
| `bool`                | `Types::BOOLEAN`            |
| `array`               | `Types::JSON`               |
| `DateInterval`        | `Types::DATEINTERVAL`       |
| `DateTime`            | `Types::DATETIME_MUTABLE`   |
| `DateTimeImmutable`   | `Types::DATETIME_IMMUTABLE` |
| PHP 8.1 enum          | enum (`enumType` подставляется автоматически с 2.11) |
| Любой другой          | `Types::STRING`             |

```php
// Достаточно одного #[Column] — тип определится из 'int'
#[Column]
private int $age;
```

> ⚠️ Nullable-свойство (`?int`) **не делает** колонку nullable. Указывай `nullable: true` явно.

---

## 4. Property Hooks (PHP 8.4)

Doctrine поддерживает PHP 8.4 property hooks. Можно объявлять getter/setter прямо на свойстве, и Doctrine это корректно сериализует.

```php
<?php
namespace App\Entity;

use Doctrine\ORM\Mapping\Column;
use Doctrine\ORM\Mapping\Entity;
use Doctrine\ORM\Mapping\Id;
use Doctrine\ORM\Mapping\GeneratedValue;

#[Entity]
class User
{
    #[Id]
    #[Column]
    #[GeneratedValue]
    private ?int $id = null;

    // Свойство с property hooks (PHP 8.4+)
    #[Column]
    public string $email {
        set(string $value) {
            // валидация при записи
            if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                throw new \InvalidArgumentException("Invalid email: $value");
            }
            $this->email = strtolower($value);
        }
        // get — по умолчанию вернёт сырое значение, его можно тоже переопределить
    }
}
```

---

## 5. Маппинг PHP Enums (Doctrine 2.11+)

### 5.1. Определение enum

`src/Enum/ArticleStatus.php`

```php
<?php
namespace App\Enum;

// ВАЖНО: это должен быть BACKED enum (с значением: 'string' или 'int')
// Pure enum (без backing-типа) Doctrine не поддерживает.
enum ArticleStatus: string
{
    case Draft     = 'draft';
    case Published = 'published';
    case Archived  = 'archived';
}
```

### 5.2. Single-value колонка с enum

```php
#[Column(type: Types::STRING, enumType: ArticleStatus::class)]
private ArticleStatus $status = ArticleStatus::Draft;
```

С Doctrine 2.11+ при наличии type-hint можно опустить и `type`, и `enumType`:

```php
#[Column]
private ArticleStatus $status = ArticleStatus::Draft;
```

### 5.3. Коллекция enums (массив enum'ов)

```php
/** @var ArticleStatus[] */
#[Column(type: Types::SIMPLE_ARRAY, enumType: ArticleStatus::class)]
private array $allowedStatuses = [];
```

### 5.4. Nullable enum

```php
#[Column(nullable: true, enumType: ArticleStatus::class)]
private ?ArticleStatus $status = null;
```

### 5.5. Default value для enum

```php
#[Column(
    enumType: ArticleStatus::class,
    options: ['default' => 'draft'], // значение БД, не enum case!
)]
private ArticleStatus $status = ArticleStatus::Draft;
```

### 5.6. Использование enum в DQL-запросах

```php
$qb = $em->createQueryBuilder()
    ->select('a')
    ->from(Article::class, 'a')
    ->where('a.status = :status')
    ->setParameter('status', ArticleStatus::Published); // передаём сам case

$articles = $qb->getQuery()->getResult();
```

### 5.7. XML-маппинг enum

```xml
<field name="status" enum-type="App\Enum\ArticleStatus"/>
```

---

## 6. Доступные Doctrine Mapping Types

Все основные типы из `Doctrine\DBAL\Types\Types`:

| Тип-константа              | SQL                | PHP                                  |
| -------------------------- | ------------------ | ------------------------------------ |
| `STRING`                   | VARCHAR            | `string`                             |
| `ASCII_STRING`             | VARCHAR (ASCII)    | `string`                             |
| `TEXT`                     | TEXT/CLOB          | `string`                             |
| `INTEGER`                  | INT                | `int`                                |
| `SMALLINT`                 | SMALLINT           | `int`                                |
| `BIGINT`                   | BIGINT             | `string` (для безопасности)          |
| `BOOLEAN`                  | BOOLEAN/TINYINT    | `bool`                               |
| `DECIMAL`                  | NUMERIC/DECIMAL    | `string`                             |
| `FLOAT`                    | FLOAT              | `float`                              |
| `BINARY`                   | VARBINARY          | resource/string                      |
| `BLOB`                     | BLOB               | resource                             |
| `GUID`                     | UUID/VARCHAR(36)   | `string`                             |
| `JSON`                     | JSON/TEXT          | `array`                              |
| `SIMPLE_ARRAY`             | TEXT (CSV)         | `array` of strings                   |
| `DATE_MUTABLE` / `IMMUTABLE` | DATE             | `DateTime` / `DateTimeImmutable`     |
| `DATETIME_MUTABLE` / `IMMUTABLE` | DATETIME     | `DateTime` / `DateTimeImmutable`     |
| `DATETIMETZ_MUTABLE` / `IMMUTABLE` | DATETIME w/ TZ | `DateTime` / `DateTimeImmutable`  |
| `TIME_MUTABLE` / `IMMUTABLE` | TIME             | `DateTime` / `DateTimeImmutable`     |
| `DATEINTERVAL`             | VARCHAR            | `DateInterval`                       |
| `ENUM`                     | (см. enumType)     | PHP enum                             |

> 💡 Совет: для денежных сумм используй `DECIMAL` и держи в PHP как `string` — иначе потеряешь точность.

---

## 7. Identifiers / Primary Keys

### 7.1. Стратегии генерации идентификаторов

```php
#[Id]
#[Column]
#[GeneratedValue(strategy: 'AUTO')]
private ?int $id = null;
```

Все стратегии `#[GeneratedValue(strategy: ...)]`:

| Стратегия      | Что делает                                                            |
| -------------- | --------------------------------------------------------------------- |
| `AUTO`         | Doctrine сам выбирает оптимальную стратегию для текущей БД (default) |
| `IDENTITY`     | AUTO_INCREMENT / SERIAL — БД сама присваивает ID                      |
| `SEQUENCE`     | Использует последовательность БД (PostgreSQL, Oracle)                 |
| `TABLE`        | Эмулирует sequence через отдельную таблицу (избегай)                  |
| `CUSTOM`       | Свой генератор через `#[CustomIdGenerator]`                           |
| `NONE`         | Никакой генерации — присваиваешь ID сам ДО `persist()`                |

### 7.2. Sequence Generator (PostgreSQL / Oracle)

```php
use Doctrine\ORM\Mapping\SequenceGenerator;

#[Id]
#[Column]
#[GeneratedValue(strategy: 'SEQUENCE')]
#[SequenceGenerator(
    sequenceName: 'article_id_seq',  // имя последовательности в БД
    allocationSize: 100,             // сколько ID забирать за раз (производительность)
    initialValue: 1,                 // стартовое значение
)]
private ?int $id = null;
```

### 7.3. Custom ID Generator (например, UUID)

```php
use Doctrine\ORM\Mapping\CustomIdGenerator;
use Symfony\Bridge\Doctrine\IdGenerator\UuidGenerator;

#[Id]
#[Column(type: 'uuid', unique: true)]
#[GeneratedValue(strategy: 'CUSTOM')]
#[CustomIdGenerator(class: UuidGenerator::class)]
private ?Uuid $id = null;
```

### 7.4. Ручной ID (NONE)

```php
#[Id]
#[Column]
#[GeneratedValue(strategy: 'NONE')]
private string $code; // присваиваешь сам перед persist()
```

### 7.5. Composite Keys (составной ключ)

```php
#[Entity]
class OrderItem
{
    // Несколько свойств помечены #[Id] → составной ключ
    #[Id]
    #[Column]
    private int $orderId;

    #[Id]
    #[Column]
    private int $productId;

    #[Column]
    private int $quantity;
}
```

> ⚠️ Composite keys **нельзя** использовать вместе с `#[GeneratedValue]` — ID должны быть присвоены вручную.
>
> 💡 Best practice от Doctrine: **избегай составных ключей**, используй суррогатный auto-increment ID + уникальный индекс.

### 7.6. Identity through foreign entities (внешний ключ как ID)

```php
#[Entity]
class Profile
{
    #[Id]
    #[OneToOne(targetEntity: User::class)]
    #[JoinColumn(name: 'user_id', referencedColumnName: 'id')]
    private User $user; // PK = FK на User.id
}
```

---

## 8. Quoting Reserved Words

Если имя поля или таблицы совпадает со SQL-словом (например, `user`, `order`, `select`), оберни его в обратные кавычки прямо в маппинге — Doctrine сам экранирует под целевую БД:

```php
#[Entity]
#[Table(name: '`order`')]      // зарезервированное слово таблицы
class Order
{
    #[Id]
    #[Column]
    #[GeneratedValue]
    private ?int $id = null;

    #[Column(name: '`select`')] // зарезервированное имя колонки
    private string $select;
}
```

В XML:

```xml
<entity name="App\Entity\Order" table="`order`">
    <field name="select" column="`select`"/>
</entity>
```

> ⚠️ Best practice от Doctrine: **не используй** зарезервированные слова и спецсимволы в именах — это создаёт проблемы при миграциях между БД. Лучше переименовать (`orders`, `selection`).

---

## 9. Полная сводная таблица опций `#[Column]`

| Опция              | Тип       | Default          | Назначение                                                |
| ------------------ | --------- | ---------------- | --------------------------------------------------------- |
| `type`             | string    | `'string'`       | Doctrine mapping type                                     |
| `name`             | string    | имя свойства     | Имя колонки в БД                                          |
| `length`           | int       | 255              | Длина (только для string)                                 |
| `unique`           | bool      | `false`          | UNIQUE constraint                                         |
| `nullable`         | bool      | `false`          | NULL разрешён                                             |
| `insertable`       | bool      | `true`           | Включать в INSERT                                         |
| `updatable`        | bool      | `true`           | Включать в UPDATE                                         |
| `generated`        | ?string   | `null`           | `'NEVER'` / `'INSERT'` / `'ALWAYS'` (DB-generated)        |
| `enumType`         | ?string   | `null`           | FQCN PHP enum-класса                                      |
| `precision`        | int       | 0                | Decimal: всего цифр                                       |
| `scale`            | int       | 0                | Decimal: цифр после запятой                               |
| `columnDefinition` | ?string   | `null`           | Сырой DDL (осторожно с SchemaTool!)                       |
| `options`          | array     | `[]`             | Опции БД: `default`, `comment`, `collation`, `unsigned`   |

---

## 10. Best Practices из документации

```text
✓ Используй #[GeneratedValue(strategy: 'AUTO')] — Doctrine выберет лучшее для БД.
✓ Decimal-поля держи в PHP как string — float теряет точность.
✓ Указывай nullable явно — type-hint ?type не делает колонку nullable.
✓ Используй DATETIME_IMMUTABLE / DateTimeImmutable — иммутабельные даты безопаснее.
✓ Backed enum + автоматический type/enumType (Doctrine 2.11+) — самый чистый код.

✗ Избегай composite keys — лучше суррогатный ID.
✗ Не используй reserved words в именах — миграции между БД станут болью.
✗ Не используй спецсимволы и identifier quoting без крайней необходимости.
✗ Не маппи foreign key как обычное поле — используй ассоциации (см. след. главу).
```

---

## 11. Минимальный рабочий пример

`src/Entity/Product.php`

```php
<?php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;
use Doctrine\DBAL\Types\Types;

#[ORM\Entity]
#[ORM\Table(name: 'products')]
class Product
{
    #[ORM\Id]
    #[ORM\Column]
    #[ORM\GeneratedValue]
    private ?int $id = null;

    #[ORM\Column(length: 200)]
    private string $name;

    #[ORM\Column(type: Types::DECIMAL, precision: 10, scale: 2)]
    private string $price;

    #[ORM\Column(type: Types::DATETIME_IMMUTABLE, options: [
        'default' => new \Doctrine\DBAL\Schema\DefaultExpression\CurrentTimestamp(),
    ], insertable: false, updatable: false)]
    private \DateTimeImmutable $createdAt;

    public function __construct(string $name, string $price)
    {
        $this->name  = $name;
        $this->price = $price;
    }

    public function getId(): ?int           { return $this->id; }
    public function getName(): string       { return $this->name; }
    public function getPrice(): string      { return $this->price; }
    public function getCreatedAt(): \DateTimeImmutable { return $this->createdAt; }
}
```

Использование:

```php
$product = new Product('Coffee', '4.50');
$em->persist($product);
$em->flush();

echo $product->getId(); // 1 (auto-generated)
```