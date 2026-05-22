# SOLID — пять принципов объектно-ориентированного дизайна

**SOLID** — акроним пяти принципов проектирования ООП, сформулированных Робертом Мартином (Uncle Bob). Они помогают писать код, который легко **поддерживать**, **расширять** и **тестировать**.

| Буква | Принцип                            | Суть в одну строку                                                       |
| ----- | ---------------------------------- | ------------------------------------------------------------------------ |
| **S** | Single Responsibility Principle    | Один класс — одна причина для изменения                                  |
| **O** | Open/Closed Principle              | Открыт для расширения, закрыт для модификации                            |
| **L** | Liskov Substitution Principle      | Наследник должен заменять родителя без поломки кода                      |
| **I** | Interface Segregation Principle    | Много маленьких интерфейсов лучше одного огромного                       |
| **D** | Dependency Inversion Principle     | Зависимости — от абстракций, не от конкретных реализаций                 |

---

## S — Single Responsibility Principle (Принцип единственной ответственности)

> **Класс должен иметь только одну причину для изменения.**

Каждый класс отвечает за одну часть функциональности приложения, и эта функциональность полностью инкапсулирована в нём.

### ✗ Плохо

```php
<?php
// Этот класс делает СЛИШКОМ много: хранит данные, ходит в БД и шлёт email.
// 3 причины для изменения → нарушение SRP.
class User
{
    public function __construct(
        private string $email,
        private string $name,
    ) {}

    public function save(): void
    {
        // Логика работы с БД
        $pdo = new PDO('mysql:host=localhost', 'user', 'pass');
        $pdo->prepare('INSERT INTO users (email, name) VALUES (?, ?)')
            ->execute([$this->email, $this->name]);
    }

    public function sendWelcomeEmail(): void
    {
        // Логика отправки email
        mail($this->email, 'Welcome!', "Hello, $this->name!");
    }
}
```

### ✓ Хорошо

```php
<?php
// 1) Сущность — только данные пользователя
class User
{
    public function __construct(
        public readonly string $email,
        public readonly string $name,
    ) {}
}

// 2) Репозиторий — только работа с БД
class UserRepository
{
    public function __construct(private PDO $pdo) {}

    public function save(User $user): void
    {
        $this->pdo->prepare('INSERT INTO users (email, name) VALUES (?, ?)')
            ->execute([$user->email, $user->name]);
    }
}

// 3) Сервис рассылки — только email
class WelcomeEmailSender
{
    public function send(User $user): void
    {
        mail($user->email, 'Welcome!', "Hello, $user->name!");
    }
}
```

Теперь у каждого класса **одна причина для изменения**: схема БД изменилась → правишь только `UserRepository`; шаблон письма поменялся → только `WelcomeEmailSender`.

---

## O — Open/Closed Principle (Принцип открытости/закрытости)

> **Программные сущности должны быть открыты для расширения, но закрыты для модификации.**

Добавление новой функциональности не должно требовать правки уже работающего кода — расширяй через наследование или композицию.

### ✗ Плохо

```php
<?php
// При добавлении нового типа фигуры приходится лезть в этот класс и
// дописывать ещё один if. Это и есть "модификация".
class AreaCalculator
{
    public function calculate(array $shapes): float
    {
        $total = 0.0;
        foreach ($shapes as $shape) {
            if ($shape instanceof Circle) {
                $total += M_PI * $shape->radius ** 2;
            } elseif ($shape instanceof Square) {
                $total += $shape->side ** 2;
            }
            // А завтра — Triangle? Опять if!
        }
        return $total;
    }
}
```

### ✓ Хорошо

```php
<?php
// Контракт: любая фигура умеет считать свою площадь
interface Shape
{
    public function area(): float;
}

class Circle implements Shape
{
    public function __construct(public readonly float $radius) {}
    public function area(): float
    {
        return M_PI * $this->radius ** 2;
    }
}

class Square implements Shape
{
    public function __construct(public readonly float $side) {}
    public function area(): float
    {
        return $this->side ** 2;
    }
}

// Калькулятор закрыт для модификации — менять его не нужно.
// Чтобы добавить Triangle, достаточно создать новый класс,
// реализующий Shape — расширение без правки существующего кода.
class AreaCalculator
{
    public function calculate(array $shapes): float
    {
        return array_sum(array_map(fn(Shape $s) => $s->area(), $shapes));
    }
}
```

---

## L — Liskov Substitution Principle (Принцип подстановки Барбары Лисков)

> **Объекты в программе должны быть заменяемы экземплярами их подтипов без изменения корректности работы программы.**

Если код работает с классом `A`, он должен работать с любым наследником `A` без сюрпризов. Наследник **не должен** усиливать предусловия, ослаблять постусловия или ломать инварианты родителя.

### ✗ Плохо — классический пример с квадратом и прямоугольником

