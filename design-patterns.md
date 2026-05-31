# Паттерны проектирования (Design Patterns)

> На основе классификации [refactoring.guru](https://refactoring.guru/ru/design-patterns) с примерами на PHP.

**Паттерн проектирования** — это типовое решение часто встречающейся проблемы при проектировании архитектуры программ. В отличие от готовых функций или библиотек, паттерн нельзя просто скопировать — это общая концепция решения, которую нужно адаптировать под свою задачу.

## Содержание

- [Классификация](#классификация)
- [SOLID — принципы, лежащие в основе](#solid)
- [Порождающие паттерны](#порождающие-паттерны-creational)
  - [Factory Method (Фабричный метод)](#factory-method-фабричный-метод)
  - [Abstract Factory (Абстрактная фабрика)](#abstract-factory-абстрактная-фабрика)
  - [Builder (Строитель)](#builder-строитель)
  - [Prototype (Прототип)](#prototype-прототип)
  - [Singleton (Одиночка)](#singleton-одиночка)
- [Структурные паттерны](#структурные-паттерны-structural)
  - [Adapter (Адаптер)](#adapter-адаптер)
  - [Bridge (Мост)](#bridge-мост)
  - [Composite (Компоновщик)](#composite-компоновщик)
  - [Decorator (Декоратор)](#decorator-декоратор)
  - [Facade (Фасад)](#facade-фасад)
  - [Flyweight (Легковес)](#flyweight-легковес)
  - [Proxy (Заместитель)](#proxy-заместитель)
- [Поведенческие паттерны](#поведенческие-паттерны-behavioral)
  - [Chain of Responsibility (Цепочка обязанностей)](#chain-of-responsibility-цепочка-обязанностей)
  - [Command (Команда)](#command-команда)
  - [Iterator (Итератор)](#iterator-итератор)
  - [Mediator (Посредник)](#mediator-посредник)
  - [Memento (Снимок)](#memento-снимок)
  - [Observer (Наблюдатель)](#observer-наблюдатель)
  - [State (Состояние)](#state-состояние)
  - [Strategy (Стратегия)](#strategy-стратегия)
  - [Template Method (Шаблонный метод)](#template-method-шаблонный-метод)
  - [Visitor (Посетитель)](#visitor-посетитель)

---

## Классификация

Паттерны делятся на три группы по назначению:

| Группа | Назначение | Паттерны |
|--------|-----------|----------|
| **Порождающие** (Creational) | Гибкое создание объектов, повышение переиспользуемости кода | Factory Method, Abstract Factory, Builder, Prototype, Singleton |
| **Структурные** (Structural) | Компоновка объектов в гибкие структуры | Adapter, Bridge, Composite, Decorator, Facade, Flyweight, Proxy |
| **Поведенческие** (Behavioral) | Эффективное взаимодействие и распределение обязанностей между объектами | Chain of Responsibility, Command, Iterator, Mediator, Memento, Observer, State, Strategy, Template Method, Visitor |

---

## SOLID

Паттерны опираются на принципы хорошего ООП-дизайна:

- **S** — Single Responsibility (единственная ответственность): у класса должна быть только одна причина для изменения.
- **O** — Open/Closed (открытость/закрытость): классы открыты для расширения, но закрыты для модификации.
- **L** — Liskov Substitution (подстановка Барбары Лисков): подтипы должны заменять базовые типы без поломки программы.
- **I** — Interface Segregation (разделение интерфейсов): много специализированных интерфейсов лучше одного общего.
- **D** — Dependency Inversion (инверсия зависимостей): зависимости от абстракций, а не от конкретных классов.

---

## Порождающие паттерны (Creational)

### Factory Method (Фабричный метод)

**Проблема:** нужно создавать объекты, но конкретный класс определяется в подклассах.

**Решение:** определяем интерфейс для создания объекта, а решение о том, какой класс инстанцировать, делегируем подклассам.

**Когда применять:**
- заранее неизвестны типы и зависимости объектов;
- хотим дать пользователям библиотеки способ расширять её компоненты.

```php
<?php

interface Transport
{
    public function deliver(): string;
}

class Truck implements Transport
{
    public function deliver(): string
    {
        return 'Доставка по дороге в коробке';
    }
}

class Ship implements Transport
{
    public function deliver(): string
    {
        return 'Доставка по морю в контейнере';
    }
}

abstract class Logistics
{
    abstract public function createTransport(): Transport;

    public function planDelivery(): string
    {
        $transport = $this->createTransport();
        return $transport->deliver();
    }
}

class RoadLogistics extends Logistics
{
    public function createTransport(): Transport
    {
        return new Truck();
    }
}

class SeaLogistics extends Logistics
{
    public function createTransport(): Transport
    {
        return new Ship();
    }
}

// Использование
echo (new RoadLogistics())->planDelivery(); // Доставка по дороге в коробке
echo (new SeaLogistics())->planDelivery();  // Доставка по морю в контейнере
```

---

### Abstract Factory (Абстрактная фабрика)

**Проблема:** нужно создавать семейства связанных объектов без привязки к конкретным классам.

**Решение:** интерфейс для создания каждого продукта семейства; конкретные фабрики создают согласованные наборы.

**Отличие от Factory Method:** фабричный метод создаёт один продукт, абстрактная фабрика — целое семейство связанных продуктов.

```php
<?php

interface Button { public function render(): string; }
interface Checkbox { public function render(): string; }

class WinButton implements Button {
    public function render(): string { return 'Кнопка Windows'; }
}
class MacButton implements Button {
    public function render(): string { return 'Кнопка macOS'; }
}
class WinCheckbox implements Checkbox {
    public function render(): string { return 'Чекбокс Windows'; }
}
class MacCheckbox implements Checkbox {
    public function render(): string { return 'Чекбокс macOS'; }
}

interface GUIFactory {
    public function createButton(): Button;
    public function createCheckbox(): Checkbox;
}

class WinFactory implements GUIFactory {
    public function createButton(): Button { return new WinButton(); }
    public function createCheckbox(): Checkbox { return new WinCheckbox(); }
}
class MacFactory implements GUIFactory {
    public function createButton(): Button { return new MacButton(); }
    public function createCheckbox(): Checkbox { return new MacCheckbox(); }
}

// Использование
function renderUI(GUIFactory $factory): void {
    echo $factory->createButton()->render() . PHP_EOL;
    echo $factory->createCheckbox()->render() . PHP_EOL;
}
renderUI(new MacFactory());
```

---

### Builder (Строитель)

**Проблема:** объект со множеством параметров и сложной пошаговой конструкцией («телескопический» конструктор).

**Решение:** выносим построение в отдельный объект-строитель; одни и те же шаги дают разные представления.

```php
<?php

class Pizza
{
    public array $toppings = [];
    public string $size = 'medium';
    public string $dough = 'thin';
}

class PizzaBuilder
{
    private Pizza $pizza;

    public function __construct() { $this->reset(); }
    public function reset(): void { $this->pizza = new Pizza(); }

    public function setSize(string $size): static { $this->pizza->size = $size; return $this; }
    public function setDough(string $dough): static { $this->pizza->dough = $dough; return $this; }
    public function addTopping(string $t): static { $this->pizza->toppings[] = $t; return $this; }

    public function build(): Pizza
    {
        $result = $this->pizza;
        $this->reset();
        return $result;
    }
}

// Использование (fluent-интерфейс)
$pizza = (new PizzaBuilder())
    ->setSize('large')
    ->setDough('thick')
    ->addTopping('сыр')
    ->addTopping('грибы')
    ->build();
```

---

### Prototype (Прототип)

**Проблема:** нужно копировать объекты, не завися от их классов.

**Решение:** объект сам умеет себя клонировать. В PHP — магический метод `__clone()`.

```php
<?php

class Document
{
    public array $content;
    public ComplexMeta $meta;

    public function __construct(array $content, ComplexMeta $meta)
    {
        $this->content = $content;
        $this->meta = $meta;
    }

    // Глубокое копирование вложенных объектов
    public function __clone()
    {
        $this->meta = clone $this->meta;
    }
}

class ComplexMeta { public string $author = ''; }

$original = new Document(['текст'], new ComplexMeta());
$copy = clone $original; // независимая копия, включая $meta
```

---

### Singleton (Одиночка)

**Проблема:** нужен ровно один экземпляр класса с глобальной точкой доступа.

**Решение:** приватный конструктор + статический метод доступа.

> ⚠️ Часто считается антипаттерном: усложняет тестирование, скрывает зависимости, нарушает SRP. Лучше использовать DI-контейнер.

```php
<?php

final class Database
{
    private static ?Database $instance = null;

    private function __construct() {}        // запрет new
    private function __clone() {}            // запрет clone
    public function __wakeup() { throw new \Exception('Нельзя десериализовать'); }

    public static function getInstance(): Database
    {
        if (self::$instance === null) {
            self::$instance = new self();
        }
        return self::$instance;
    }
}

$db = Database::getInstance();
```

---

## Структурные паттерны (Structural)

### Adapter (Адаптер)

**Проблема:** несовместимые интерфейсы должны работать вместе.

**Решение:** обёртка, преобразующая интерфейс одного класса в интерфейс, ожидаемый клиентом.

```php
<?php

interface JsonLogger { public function log(string $message): void; }

// Сторонний класс с несовместимым интерфейсом
class XmlExternalLogger {
    public function writeXml(string $xml): void { echo $xml . PHP_EOL; }
}

class XmlLoggerAdapter implements JsonLogger
{
    public function __construct(private XmlExternalLogger $external) {}

    public function log(string $message): void
    {
        $xml = "<log>{$message}</log>";
        $this->external->writeXml($xml);
    }
}

$logger = new XmlLoggerAdapter(new XmlExternalLogger());
$logger->log('Привет'); // <log>Привет</log>
```

---

### Bridge (Мост)

**Проблема:** класс растёт в двух независимых измерениях (например, форма × цвет), что ведёт к комбинаторному взрыву подклассов.

**Решение:** разделяем абстракцию и реализацию на две иерархии, связанные композицией.

```php
<?php

interface Renderer { public function renderShape(string $shape): string; }

class VectorRenderer implements Renderer {
    public function renderShape(string $shape): string { return "Вектор: $shape"; }
}
class RasterRenderer implements Renderer {
    public function renderShape(string $shape): string { return "Растр: $shape"; }
}

abstract class Shape {
    public function __construct(protected Renderer $renderer) {}
    abstract public function draw(): string;
}

class Circle extends Shape {
    public function draw(): string { return $this->renderer->renderShape('круг'); }
}

echo (new Circle(new VectorRenderer()))->draw(); // Вектор: круг
echo (new Circle(new RasterRenderer()))->draw(); // Растр: круг
```

---

### Composite (Компоновщик)

**Проблема:** нужно работать с деревом объектов единообразно — и с отдельными элементами, и с группами.

**Решение:** общий интерфейс для «листьев» и «контейнеров».

```php
<?php

interface FileSystemItem { public function getSize(): int; }

class File implements FileSystemItem {
    public function __construct(private int $size) {}
    public function getSize(): int { return $this->size; }
}

class Directory implements FileSystemItem {
    private array $items = [];
    public function add(FileSystemItem $item): void { $this->items[] = $item; }
    public function getSize(): int {
        return array_sum(array_map(fn($i) => $i->getSize(), $this->items));
    }
}

$root = new Directory();
$root->add(new File(100));
$sub = new Directory();
$sub->add(new File(50));
$root->add($sub);
echo $root->getSize(); // 150
```

---

### Decorator (Декоратор)

**Проблема:** нужно добавлять объекту новое поведение динамически, без наследования.

**Решение:** объект-обёртка с тем же интерфейсом, что и оборачиваемый объект.

```php
<?php

interface Coffee { public function cost(): float; public function description(): string; }

class SimpleCoffee implements Coffee {
    public function cost(): float { return 2.0; }
    public function description(): string { return 'Кофе'; }
}

abstract class CoffeeDecorator implements Coffee {
    public function __construct(protected Coffee $coffee) {}
}

class MilkDecorator extends CoffeeDecorator {
    public function cost(): float { return $this->coffee->cost() + 0.5; }
    public function description(): string { return $this->coffee->description() . ' + молоко'; }
}

class SugarDecorator extends CoffeeDecorator {
    public function cost(): float { return $this->coffee->cost() + 0.2; }
    public function description(): string { return $this->coffee->description() . ' + сахар'; }
}

$coffee = new SugarDecorator(new MilkDecorator(new SimpleCoffee()));
echo $coffee->description() . ': ' . $coffee->cost(); // Кофе + молоко + сахар: 2.7
```

---

### Facade (Фасад)

**Проблема:** сложная подсистема с множеством классов, которую тяжело использовать напрямую.

**Решение:** единый упрощённый интерфейс к подсистеме.

```php
<?php

class VideoFile { public function __construct(public string $name) {} }
class Codec {}
class BitrateReader { public static function read(string $f, Codec $c): string { return "data:$f"; } }
class AudioMixer { public function fix(string $d): string { return "$d:mixed"; } }

class VideoConverter
{
    public function convert(string $filename, string $format): string
    {
        $file = new VideoFile($filename);
        $data = BitrateReader::read($file->name, new Codec());
        $result = (new AudioMixer())->fix($data);
        return "$result.$format";
    }
}

// Клиенту не нужно знать о внутренней кухне
echo (new VideoConverter())->convert('video.ogg', 'mp4');
```

---

### Flyweight (Легковес)

**Проблема:** огромное количество однотипных объектов съедает память.

**Решение:** выносим повторяющееся (внутреннее) состояние в общие разделяемые объекты, а уникальное (внешнее) состояние передаём извне.

```php
<?php

// Внутреннее состояние (разделяемое)
class TreeType {
    public function __construct(
        public string $name,
        public string $color,
        public string $texture
    ) {}
}

class TreeFactory {
    private static array $types = [];
    public static function get(string $name, string $color, string $texture): TreeType {
        $key = "$name:$color:$texture";
        return self::$types[$key] ??= new TreeType($name, $color, $texture);
    }
}

// Внешнее состояние (уникальное) хранится отдельно
class Tree {
    public function __construct(public int $x, public int $y, public TreeType $type) {}
}

$forest = [];
for ($i = 0; $i < 1000; $i++) {
    // Все дубы делят один объект TreeType
    $forest[] = new Tree($i, $i, TreeFactory::get('Дуб', 'зелёный', 'кора'));
}
```

---

### Proxy (Заместитель)

**Проблема:** нужно контролировать доступ к объекту (ленивая загрузка, кэш, логирование, права).

**Решение:** объект-заместитель с тем же интерфейсом, что и реальный объект.

```php
<?php

interface Image { public function display(): string; }

class RealImage implements Image {
    public function __construct(private string $file) {
        echo "Загрузка $file с диска...\n"; // дорогая операция
    }
    public function display(): string { return "Показ {$this->file}"; }
}

class ProxyImage implements Image {
    private ?RealImage $real = null;
    public function __construct(private string $file) {}

    public function display(): string {
        $this->real ??= new RealImage($this->file); // ленивая загрузка
        return $this->real->display();
    }
}

$image = new ProxyImage('photo.jpg'); // диск ещё не читается
echo $image->display(); // загрузка происходит только сейчас
```

---

## Поведенческие паттерны (Behavioral)

### Chain of Responsibility (Цепочка обязанностей)

**Проблема:** запрос должен обработать один из нескольких обработчиков, и набор обработчиков меняется динамически.

**Решение:** связываем обработчики в цепочку; каждый решает — обработать запрос или передать дальше.

```php
<?php

abstract class Handler {
    protected ?Handler $next = null;
    public function setNext(Handler $h): Handler { $this->next = $h; return $h; }
    abstract public function handle(int $amount): ?string;
    protected function passNext(int $amount): ?string {
        return $this->next?->handle($amount);
    }
}

class CashHandler extends Handler {
    public function handle(int $amount): ?string {
        return $amount < 100 ? 'Оплата наличными' : $this->passNext($amount);
    }
}
class CardHandler extends Handler {
    public function handle(int $amount): ?string {
        return $amount < 10000 ? 'Оплата картой' : $this->passNext($amount);
    }
}

$cash = new CashHandler();
$cash->setNext(new CardHandler());
echo $cash->handle(500); // Оплата картой
```

---

### Command (Команда)

**Проблема:** нужно параметризовать объекты операциями, ставить их в очередь, логировать, отменять.

**Решение:** инкапсулируем запрос как объект.

```php
<?php

interface Command { public function execute(): void; public function undo(): void; }

class Light { public function on(): void { echo "Свет включён\n"; } public function off(): void { echo "Свет выключен\n"; } }

class LightOnCommand implements Command {
    public function __construct(private Light $light) {}
    public function execute(): void { $this->light->on(); }
    public function undo(): void { $this->light->off(); }
}

class RemoteControl {
    private array $history = [];
    public function submit(Command $c): void { $c->execute(); $this->history[] = $c; }
    public function undoLast(): void { array_pop($this->history)?->undo(); }
}

$remote = new RemoteControl();
$remote->submit(new LightOnCommand(new Light())); // Свет включён
$remote->undoLast();                              // Свет выключен
```

---

### Iterator (Итератор)

**Проблема:** нужно обходить элементы коллекции, не раскрывая её внутреннее устройство.

**Решение:** выносим логику обхода в отдельный объект. В PHP — встроенный интерфейс `Iterator`.

```php
<?php

class NumberCollection implements IteratorAggregate {
    private array $items = [];
    public function add(int $n): void { $this->items[] = $n; }
    public function getIterator(): Iterator {
        return new ArrayIterator($this->items);
    }
}

$collection = new NumberCollection();
$collection->add(1);
$collection->add(2);
foreach ($collection as $n) { echo $n; } // 12
```

---

### Mediator (Посредник)

**Проблема:** объекты слишком сильно связаны друг с другом (каждый знает о многих).

**Решение:** объекты общаются через посредника, не зная друг о друге напрямую.

```php
<?php

interface Mediator { public function notify(object $sender, string $event): void; }

class Dialog implements Mediator {
    public Button $button;
    public TextBox $textBox;

    public function notify(object $sender, string $event): void {
        if ($sender instanceof TextBox && $event === 'changed') {
            $this->button->enabled = $this->textBox->value !== '';
        }
    }
}

class Button { public bool $enabled = false; }
class TextBox {
    public string $value = '';
    public function __construct(private Mediator $mediator) {}
    public function setValue(string $v): void {
        $this->value = $v;
        $this->mediator->notify($this, 'changed');
    }
}

$dialog = new Dialog();
$dialog->button = new Button();
$dialog->textBox = new TextBox($dialog);
$dialog->textBox->setValue('текст');
var_dump($dialog->button->enabled); // true
```

---

### Memento (Снимок)

**Проблема:** нужно сохранять и восстанавливать состояние объекта, не нарушая инкапсуляцию.

**Решение:** объект создаёт снимок своего состояния; хранитель управляет историей.

```php
<?php

class EditorMemento {
    public function __construct(public readonly string $content) {}
}

class Editor {
    private string $content = '';
    public function type(string $words): void { $this->content .= $words; }
    public function getContent(): string { return $this->content; }
    public function save(): EditorMemento { return new EditorMemento($this->content); }
    public function restore(EditorMemento $m): void { $this->content = $m->content; }
}

$editor = new Editor();
$editor->type('Привет');
$saved = $editor->save();
$editor->type(' мир');
$editor->restore($saved);
echo $editor->getContent(); // Привет
```

---

### Observer (Наблюдатель)

**Проблема:** при изменении одного объекта нужно уведомить множество зависимых объектов.

**Решение:** механизм подписки — субъект уведомляет всех подписчиков об изменениях.

```php
<?php

interface Observer { public function update(string $event): void; }

class Subject {
    private array $observers = [];
    public function subscribe(Observer $o): void { $this->observers[] = $o; }
    public function notify(string $event): void {
        foreach ($this->observers as $o) { $o->update($event); }
    }
}

class EmailNotifier implements Observer {
    public function update(string $event): void { echo "Email: $event\n"; }
}
class SmsNotifier implements Observer {
    public function update(string $event): void { echo "SMS: $event\n"; }
}

$subject = new Subject();
$subject->subscribe(new EmailNotifier());
$subject->subscribe(new SmsNotifier());
$subject->notify('Новый заказ'); // Email: ... / SMS: ...
```

---

### State (Состояние)

**Проблема:** поведение объекта зависит от его состояния, и много условных операторов на этом завязано.

**Решение:** выносим каждое состояние в отдельный класс; объект делегирует поведение текущему состоянию.

```php
<?php

interface State { public function next(Order $order): void; public function getName(): string; }

class NewState implements State {
    public function next(Order $o): void { $o->setState(new PaidState()); }
    public function getName(): string { return 'Новый'; }
}
class PaidState implements State {
    public function next(Order $o): void { $o->setState(new ShippedState()); }
    public function getName(): string { return 'Оплачен'; }
}
class ShippedState implements State {
    public function next(Order $o): void { /* конечное состояние */ }
    public function getName(): string { return 'Отправлен'; }
}

class Order {
    private State $state;
    public function __construct() { $this->state = new NewState(); }
    public function setState(State $s): void { $this->state = $s; }
    public function next(): void { $this->state->next($this); }
    public function getStatus(): string { return $this->state->getName(); }
}

$order = new Order();
echo $order->getStatus(); // Новый
$order->next();
echo $order->getStatus(); // Оплачен
```

---

### Strategy (Стратегия)

**Проблема:** есть несколько вариантов одного алгоритма, и нужно переключать их во время выполнения.

**Решение:** выносим каждый алгоритм в отдельный класс с общим интерфейсом.

```php
<?php

interface PaymentStrategy { public function pay(int $amount): string; }

class CreditCardPayment implements PaymentStrategy {
    public function pay(int $amount): string { return "Оплата $amount картой"; }
}
class PayPalPayment implements PaymentStrategy {
    public function pay(int $amount): string { return "Оплата $amount через PayPal"; }
}

class Checkout {
    public function __construct(private PaymentStrategy $strategy) {}
    public function setStrategy(PaymentStrategy $s): void { $this->strategy = $s; }
    public function process(int $amount): string { return $this->strategy->pay($amount); }
}

$checkout = new Checkout(new CreditCardPayment());
echo $checkout->process(100);            // Оплата 100 картой
$checkout->setStrategy(new PayPalPayment());
echo $checkout->process(200);            // Оплата 200 через PayPal
```

> **Strategy vs State:** структурно похожи, но Strategy переключает *взаимозаменяемые* алгоритмы (клиент выбирает), а State моделирует *жизненный цикл* (состояния знают друг о друге и сами себя переключают).

---

### Template Method (Шаблонный метод)

**Проблема:** несколько алгоритмов отличаются лишь отдельными шагами.

**Решение:** базовый класс задаёт скелет алгоритма, подклассы переопределяют отдельные шаги.

```php
<?php

abstract class DataProcessor {
    // Шаблонный метод — задаёт порядок шагов
    final public function process(): void {
        $data = $this->read();
        $result = $this->transform($data);
        $this->save($result);
    }
    abstract protected function read(): string;
    abstract protected function transform(string $data): string;
    protected function save(string $data): void { echo "Сохранено: $data\n"; } // хук с реализацией по умолчанию
}

class CsvProcessor extends DataProcessor {
    protected function read(): string { return 'a,b,c'; }
    protected function transform(string $data): string { return strtoupper($data); }
}

(new CsvProcessor())->process(); // Сохранено: A,B,C
```

---

### Visitor (Посетитель)

**Проблема:** нужно добавлять операции к объектам иерархии, не меняя их классы.

**Решение:** выносим операции в объект-посетитель; элементы «принимают» посетителя.

```php
<?php

interface Visitor {
    public function visitCircle(Circle $c): string;
    public function visitSquare(Square $s): string;
}

interface ShapeElement { public function accept(Visitor $v): string; }

class Circle implements ShapeElement {
    public function __construct(public float $radius) {}
    public function accept(Visitor $v): string { return $v->visitCircle($this); }
}
class Square implements ShapeElement {
    public function __construct(public float $side) {}
    public function accept(Visitor $v): string { return $v->visitSquare($this); }
}

class AreaVisitor implements Visitor {
    public function visitCircle(Circle $c): string { return 'Площадь круга: ' . (3.14 * $c->radius ** 2); }
    public function visitSquare(Square $s): string { return 'Площадь квадрата: ' . ($s->side ** 2); }
}

$shapes = [new Circle(2), new Square(3)];
$visitor = new AreaVisitor();
foreach ($shapes as $shape) { echo $shape->accept($visitor) . PHP_EOL; }
```

---

## Шпаргалка: что выбрать

| Задача | Паттерн |
|--------|---------|
| Создать объект, тип которого решается позже | Factory Method |
| Создать семейство согласованных объектов | Abstract Factory |
| Собрать сложный объект по шагам | Builder |
| Скопировать объект без привязки к классу | Prototype |
| Гарантировать один экземпляр | Singleton |
| Подружить несовместимые интерфейсы | Adapter |
| Разделить абстракцию и реализацию | Bridge |
| Работать с деревом единообразно | Composite |
| Добавить поведение динамически | Decorator |
| Упростить сложную подсистему | Facade |
| Сэкономить память на множестве объектов | Flyweight |
| Контролировать доступ к объекту | Proxy |
| Передать запрос по цепочке обработчиков | Chain of Responsibility |
| Инкапсулировать операцию (undo/очередь) | Command |
| Обойти коллекцию | Iterator |
| Развязать множество связанных объектов | Mediator |
| Сохранить/откатить состояние | Memento |
| Уведомлять подписчиков об изменениях | Observer |
| Менять поведение в зависимости от состояния | State |
| Переключать взаимозаменяемые алгоритмы | Strategy |
| Задать скелет алгоритма с переопределяемыми шагами | Template Method |
| Добавить операцию без изменения классов | Visitor |

---

*Источник классификации и описаний: [refactoring.guru/ru/design-patterns](https://refactoring.guru/ru/design-patterns). Примеры на PHP 8.x.*
