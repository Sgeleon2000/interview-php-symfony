# Doctrine ORM Association Mapping — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [Association Mapping](https://www.doctrine-project.org/projects/doctrine-orm/en/latest/reference/association-mapping.html), с пояснениями в комментариях.

В Doctrine ты работаешь не с **внешними ключами**, а с **ссылками на объекты**. Doctrine сама превращает их в FK в БД.

- Ссылка на один объект → внешний ключ
- Коллекция объектов → много FK, указывающих на владельца коллекции

**Как читать имя связи** (слева направо, "слева" = текущая сущность):
- `OneToMany` — у одной текущей сущности **много** ссылок на целевую
- `ManyToOne` — много текущих сущностей ссылаются на **одну** целевую
- `OneToOne` — одна к одной
- `ManyToMany` — много к многим

**Однонаправленная (unidirectional)** связь — только одна сторона знает о другой.
**Двунаправленная (bidirectional)** — обе стороны видят друг друга.

В bidirectional-связи **есть owning side и inverse side**:
- `inversedBy` — на **owning** side (она держит FK, она управляет связью)
- `mappedBy` — на **inverse** side (зеркало, только для удобства чтения)

---

## 1. Composite Foreign Keys

Когда целевая сущность имеет составной первичный ключ, нужно несколько `JoinColumn`:

```php
<?php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class OrderItem
{
    #[ORM\ManyToOne(targetEntity: Product::class)]
    #[ORM\JoinColumn(name: 'product_id', referencedColumnName: 'id')]
    #[ORM\JoinColumn(name: 'product_version', referencedColumnName: 'version')]
    // Несколько #[JoinColumn] на одном свойстве = составной FK
    private Product $product;
}
```

---

## 2. Many-To-One, Unidirectional (самая частая связь)

Много пользователей живёт по одному адресу.

`src/Entity/User.php`

```php
<?php
namespace App\Entity;

use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    #[ORM\Id]
    #[ORM\Column]
    #[ORM\GeneratedValue]
    private ?int $id = null;

    // ManyToOne — owning side (FK живёт в этой таблице, в колонке address_id)
    // #[JoinColumn] МОЖНО опустить — Doctrine использует дефолт:
    //    name: '<fieldName>_id'          → 'address_id'
    //    referencedColumnName: 'id'      → 'id'
    // targetEntity тоже можно опустить — берётся из type-hint свойства.
    #[ORM\ManyToOne(targetEntity: Address::class)]
    #[ORM\JoinColumn(name: 'address_id', referencedColumnName: 'id')]
    private ?Address $address = null;
}

#[ORM\Entity]
class Address
{
    #[ORM\Id]
    #[ORM\Column]
    #[ORM\GeneratedValue]
    private ?int $id = null;
    // Не знает ничего про User → связь однонаправленная.
}
```

Сгенерированная схема MySQL:

```sql
CREATE TABLE User (
    id INT AUTO_INCREMENT NOT NULL,
    address_id INT DEFAULT NULL,
    PRIMARY KEY(id)
);
ALTER TABLE User ADD FOREIGN KEY (address_id) REFERENCES Address(id);
```

---

## 3. One-To-One, Unidirectional

`Product` ссылается на одну `Shipment`. `Shipment` про `Product` ничего не знает.

```php
<?php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Product
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // Owning side: FK + UNIQUE constraint в БД (отличие от ManyToOne!)
    #[ORM\OneToOne(targetEntity: Shipment::class)]
    #[ORM\JoinColumn(name: 'shipment_id', referencedColumnName: 'id')]
    private ?Shipment $shipment = null;
}

#[ORM\Entity]
class Shipment
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;
}
```

```sql
CREATE TABLE Product (
    id INT AUTO_INCREMENT NOT NULL,
    shipment_id INT DEFAULT NULL,
    UNIQUE INDEX UNIQ_xxx (shipment_id),  -- UNIQUE отличает OneToOne от ManyToOne
    PRIMARY KEY(id)
);
```

---

## 4. One-To-One, Bidirectional

`Customer` ↔ `Cart`. Обе стороны видят друг друга.

```php
<?php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Customer
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // INVERSE side — БЕЗ FK, только зеркало.
    // mappedBy указывает, в каком свойстве Cart лежит обратная ссылка.
    #[ORM\OneToOne(targetEntity: Cart::class, mappedBy: 'customer')]
    private ?Cart $cart = null;
}

#[ORM\Entity]
class Cart
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // OWNING side — здесь FK (customer_id).
    // inversedBy указывает, в каком свойстве Customer лежит обратная ссылка.
    #[ORM\OneToOne(targetEntity: Customer::class, inversedBy: 'cart')]
    #[ORM\JoinColumn(name: 'customer_id', referencedColumnName: 'id')]
    private ?Customer $customer = null;
}
```