```php
<?php
class Rectangle
{
    public function __construct(
        protected float $width,
        protected float $height,
    ) {}

    public function setWidth(float $w): void  { $this->width = $w; }
    public function setHeight(float $h): void { $this->height = $h; }
    public function area(): float             { return $this->width * $this->height; }
}

// "Квадрат — частный случай прямоугольника" в математике, но НЕ в ООП!
class Square extends Rectangle
{
    public function setWidth(float $w): void
    {
        $this->width = $w;
        $this->height = $w; // Меняем оба — иначе квадрат сломается
    }

    public function setHeight(float $h): void
    {
        $this->width = $h;
        $this->height = $h;
    }
}

// Код, рассчитанный на Rectangle, ломается с Square:
function test(Rectangle $r): void
{
    $r->setWidth(5);
    $r->setHeight(10);
    assert($r->area() === 50.0); // ✗ Для Square вернёт 100, а не 50!
}
```

### ✓ Хорошо

```php
<?php
// Делаем фигуры неизменяемыми (readonly) и не наследуем одну от другой.
// Квадрат и прямоугольник — два независимых класса с общим интерфейсом.
interface Shape
{
    public function area(): float;
}

final class Rectangle implements Shape
{
    public function __construct(
        public readonly float $width,
        public readonly float $height,
    ) {}
    public function area(): float { return $this->width * $this->height; }
}

final class Square implements Shape
{
    public function __construct(public readonly float $side) {}
    public function area(): float { return $this->side ** 2; }
}
```

### Ещё одно классическое нарушение — `Bird`/`Penguin`

```php
<?php
// ✗ Плохо
class Bird
{
    public function fly(): void { /* ... */ }
}

class Penguin extends Bird
{
    public function fly(): void
    {
        throw new LogicException('Пингвины не летают!');
        // Нарушение LSP: код, работающий с Bird, упадёт на Penguin.
    }
}

// ✓ Хорошо — разделить иерархию по способностям
interface Bird {}
interface FlyingBird extends Bird
{
    public function fly(): void;
}

class Sparrow implements FlyingBird
{
    public function fly(): void { /* ... */ }
}

class Penguin implements Bird {} // Пингвин — птица, но не летающая
```

---

## I — Interface Segregation Principle (Принцип разделения интерфейсов)

> **Клиенты не должны зависеть от методов, которые они не используют.**

Лучше много маленьких узкоспециализированных интерфейсов, чем один "толстый". Иначе классы вынуждены реализовывать методы, которые им не нужны.

### ✗ Плохо

```php
<?php
// "Жирный" интерфейс — все, кто реализует, обязан написать ВСЁ.
interface Worker
{
    public function work(): void;
    public function eat(): void;
    public function sleep(): void;
}

class Human implements Worker
{
    public function work(): void  { /* работаю */ }
    public function eat(): void   { /* ем */ }
    public function sleep(): void { /* сплю */ }
}

class Robot implements Worker
{
    public function work(): void  { /* работаю */ }

    // Робот не ест и не спит — вынуждены кидать исключения или оставлять пустыми.
    public function eat(): void   { throw new LogicException('Не ем'); }
    public function sleep(): void { throw new LogicException('Не сплю'); }
}
```

### ✓ Хорошо

```php
<?php
// Разделили на узкие интерфейсы — каждый класс реализует только нужное.
interface Workable
{
    public function work(): void;
}

interface Feedable
{
    public function eat(): void;
}

interface Sleepable
{
    public function sleep(): void;
}

class Human implements Workable, Feedable, Sleepable
{
    public function work(): void  { /* ... */ }
    public function eat(): void   { /* ... */ }
    public function sleep(): void { /* ... */ }
}

class Robot implements Workable
{
    public function work(): void { /* ... */ }
    // Никаких пустых методов — реализует только то, что умеет.
}
```

---

## D — Dependency Inversion Principle (Принцип инверсии зависимостей)

> **Модули верхнего уровня не должны зависеть от модулей нижнего уровня. Оба должны зависеть от абстракций. Абстракции не должны зависеть от деталей — детали должны зависеть от абстракций.**

В коде это значит: type-hint на **интерфейс**, не на конкретный класс. Тогда реализацию можно подменить (мок в тестах, другой драйвер, другой провайдер).

### ✗ Плохо

```php
<?php
// Конкретная реализация
class MySQLConnection
{
    public function query(string $sql): array { /* ... */ }
}

// "Верхнеуровневый" класс жёстко привязан к MySQLConnection.
// Захотим SQLite или мок в тестах — придётся менять UserService.
class UserService
{
    private MySQLConnection $db;

    public function __construct()
    {
        $this->db = new MySQLConnection(); // ✗ зашитая зависимость
    }

    public function findById(int $id): array
    {
        return $this->db->query("SELECT * FROM users WHERE id = $id");
    }
}
```

### ✓ Хорошо

