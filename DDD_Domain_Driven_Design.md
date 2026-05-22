# DDD — Domain-Driven Design

**Domain-Driven Design** (предметно-ориентированное проектирование) — подход к разработке сложного ПО, при котором **бизнес-логика (предметная область, domain)** ставится в центр архитектуры. Сформулирован Эриком Эвансом в книге "Domain-Driven Design: Tackling Complexity in the Heart of Software" (2003).

**Главная идея:** код должен говорить на языке бизнеса. Разработчики и эксперты предметной области используют **один и тот же язык** (Ubiquitous Language), а структура кода отражает структуру бизнеса.

DDD имеет две части:
- **Стратегическая** — как делить большую систему на куски (Bounded Contexts, контекстные карты)
- **Тактическая** — какими блоками строить код внутри одного куска (Entity, Value Object, Aggregate, Repository, Service, Event)

---

## Когда применять DDD

```text
✓ Сложная бизнес-логика (банковские системы, ERP, медицина, логистика, биллинг)
✓ Долгоживущие проекты, где правила бизнеса часто меняются
✓ Большая команда, нужна изоляция модулей
✓ Бизнес-эксперты активно участвуют в разработке

✗ CRUD-приложения без сложной логики
✗ Прототипы и одноразовые скрипты
✗ Простые админки и лендинги
✗ Когда команда не готова инвестировать в моделирование
```

DDD — не серебряная пуля. Это **инструмент для борьбы со сложностью**. Если сложности нет — DDD её создаёт сам.

---

## Стратегические паттерны

### 1. Ubiquitous Language (Единый язык)

Один и тот же словарь у программистов, продактов, аналитиков и экспертов бизнеса.

```text
✗ ПЛОХО:
   Аналитик: "Когда клиент оформляет покупку..."
   Программист: "У нас есть Order с status=NEW, потом меняем на PAID..."

✓ ХОРОШО:
   Все говорят одинаково: "Клиент размещает Заказ. Заказ может быть
   Размещён, Оплачен, Отгружен, Доставлен, Отменён."
   В коде: class Order, метод $order->place(), $order->pay(), $order->ship().
```

В коде имена классов, методов и переменных — это и есть слова бизнеса.

### 2. Bounded Context (Ограниченный контекст)

Граница, внутри которой один и тот же термин имеет одно конкретное значение.

```text
Пример: слово "Клиент" в разных подсистемах одной компании:

  📦 Sales Context              → Клиент = тот, кто покупает
                                  поля: ФИО, email, телефон, скидка

  💰 Billing Context            → Клиент = плательщик
                                  поля: ИНН, расчётный счёт, лимит кредита

  📞 Support Context            → Клиент = автор тикета
                                  поля: id, имя, тариф поддержки

ОДИН термин — РАЗНЫЕ модели. И это нормально.
Bounded Context разделяет их и не даёт смешать.
```

В коде это обычно отдельные модули / неймспейсы / даже отдельные сервисы:

```text
src/
├── Sales/          ← один bounded context
│   └── Domain/
│       └── Customer.php
├── Billing/        ← другой bounded context
│   └── Domain/
│       └── Customer.php   ← это ДРУГОЙ Customer, и это ок
└── Support/
    └── Domain/
        └── Customer.php
```

### 3. Context Map (Контекстная карта)

Описание, как bounded contexts взаимодействуют друг с другом. Типовые отношения:

| Отношение                  | Что значит                                                    |
| -------------------------- | ------------------------------------------------------------- |
| **Partnership**            | Две команды вместе развивают оба контекста                    |
| **Customer–Supplier**      | Один контекст потребитель, другой поставщик API               |
| **Conformist**             | Контекст А слепо использует модель Б (нет влияния на Б)       |
| **Anti-Corruption Layer**  | Слой-переводчик, защищающий модель А от чужой модели          |
| **Open Host Service**      | Контекст публикует API, которым пользуются все                |
| **Published Language**     | Контекст публикует формат данных (event schema, JSON-схема)   |
| **Shared Kernel**          | Контексты делят общий код (опасно — связывает команды)        |
| **Separate Ways**          | Контексты не общаются вообще                                  |

**Anti-Corruption Layer** — частый и важный паттерн: когда подключаешься к внешней системе (например, к legacy-API), не тащишь её модель внутрь — пишешь "переводчик".