Сторона выбирается так: **где FK — там owning side** (там `inversedBy`).

---

## 5. One-To-One, Self-Referencing

Студент имеет одного ментора (тоже студента).

```php
<?php
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Student
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // targetEntity = self → self-referencing
    #[ORM\OneToOne(targetEntity: Student::class)]
    #[ORM\JoinColumn(name: 'mentor_id', referencedColumnName: 'id')]
    private ?Student $mentor = null;
}
```

```sql
CREATE TABLE Student (
    id INT AUTO_INCREMENT NOT NULL,
    mentor_id INT DEFAULT NULL,
    PRIMARY KEY(id)
);
ALTER TABLE Student ADD FOREIGN KEY (mentor_id) REFERENCES Student(id);
```

---

## 6. One-To-Many, Bidirectional

> ⚠️ `OneToMany` **всегда требует пары** с `ManyToOne` на другой стороне (либо специальной таблицы-связи, см. п.7). Чисто unidirectional `OneToMany` без join-table в Doctrine невозможен — это ограничение реляционных моделей.

Featured / Article — у одной фичи много статей.

```php
<?php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Feature
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // INVERSE side: коллекция, без FK в этой таблице.
    // mappedBy указывает на свойство в Article, которое держит обратную ссылку.
    /** @var Collection<int, Article> */
    #[ORM\OneToMany(targetEntity: Article::class, mappedBy: 'feature')]
    private Collection $articles;

    public function __construct()
    {
        // ВАЖНО: всегда инициализируй коллекции в конструкторе!
        $this->articles = new ArrayCollection();
    }
}

#[ORM\Entity]
class Article
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // OWNING side: здесь FK (feature_id).
    // inversedBy указывает на коллекцию в Feature.
    #[ORM\ManyToOne(targetEntity: Feature::class, inversedBy: 'articles')]
    #[ORM\JoinColumn(name: 'feature_id', referencedColumnName: 'id')]
    private ?Feature $feature = null;
}
```

---

## 7. One-To-Many, Unidirectional with Join Table

Если нужна `OneToMany` БЕЗ обратной ссылки, придётся использовать промежуточную таблицу с UNIQUE на FK target-сущности.

```php
<?php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    /** @var Collection<int, Phonenumber> */
    #[ORM\OneToMany(targetEntity: Phonenumber::class)]
    // Промежуточная таблица users_phonenumbers (user_id, phonenumber_id)
    #[ORM\JoinTable(name: 'users_phonenumbers')]
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id')]
    // UNIQUE на phonenumber_id — гарантирует, что один номер привязан только к одному user
    #[ORM\InverseJoinColumn(name: 'phonenumber_id', referencedColumnName: 'id', unique: true)]
    private Collection $phonenumbers;

    public function __construct()
    {
        $this->phonenumbers = new ArrayCollection();
    }
}
```

---

## 8. One-To-Many, Self-Referencing

Категория с под-категориями (дерево).

```php
<?php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class Category
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // INVERSE: коллекция детей
    /** @var Collection<int, Category> */
    #[ORM\OneToMany(targetEntity: Category::class, mappedBy: 'parent')]
    private Collection $children;

    // OWNING: ссылка на родителя
    #[ORM\ManyToOne(targetEntity: Category::class, inversedBy: 'children')]
    #[ORM\JoinColumn(name: 'parent_id', referencedColumnName: 'id')]
    private ?Category $parent = null;

    public function __construct()
    {
        $this->children = new ArrayCollection();
    }
}
```

---

## 9. Many-To-Many, Unidirectional

User ↔ Group, но Group не знает про своих юзеров.

```php
<?php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    /** @var Collection<int, Group> */
    #[ORM\ManyToMany(targetEntity: Group::class)]
    #[ORM\JoinTable(name: 'users_groups')]                             // имя join-таблицы
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id')]     // FK на User
    #[ORM\InverseJoinColumn(name: 'group_id', referencedColumnName: 'id')] // FK на Group
    private Collection $groups;

    public function __construct()
    {
        $this->groups = new ArrayCollection();
    }
}

#[ORM\Entity]
class Group
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;
    // ничего не знает о User
}
```

Сгенерированная таблица:

```sql
CREATE TABLE users_groups (
    user_id INT NOT NULL,
    group_id INT NOT NULL,
    PRIMARY KEY(user_id, group_id)
);
```

---

## 10. Many-To-Many, Bidirectional