```php
<?php
// Абстракция — интерфейс
interface DatabaseConnection
{
    public function query(string $sql): array;
}

// Реализации зависят от абстракции (реализуют интерфейс)
class MySQLConnection implements DatabaseConnection
{
    public function query(string $sql): array { /* ... */ }
}

class SqliteConnection implements DatabaseConnection
{
    public function query(string $sql): array { /* ... */ }
}

// В тестах легко подменить на мок
class InMemoryConnection implements DatabaseConnection
{
    public function query(string $sql): array { return []; }
}

// "Верхнеуровневый" класс зависит от абстракции, а не от конкретики
class UserService
{
    // Constructor injection + type-hint на интерфейс
    public function __construct(private DatabaseConnection $db) {}

    public function findById(int $id): array
    {
        return $this->db->query("SELECT * FROM users WHERE id = $id");
    }
}

// Подмена реализации — через конструктор:
$service = new UserService(new MySQLConnection());
$service = new UserService(new SqliteConnection());
$service = new UserService(new InMemoryConnection()); // для тестов
```

---

## Все пять принципов в одном примере

Маленькая система отправки уведомлений:

```php
<?php

// S — каждый класс делает одно дело
// I — узкие интерфейсы по способностям
// D — зависимости через абстракции

// --- Контракты (абстракции) ---
interface Notification
{
    public function getRecipient(): string;
    public function getMessage(): string;
}

interface NotificationChannel
{
    public function send(Notification $notification): void;
}

// --- Конкретные уведомления (S) ---
final class WelcomeNotification implements Notification
{
    public function __construct(
        private readonly string $recipient,
        private readonly string $userName,
    ) {}

    public function getRecipient(): string { return $this->recipient; }
    public function getMessage(): string   { return "Welcome, {$this->userName}!"; }
}

// --- Каналы доставки (O — добавляем новые каналы без правки старых) ---
final class EmailChannel implements NotificationChannel
{
    public function send(Notification $n): void
    {
        mail($n->getRecipient(), 'Notification', $n->getMessage());
    }
}

final class SmsChannel implements NotificationChannel
{
    public function send(Notification $n): void { /* отправка SMS */ }
}

// Telegram добавили завтра — просто новый класс, существующий код не трогаем
final class TelegramChannel implements NotificationChannel
{
    public function send(Notification $n): void { /* отправка через Bot API */ }
}

// --- Сервис, зависящий от АБСТРАКЦИЙ (D) ---
final class Notifier
{
    /** @param NotificationChannel[] $channels */
    public function __construct(private readonly array $channels) {}

    public function notify(Notification $notification): void
    {
        foreach ($this->channels as $channel) {
            // L: любой NotificationChannel здесь работает корректно,
            // ни один наследник не ломает контракт.
            $channel->send($notification);
        }
    }
}

// --- Сборка ---
$notifier = new Notifier([
    new EmailChannel(),
    new SmsChannel(),
    new TelegramChannel(),
]);

$notifier->notify(new WelcomeNotification('alice@example.com', 'Alice'));
```

В этом коде:
- **S**: `WelcomeNotification` хранит данные, `EmailChannel` отправляет письма, `Notifier` оркестрирует — у каждого одна ответственность.
- **O**: новый канал добавляется отдельным классом, существующие не правятся.
- **L**: любая реализация `NotificationChannel` подставима — `Notifier` работает одинаково.
- **I**: `Notification` и `NotificationChannel` — два узких интерфейса, не один "жирный".
- **D**: `Notifier` зависит от интерфейсов, а не от `EmailChannel` напрямую.

---

## Когда не стоит фанатеть

```text
SOLID — это рекомендации, а не догмы.

✓ Применяй, когда:
  • Проект развивается, требования меняются
  • Код будут читать другие люди
  • Нужно покрывать модулями тестами (моки реализуются легко)
  • Бизнес-логика сложная и нужно её изолировать

✗ Не оверинжинирь, когда:
  • Это одноразовый скрипт на 50 строк
  • Прототип, который завтра выкинут
  • Простая утилита без зависимостей

Главная цель — управляемая сложность. Если SOLID её увеличивает —
вместо того чтобы уменьшать, — значит, в данном месте он лишний.
```

---

## Шпаргалка

| Принцип | Спроси у класса/интерфейса                                                |
| ------- | ------------------------------------------------------------------------- |
| **S**   | "Сколько причин у меня поменяться?" Больше одной — раздели.               |
| **O**   | "Чтобы добавить фичу, нужно править существующий код?" → введи абстракцию.|
| **L**   | "Мои наследники могут заменить меня без сюрпризов?" Нет — переделай иерархию. |
| **I**   | "Все ли клиенты моего интерфейса используют ВСЕ его методы?" Нет — разрежь. |
| **D**   | "Я завишу от конкретного класса в типах?" Замени на интерфейс.            |