```php
<?php
// Anti-Corruption Layer для интеграции с legacy CRM
namespace App\Sales\Infrastructure\CrmAdapter;

use App\Sales\Domain\Customer;

class LegacyCrmCustomerAdapter
{
    public function __construct(private LegacyCrmClient $crm) {}

    public function findById(int $id): Customer
    {
        // Получаем "сырую" модель из чужой системы
        $raw = $this->crm->getCustomerRaw($id);
        // raw->cli_nm, raw->cli_birth_dt, raw->status_cd  — ужас

        // Переводим в НАШУ доменную модель — наш язык, наши инварианты
        return new Customer(
            id: $raw->cli_id,
            name: $raw->cli_nm,
            email: $raw->cli_email,
        );
    }
}
```

### 4. Core / Supporting / Generic Subdomains

Не все части системы одинаково важны. DDD делит их на:

- **Core Domain** — то, ради чего система существует, конкурентное преимущество. Сюда вкладываем больше всего сил.
- **Supporting Subdomain** — нужно, но не уникально (например, биллинг для онлайн-школы).
- **Generic Subdomain** — типовое, можно купить готовое (аутентификация, отправка email).

```text
Пример: маркетплейс ручной работы
  Core    : подбор товаров, рейтинг мастеров, рекомендации
  Support : каталог категорий, корзина
  Generic : аутентификация (бери Auth0), email (бери Mailgun)
```

---

## Тактические паттерны

Кирпичики, из которых строится код внутри одного bounded context. Обычно структура такая:

```text
src/Order/                          ← один Bounded Context
├── Domain/                         ← Чистая бизнес-логика (без БД, без HTTP)
│   ├── Order.php                   ← Entity / Aggregate Root
│   ├── OrderLine.php               ← Entity внутри агрегата
│   ├── Money.php                   ← Value Object
│   ├── OrderStatus.php             ← Value Object (enum)
│   ├── OrderId.php                 ← Value Object
│   ├── OrderRepository.php         ← Интерфейс репозитория
│   ├── Event/
│   │   └── OrderPlacedEvent.php   ← Domain Event
│   └── Service/
│       └── PricingService.php      ← Domain Service (если логика не в Entity)
│
├── Application/                    ← Оркестрация use-case'ов
│   ├── Command/
│   │   ├── PlaceOrderCommand.php
│   │   └── PlaceOrderHandler.php
│   └── Query/
│       └── GetOrderHandler.php
│
└── Infrastructure/                 ← Конкретные технологии (БД, HTTP, очереди)
    ├── Persistence/
    │   └── DoctrineOrderRepository.php
    └── Controller/
        └── OrderController.php
```

### 1. Value Object — значение без идентичности

Объект, у которого важно **значение**, а не "кто он". Два VO с одинаковыми полями — равны.

**Свойства:** иммутабельный, сравнивается по значению, не имеет ID.

```php
<?php
namespace App\Order\Domain;

// readonly + final = классический Value Object в PHP 8.2+
final readonly class Money
{
    public function __construct(
        public int $amountInCents,    // храним в копейках, чтобы избежать float-проблем
        public string $currency,
    ) {
        // Инвариант — гарантирует, что объект всегда в валидном состоянии
        if ($amountInCents < 0) {
            throw new \InvalidArgumentException('Money не может быть отрицательной');
        }
        if (!in_array($currency, ['USD', 'EUR', 'RUB'], true)) {
            throw new \InvalidArgumentException("Неизвестная валюта: $currency");
        }
    }

    // Поведение — не set'еры, а БИЗНЕС-методы.
    // Возвращают НОВЫЙ объект (иммутабельность).
    public function add(Money $other): Money
    {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Нельзя складывать разные валюты');
        }
        return new Money($this->amountInCents + $other->amountInCents, $this->currency);
    }

    public function equals(Money $other): bool
    {
        return $this->amountInCents === $other->amountInCents
            && $this->currency === $other->currency;
    }
}

// Использование:
$price = new Money(1500, 'USD');     // $15.00
$tax   = new Money(150, 'USD');      // $1.50
$total = $price->add($tax);          // НОВЫЙ Money($16.50, 'USD'), $price не изменился
```

**Типичные VO:** `Money`, `Email`, `Address`, `DateRange`, `Coordinates`, `OrderId`.

### 2. Entity — объект с идентичностью