User ↔ Group, обе стороны видят друг друга.

```php
<?php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // OWNING side: эта сторона держит join-таблицу.
    // Только на owning side объявляется #[JoinTable]!
    /** @var Collection<int, Group> */
    #[ORM\ManyToMany(targetEntity: Group::class, inversedBy: 'users')]
    #[ORM\JoinTable(name: 'users_groups')]
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id')]
    #[ORM\InverseJoinColumn(name: 'group_id', referencedColumnName: 'id')]
    private Collection $groups;

    public function __construct() { $this->groups = new ArrayCollection(); }
}

#[ORM\Entity]
class Group
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    // INVERSE side: только mappedBy, никаких JoinTable/JoinColumn!
    /** @var Collection<int, User> */
    #[ORM\ManyToMany(targetEntity: User::class, mappedBy: 'groups')]
    private Collection $users;

    public function __construct() { $this->users = new ArrayCollection(); }
}
```

### 10.1. Owning vs Inverse в ManyToMany

```text
В ManyToMany сторону FK не определишь (она в join-таблице) → нужно ВЫБРАТЬ owning side.

ПРАВИЛА выбора:
  • На owning side кладётся #[JoinTable] и #[JoinColumn]/#[InverseJoinColumn].
  • На inverse side — только mappedBy.
  • Doctrine отслеживает изменения ТОЛЬКО на owning side!
    Это значит: если ты добавишь объект только в коллекцию inverse side,
    в БД ничего не сохранится. Всегда меняй owning side
    (или синхронизируй обе через addX()/removeX()).
```

---

## 11. Many-To-Many, Self-Referencing

Друзья пользователя — другие пользователи.

```php
<?php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity]
class User
{
    #[ORM\Id, ORM\Column, ORM\GeneratedValue]
    private ?int $id = null;

    /** @var Collection<int, User> */
    #[ORM\ManyToMany(targetEntity: User::class)]
    #[ORM\JoinTable(name: 'friends')]
    // 'user_id' — текущий пользователь
    #[ORM\JoinColumn(name: 'user_id', referencedColumnName: 'id')]
    // 'friend_user_id' — друг (тот же класс)
    #[ORM\InverseJoinColumn(name: 'friend_user_id', referencedColumnName: 'id')]
    private Collection $friends;

    public function __construct() { $this->friends = new ArrayCollection(); }
}
```

---

## 12. Mapping Defaults — что можно опустить

Doctrine применяет дефолты, чтобы не писать `#[JoinColumn]` каждый раз:

```php
// Минимальная форма (всё по умолчанию):
#[ORM\ManyToOne]
private ?Address $address = null;

// Эквивалент полной формы:
#[ORM\ManyToOne(targetEntity: Address::class)]
#[ORM\JoinColumn(name: 'address_id', referencedColumnName: 'id')]
private ?Address $address = null;
```

### Правила автогенерации:

| Что               | Дефолт                                                          |
| ----------------- | --------------------------------------------------------------- |
| `targetEntity`    | Берётся из PHP type-hint свойства                               |
| `JoinColumn.name` | `<имя_свойства>_id` (например, `address_id`)                    |
| `referencedColumnName` | `id`                                                       |
| `JoinTable.name`  | `<owning_table>_<inverse_table>` (например, `users_groups`)     |
| `JoinTable.joinColumn.name`         | `<owning_field>_id`                           |
| `JoinTable.inverseJoinColumn.name`  | `<inverse_field>_id`                          |

Минимальный пример ManyToMany с дефолтами:

```php
#[ORM\ManyToMany(targetEntity: Group::class, inversedBy: 'users')]
private Collection $groups;
// Doctrine сама создаст таблицу 'user_group' с колонками user_id и group_id
```

---

## 13. Collections — обязательная инициализация

### 13.1. ArrayCollection в конструкторе

```php
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;

class User
{
    /** @var Collection<int, Group> */
    #[ORM\ManyToMany(targetEntity: Group::class)]
    private Collection $groups;

    public function __construct()
    {
        // КРИТИЧНО: без этого получишь "Call to a member function add() on null"
        // как только попробуешь $user->getGroups()->add(...)
        $this->groups = new ArrayCollection();
    }
}
```

### 13.2. Type-hint всегда `Collection`, не `ArrayCollection`

```php
// ✓ Правильно — Collection (interface)
private Collection $groups;

// ✗ Неправильно — ArrayCollection (concrete)
// Когда Doctrine загрузит сущность из БД, она положит туда
// PersistentCollection, а не ArrayCollection!
private ArrayCollection $groups;
```