То, что важно отличать от других экземпляров **по ID**, даже если все остальные поля одинаковы.

```php
<?php
namespace App\Order\Domain;

final class Customer
{
    public function __construct(
        private readonly CustomerId $id,    // <- идентичность
        private string $name,                // эти поля могут меняться
        private Email $email,
    ) {}

    // Два покупателя могут иметь одинаковое имя и email — это разные сущности
    // (например, тёзки), идентичность определяется ID, а не полями.
    public function equals(Customer $other): bool
    {
        return $this->id->equals($other->id);
    }

    public function changeEmail(Email $newEmail): void
    {
        // Бизнес-логика внутри сущности, а не в "сервисе"
        if ($newEmail->equals($this->email)) {
            return;
        }
        $this->email = $newEmail;
    }
}
```

### 3. Aggregate и Aggregate Root

**Aggregate** — кластер связанных Entity и VO, которые меняются вместе и обладают общим инвариантом (правилом, которое нельзя нарушить).

**Aggregate Root** — главная сущность кластера. Только через неё можно изменять состояние агрегата извне.

**Правила:**
1. Снаружи агрегата ссылка только на корень — не на внутренние Entity
2. Все инварианты агрегата проверяются внутри
3. Изменения сохраняются транзакционно (весь агрегат целиком)
4. Между агрегатами — связь по ID, не по объекту

```php
<?php
namespace App\Order\Domain;

// AGGREGATE ROOT — точка входа для всех операций над заказом
final class Order
{
    /** @var OrderLine[] */
    private array $lines = [];
    private OrderStatus $status;

    public function __construct(
        private readonly OrderId $id,
        private readonly CustomerId $customerId,  // ссылка на ДРУГОЙ агрегат по ID
    ) {
        $this->status = OrderStatus::Draft;
    }

    // === Бизнес-операции (поведение!) ===

    public function addLine(ProductId $productId, int $quantity, Money $unitPrice): void
    {
        // Инвариант: нельзя добавлять позиции в уже размещённый заказ
        if ($this->status !== OrderStatus::Draft) {
            throw new \DomainException('Нельзя менять размещённый заказ');
        }
        if ($quantity <= 0) {
            throw new \DomainException('Количество должно быть положительным');
        }

        $this->lines[] = new OrderLine($productId, $quantity, $unitPrice);
    }

    public function place(): void
    {
        // Инвариант: пустой заказ нельзя разместить
        if (empty($this->lines)) {
            throw new \DomainException('Заказ не может быть пустым');
        }
        if ($this->status !== OrderStatus::Draft) {
            throw new \DomainException('Заказ уже размещён');
        }

        $this->status = OrderStatus::Placed;
        // Тут можно записать domain event (см. ниже)
    }

    public function total(): Money
    {
        return array_reduce(
            $this->lines,
            fn(Money $acc, OrderLine $line) => $acc->add($line->subtotal()),
            new Money(0, 'USD'),
        );
    }

    public function id(): OrderId { return $this->id; }
}

// Entity ВНУТРИ агрегата — наружу не торчит
final class OrderLine
{
    public function __construct(
        private readonly ProductId $productId,
        private readonly int $quantity,
        private readonly Money $unitPrice,
    ) {}

    public function subtotal(): Money
    {
        return new Money($this->unitPrice->amountInCents * $this->quantity, $this->unitPrice->currency);
    }
}
```

### 4. Repository — коллекция агрегатов

Интерфейс, который "делает вид", что агрегаты лежат в памяти. Скрывает БД.

```php
<?php
namespace App\Order\Domain;

// Интерфейс — в Domain (он часть бизнес-языка)
interface OrderRepository
{
    public function findById(OrderId $id): ?Order;
    public function save(Order $order): void;
    public function nextIdentity(): OrderId;  // фабрика новых ID
}
```

Реализация — в Infrastructure (Doctrine, PDO, in-memory для тестов):

```php
<?php
namespace App\Order\Infrastructure\Persistence;

use App\Order\Domain\{Order, OrderId, OrderRepository};
use Doctrine\ORM\EntityManagerInterface;

final class DoctrineOrderRepository implements OrderRepository
{
    public function __construct(private EntityManagerInterface $em) {}

    public function findById(OrderId $id): ?Order
    {
        return $this->em->find(Order::class, $id->value);
    }

    public function save(Order $order): void
    {
        $this->em->persist($order);
        $this->em->flush();
    }

    public function nextIdentity(): OrderId
    {
        return new OrderId(\Symfony\Component\Uid\Uuid::v4()->toString());
    }
}
```

### 5. Domain Service — логика, которой не место в Entity

Когда логика касается **нескольких агрегатов** или **не принадлежит ни одному из них**.

```php
<?php
namespace App\Order\Domain\Service;

use App\Order\Domain\{Order, Customer};

// Расчёт скидки зависит от уровня клиента И содержимого заказа.
// Логика не "принадлежит" ни Order, ни Customer — отдельный domain service.
final class DiscountCalculator
{
    public function calculate(Order $order, Customer $customer): Money
    {
        $total = $order->total();
        $rate  = match (true) {
            $customer->isVip()    => 0.15,
            $customer->isLoyal()  => 0.05,
            default               => 0.0,
        };
        return new Money((int)($total->amountInCents * $rate), $total->currency);
    }
}
```

> **Важно:** Domain Service ≠ Application Service. Domain Service содержит **бизнес-правила**. Application Service — **оркестрирует** работу (см. ниже).

### 6. Domain Event — то, что случилось в домене

Факт о произошедшем бизнес-событии. Прошедшее время в имени.

```php
<?php
namespace App\Order\Domain\Event;

use App\Order\Domain\OrderId;

// Иммутабельный объект с фактом события
final readonly class OrderPlacedEvent
{
    public function __construct(
        public OrderId $orderId,
        public \DateTimeImmutable $occurredAt,
    ) {}
}
```

Агрегат регистрирует событие у себя, наружу его собирают позже:

```php
final class Order
{
    /** @var object[] */
    private array $recordedEvents = [];

    public function place(): void
    {
        // ... бизнес-логика ...
        $this->status = OrderStatus::Placed;

        // Записываем событие
        $this->recordedEvents[] = new OrderPlacedEvent($this->id, new \DateTimeImmutable());
    }

    public function pullEvents(): array
    {
        $events = $this->recordedEvents;
        $this->recordedEvents = [];
        return $events;
    }
}
```

После сохранения агрегата Application Service публикует события — подписчики реагируют (отправляют email, уведомления, обновляют read-модели и т.п.).

### 7. Application Service — оркестратор use-case'а

Тонкий слой между внешним миром (HTTP, CLI, очередь) и доменом. Один метод = один use-case.

```php
<?php
namespace App\Order\Application\Command;

use App\Order\Domain\{Order, OrderRepository, CustomerId, ProductId, Money};
use Psr\EventDispatcher\EventDispatcherInterface;

// 1) Command — данные для use-case (DTO)
final readonly class PlaceOrderCommand
{
    public function __construct(
        public string $customerId,
        public array $items, // [{productId, quantity, priceInCents, currency}]
    ) {}
}

// 2) Handler — оркестратор use-case'а
final class PlaceOrderHandler
{
    public function __construct(
        private OrderRepository $orders,
        private EventDispatcherInterface $events,
    ) {}

    public function __invoke(PlaceOrderCommand $cmd): string
    {
        // Application Service:
        //   - НЕ содержит бизнес-логики (она в Order::place())
        //   - Координирует: достать → изменить → сохранить → опубликовать события
        //   - Управляет транзакциями

        $order = new Order(
            $this->orders->nextIdentity(),
            new CustomerId($cmd->customerId),
        );

        foreach ($cmd->items as $item) {
            $order->addLine(
                new ProductId($item['productId']),
                $item['quantity'],
                new Money($item['priceInCents'], $item['currency']),
            );
        }

        $order->place(); // ← здесь живёт бизнес-логика и инварианты

        $this->orders->save($order);

        // Публикуем события после успешного сохранения
        foreach ($order->pullEvents() as $event) {
            $this->events->dispatch($event);
        }

        return $order->id()->value;
    }
}
```

### 8. Factory — создание сложных агрегатов

Когда создание агрегата требует много логики/валидации — выносим в фабрику.

```php
<?php
final class OrderFactory
{
    public function __construct(private OrderRepository $orders) {}

    public function createFromBasket(Customer $customer, Basket $basket): Order
    {
        $order = new Order($this->orders->nextIdentity(), $customer->id());
        foreach ($basket->items() as $item) {
            $order->addLine($item->productId, $item->quantity, $item->price);
        }
        return $order;
    }
}
```