### 13.3. Зачем использовать `Doctrine\Common\Collections\Collection`

```text
Doctrine\Common\Collections\Collection расширяет PHP-массивы:
  • add(), remove(), removeElement(), contains(), filter(), map()
  • toArray() — превратить в обычный массив
  • count(), isEmpty(), first(), last()
  • matching(Criteria) — фильтрация без загрузки всей коллекции (Extra Lazy)

PersistentCollection (которую Doctrine использует за кулисами)
тоже реализует этот интерфейс, поэтому код одинаково работает
с новой сущностью (ArrayCollection) и загруженной (PersistentCollection).
```

### 13.4. Удобные методы add/remove (best practice)

```php
class Feature
{
    /** @var Collection<int, Article> */
    #[ORM\OneToMany(targetEntity: Article::class, mappedBy: 'feature')]
    private Collection $articles;

    public function __construct() { $this->articles = new ArrayCollection(); }

    // Симметричный add: добавляем И сюда, И на owning side
    public function addArticle(Article $article): void
    {
        if (!$this->articles->contains($article)) {
            $this->articles->add($article);
            $article->setFeature($this); // синхронизация owning side
        }
    }

    public function removeArticle(Article $article): void
    {
        if ($this->articles->removeElement($article)) {
            $article->setFeature(null); // синхронизация owning side
        }
    }
}
```

---

## 14. Сводная таблица всех типов связей

| Связь                              | Атрибуты                                              | Где FK                | Side(s)                              |
| ---------------------------------- | ----------------------------------------------------- | --------------------- | ------------------------------------ |
| `ManyToOne`, unidirectional        | `#[ManyToOne]` + `#[JoinColumn]`                      | в текущей таблице     | только owning                        |
| `OneToOne`, unidirectional         | `#[OneToOne]` + `#[JoinColumn]` (с UNIQUE)            | в текущей таблице     | только owning                        |
| `OneToOne`, bidirectional          | `mappedBy` + `inversedBy`                             | у `inversedBy` стороны| owning + inverse                     |
| `OneToOne`, self-referencing       | targetEntity = self                                   | в этой же таблице     | только owning                        |
| `OneToMany`, bidirectional         | `OneToMany(mappedBy)` + `ManyToOne(inversedBy)`       | у `ManyToOne` стороны | inverse + owning                     |
| `OneToMany`, unidirectional w/ JT  | `OneToMany` + `#[JoinTable]`                          | в join-таблице (UNIQUE)| только owning                       |
| `OneToMany`, self-referencing      | как bidirectional, targetEntity = self                | в этой же таблице     | inverse + owning                     |
| `ManyToMany`, unidirectional       | `#[ManyToMany]` + `#[JoinTable]`                      | в join-таблице        | только owning                        |
| `ManyToMany`, bidirectional        | `inversedBy` + `mappedBy`, `JoinTable` на owning      | в join-таблице        | owning + inverse                     |
| `ManyToMany`, self-referencing     | как unidirectional, targetEntity = self               | в join-таблице        | только owning                        |

---

## 15. Главные правила Association Mapping

```text
✓ Owning side держит FK → её изменения попадают в БД.
✓ inversedBy — на owning side (та, где FK).
✓ mappedBy — на inverse side (зеркало).
✓ Type-hint коллекции — Collection, не ArrayCollection.
✓ Инициализируй коллекции в конструкторе ($this->x = new ArrayCollection()).
✓ В ManyToMany #[JoinTable] и #[JoinColumn] — ТОЛЬКО на owning side.
✓ Делай симметричные add/remove методы для двусторонней синхронизации.

✗ Не меняй только inverse side — изменения не сохранятся.
✗ Не используй ArrayCollection как type-hint свойства.
✗ Не забывай инициализировать коллекции — иначе NPE.
✗ Не лепи bidirectional если вторая сторона реально не нужна — это лишний код.
```

---

## 16. Связанные темы (для углубления)

| Тема                                    | Где почитать                                                   |
| --------------------------------------- | -------------------------------------------------------------- |
| Owning vs Inverse side подробно         | `reference/unitofwork-associations.html`                       |
| Composite & Foreign Keys как PK         | `tutorials/composite-primary-keys.html`                        |
| Indexed Associations (`INDEX BY`)       | `tutorials/working-with-indexed-associations.html`             |
| Extra Lazy Associations (`fetch=EXTRA_LAZY`) | `tutorials/extra-lazy-associations.html`                  |
| Ordering To-Many (`#[OrderBy]`)         | `tutorials/ordered-associations.html`                          |
| Working with Associations (add/remove/cascade/orphanRemoval) | `reference/working-with-associations.html` |