---

## Сводная таблица тактических паттернов

| Паттерн              | Что это                                          | Пример                          |
| -------------------- | ------------------------------------------------ | ------------------------------- |
| **Value Object**     | Иммутабельный объект, равенство по значению      | `Money`, `Email`, `Address`     |
| **Entity**           | Объект с идентичностью (ID), меняется во времени | `Customer`, `Product`           |
| **Aggregate**        | Кластер Entity+VO с общим инвариантом            | `Order` + `OrderLine` + `Money` |
| **Aggregate Root**   | Точка входа в агрегат                            | `Order`                         |
| **Repository**       | Коллекция агрегатов, прячет БД                   | `OrderRepository`               |
| **Domain Service**   | Логика, не вмещающаяся в одну сущность           | `DiscountCalculator`            |
| **Application Service** | Use-case-оркестратор                          | `PlaceOrderHandler`             |
| **Domain Event**     | Факт того, что произошло                         | `OrderPlacedEvent`              |
| **Factory**          | Создание сложных агрегатов                       | `OrderFactory`                  |

---

## Архитектурный слой (типовая укладка)

DDD часто сочетают с **гексагональной архитектурой** / **Clean Architecture** / **Onion**. Главное правило — зависимости направлены ВНУТРЬ:

```text
┌──────────────────────────────────────────────────────────────┐
│ Infrastructure (HTTP, БД, очереди, внешние API)               │
│   ↓ зависит от                                                │
│ ┌──────────────────────────────────────────────────────────┐ │
│ │ Application (use-case'ы, command/query handlers)          │ │
│ │   ↓ зависит от                                            │ │
│ │ ┌──────────────────────────────────────────────────────┐ │ │
│ │ │ Domain (Entity, VO, Aggregate, Repository-интерфейс,  │ │ │
│ │ │         Domain Service, Domain Event)                 │ │ │
│ │ │ — НЕ ЗНАЕТ про БД, HTTP, фреймворк                    │ │ │
│ │ └──────────────────────────────────────────────────────┘ │ │
│ └──────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────┘

ПРАВИЛО: Domain — самое глубокое ядро. На него можно ссылаться сверху,
       НО ОН НЕ ССЫЛАЕТСЯ НИ НА ЧТО ИЗВНЕ. Никаких use Doctrine\... в Domain!
```

В Domain — только PHP и больше ничего. Никаких аннотаций ORM, никаких HTTP-классов. Это и значит "богатая модель домена" — изолированная и тестируемая.

---

## Минимальный полный пример (всё вместе)

```php
<?php
// ============ DOMAIN ============
namespace App\Order\Domain;

final readonly class OrderId { public function __construct(public string $value) {} }
final readonly class CustomerId { public function __construct(public string $value) {} }
final readonly class ProductId { public function __construct(public string $value) {} }

final readonly class Money
{
    public function __construct(public int $amountInCents, public string $currency) {
        if ($amountInCents < 0) throw new \InvalidArgumentException();
    }
    public function add(Money $o): Money {
        return new Money($this->amountInCents + $o->amountInCents, $this->currency);
    }
}

enum OrderStatus: string { case Draft = 'draft'; case Placed = 'placed'; }

final class Order
{
    private array $lines = [];
    private OrderStatus $status = OrderStatus::Draft;
    private array $events = [];

    public function __construct(
        public readonly OrderId $id,
        public readonly CustomerId $customerId,
    ) {}

    public function addLine(ProductId $p, int $qty, Money $price): void {
        if ($this->status !== OrderStatus::Draft) throw new \DomainException();
        $this->lines[] = ['p' => $p, 'qty' => $qty, 'price' => $price];
    }

    public function place(): void {
        if (empty($this->lines)) throw new \DomainException('Empty order');
        $this->status = OrderStatus::Placed;
        $this->events[] = new Event\OrderPlacedEvent($this->id, new \DateTimeImmutable());
    }

    public function pullEvents(): array {
        $e = $this->events; $this->events = []; return $e;
    }
}

namespace App\Order\Domain\Event;
final readonly class OrderPlacedEvent {
    public function __construct(
        public \App\Order\Domain\OrderId $orderId,
        public \DateTimeImmutable $at,
    ) {}
}

namespace App\Order\Domain;
interface OrderRepository {
    public function save(Order $o): void;
    public function nextIdentity(): OrderId;
}

// ============ APPLICATION ============
namespace App\Order\Application;

use App\Order\Domain\{Order, OrderRepository, CustomerId, ProductId, Money};
use Psr\EventDispatcher\EventDispatcherInterface;

final readonly class PlaceOrderCommand {
    public function __construct(public string $customerId, public array $items) {}
}

final class PlaceOrderHandler {
    public function __construct(
        private OrderRepository $orders,
        private EventDispatcherInterface $events,
    ) {}

    public function __invoke(PlaceOrderCommand $cmd): string {
        $order = new Order($this->orders->nextIdentity(), new CustomerId($cmd->customerId));
        foreach ($cmd->items as $i) {
            $order->addLine(new ProductId($i['productId']), $i['qty'], new Money($i['price'], 'USD'));
        }
        $order->place();
        $this->orders->save($order);
        foreach ($order->pullEvents() as $e) $this->events->dispatch($e);
        return $order->id->value;
    }
}

// ============ INFRASTRUCTURE ============
namespace App\Order\Infrastructure;

use App\Order\Domain\{Order, OrderId, OrderRepository};

// Доктриновская реализация репозитория, например
final class InMemoryOrderRepository implements OrderRepository {
    private array $storage = [];
    public function save(Order $o): void { $this->storage[$o->id->value] = $o; }
    public function nextIdentity(): OrderId { return new OrderId(uniqid('ord_', true)); }
}
```

---

## DDD и CQRS

Часто DDD идёт в паре с **CQRS** (Command Query Responsibility Segregation):

```text
WRITE-сторона (Commands)    → проходит через домен:
                              Controller → Command → Handler → Aggregate → Repo

READ-сторона (Queries)      → может миновать домен:
                              Controller → SQL → DTO → JSON

WRITE-модель оптимизирована под бизнес-инварианты (агрегаты, события).
READ-модель оптимизирована под запросы (денормализованные view, projections).
```

Это разделение упрощает обе стороны, но не обязательно — в простых случаях команды и запросы могут жить вместе.

---

## Главные правила DDD

```text
✓ Говорите с бизнесом на одном языке. Не "status_code = 1", а "order placed".
✓ Сложный домен — в центре. Технологии вокруг.
✓ Богатая модель: поведение в Entity/VO, а не в "сервисах".
✓ Value Object для всего, что не имеет идентичности (Money, Email).
✓ Один aggregate = одна транзакция. Между агрегатами — через события или ID.
✓ Repository = коллекция в памяти, а не "DAO для CRUD".
✓ Domain не должен знать про БД, HTTP, фреймворк.
✓ Application Service — тонкий, без бизнес-логики.
✓ Domain Event — прошедшее время: OrderPlaced, не PlaceOrder.

✗ Не применяй DDD в CRUD-приложениях — будет оверинжиниринг.
✗ Не делай "анемичную модель" (геттеры/сеттеры + всё в Service) — это не DDD.
✗ Не тащи бизнес-логику в контроллер.
✗ Не зови по объектам между агрегатами — связывай по ID.
✗ Не клади аннотации ORM прямо в Entity (или будь готов к компромиссу).
```

---

## С чего начать

1. **Сначала Ubiquitous Language** — поговори с бизнесом, выпиши термины
2. **Определи bounded contexts** — нарисуй контекстную карту
3. **Начни с одного core-контекста** — самой ценной части системы
4. **Моделируй вслух** — event storming, бумага, доска, что угодно
5. **Богатая модель** — поведение в Entity, не в сервисах
6. **Тесты на домене** — без БД, без HTTP, в памяти. Они должны быть быстрыми
7. **Постепенно вводи Domain Events**, когда видишь "после X должно случиться Y"

---

## Что почитать дальше

- **Eric Evans** — "Domain-Driven Design: Tackling Complexity in the Heart of Software" (синяя книга)
- **Vaughn Vernon** — "Implementing Domain-Driven Design" (красная книга, практичнее)
- **Vaughn Vernon** — "Domain-Driven Design Distilled" (короткое введение)
- **Scott Wlaschin** — "Domain Modeling Made Functional" (DDD на F#, идеи переносимы)
- Сайты: martinfowler.com, dddcommunity.org, paulmcallister.com/ddd