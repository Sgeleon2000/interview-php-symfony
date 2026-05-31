# Fibers (Файберы) в PHP

> На основе [php.net/manual/en/language.fibers.php](https://www.php.net/manual/en/language.fibers.php). Доступны с **PHP 8.1+**.

**Fiber** (файбер, «волокно») — это механизм для создания полноценного стека выполнения, который можно **приостановить** и **возобновить** в произвольный момент. Файберы дают кооперативную многозадачность: код сам решает, когда уступить управление, в отличие от вытесняющей многопоточности.

## Содержание

- [Зачем нужны fibers](#зачем-нужны-fibers)
- [Класс Fiber: API](#класс-fiber-api)
- [Базовый пример](#базовый-пример)
- [Обмен значениями: suspend и resume](#обмен-значениями-suspend-и-resume)
- [Жизненный цикл и состояния](#жизненный-цикл-и-состояния)
- [Обработка исключений](#обработка-исключений)
- [Fibers vs Generators](#fibers-vs-generators)
- [Применение: асинхронность](#применение-асинхронность)
- [Ограничения](#ограничения)

---

## Зачем нужны fibers

Файберы — низкоуровневый примитив, на котором строятся **асинхронные фреймворки** (ReactPHP, Amp, Swoole). Они позволяют писать асинхронный код в **синхронном стиле**, без «callback hell» и без явных `Promise`-цепочек.

Главная идея: файбер можно «заморозить» в середине выполнения (`Fiber::suspend()`), вернуть управление вызывающему коду, а позже продолжить с того же места (`$fiber->resume()`), сохранив весь стек и локальные переменные.

```php
<?php
// Без fibers асинхронность выглядит как цепочки колбэков:
// fetchUrl($url)->then(fn($r) => process($r))->then(...);

// С fibers — как обычный синхронный код:
// $response = await(fetchUrl($url));   // под капотом — Fiber::suspend
// process($response);
```

> Обычный прикладной разработчик редко использует `Fiber` напрямую — обычно через библиотеку. Но понимать механизм полезно для собеседований и отладки async-кода.

---

## Класс Fiber: API

```php
<?php
final class Fiber
{
    public function __construct(callable $callback) {}

    // Запустить файбер; аргументы передаются в callback
    public function start(mixed ...$args): mixed;

    // Возобновить приостановленный файбер; $value станет результатом suspend()
    public function resume(mixed $value = null): mixed;

    // Бросить исключение внутрь приостановленного файбера
    public function throw(\Throwable $exception): mixed;

    // Состояние файбера
    public function isStarted(): bool;
    public function isSuspended(): bool;
    public function isRunning(): bool;
    public function isTerminated(): bool;

    // Значение, переданное в return файбера (после завершения)
    public function getReturn(): mixed;

    // СТАТИЧЕСКИЕ — вызываются ИЗНУТРИ файбера:
    public static function suspend(mixed $value = null): mixed;  // приостановить себя
    public static function getCurrent(): ?Fiber;                  // текущий файбер или null
}
```

---

## Базовый пример

```php
<?php
$fiber = new Fiber(function (): void {
    echo "1. Старт файбера\n";
    Fiber::suspend();                 // приостановка — управление возвращается наружу
    echo "3. Файбер возобновлён\n";
});

echo "0. До start\n";
$fiber->start();                      // выполнит код до suspend()
echo "2. Управление вернулось в main\n";
$fiber->resume();                     // продолжит с места suspend()
echo "4. Конец\n";

/* Вывод:
0. До start
1. Старт файбера
2. Управление вернулось в main
3. Файбер возобновлён
4. Конец
*/
```

---

## Обмен значениями: suspend и resume

Файбер и вызывающий код обмениваются значениями в обе стороны:

- `Fiber::suspend($value)` — `$value` становится возвращаемым значением `start()`/`resume()` снаружи.
- `$fiber->resume($value)` — `$value` становится возвращаемым значением `Fiber::suspend()` внутри.

```php
<?php
$fiber = new Fiber(function (): void {
    // suspend возвращает то, что передадут в resume()
    $received = Fiber::suspend('из файбера наружу');
    echo "Файбер получил: $received\n";
});

// start() возвращает значение из первого suspend()
$valueFromFiber = $fiber->start();
echo "Main получил: $valueFromFiber\n";        // из файбера наружу

$fiber->resume('из main внутрь');               // попадёт в $received
echo $fiber->getReturn() ?? 'нет возврата';

/* Вывод:
Main получил: из файбера наружу
Файбер получил: из main внутрь
*/
```

---

## Жизненный цикл и состояния

```php
<?php
$fiber = new Fiber(function () {
    Fiber::suspend();
});

var_dump($fiber->isStarted());     // false — ещё не запущен
$fiber->start();
var_dump($fiber->isSuspended());   // true  — приостановлен
var_dump($fiber->isRunning());     // false — сейчас выполняется main, не файбер
$fiber->resume();
var_dump($fiber->isTerminated());  // true  — завершён

echo $fiber->getReturn();          // значение из return файбера
```

Состояния:
- **not started** → после `new Fiber()`.
- **running** → пока выполняется код файбера.
- **suspended** → после `Fiber::suspend()`, ждёт `resume()`.
- **terminated** → после завершения callback (нормально или через исключение).

> Нельзя `resume()` уже завершённый файбер или `start()` повторно — будет `FiberError`.

---

## Обработка исключений

Исключение можно **бросить внутрь** приостановленного файбера через `$fiber->throw()` — оно возникнет в точке `Fiber::suspend()`. Неперехваченное исключение внутри файбера «всплывает» наружу при `start()`/`resume()`.

```php
<?php
$fiber = new Fiber(function (): void {
    try {
        Fiber::suspend();
    } catch (\RuntimeException $e) {
        echo "Файбер поймал: {$e->getMessage()}\n";
    }
});

$fiber->start();
$fiber->throw(new \RuntimeException('прерывание'));   // влетит в suspend()
// Вывод: Файбер поймал: прерывание
```

---

## Fibers vs Generators

И генераторы, и файберы умеют приостанавливать выполнение, но различаются глубиной:

| | Generator (`yield`) | Fiber |
|---|---------------------|-------|
| Приостановка | только в теле самого генератора | в **любом** месте стека вызовов |
| «Заразность» | вся цепочка должна быть генераторами | прозрачно — обычные функции |
| Возврат значения | `yield` / `return` | `suspend` / `return` + двунаправленный обмен |
| Назначение | ленивые последовательности, итерация | кооперативная многозадачность, async |
| Доступно с | PHP 5.5 | PHP 8.1 |

> Ключевое преимущество fibers: приостановиться можно **глубоко внутри** вложенных вызовов функций, не помечая каждую из них как генератор. С генераторами `yield` «протекает» по всей цепочке вызовов.

```php
<?php
// С генератором пришлось бы делать генераторами ВСЕ функции в цепочке.
// С файбером suspend() можно вызвать на любой глубине:
function deep() { Fiber::suspend(); }       // обычная функция
function middle() { deep(); }                // обычная функция
$f = new Fiber(fn() => middle());
$f->start();   // приостановится внутри deep()
```

---

## Применение: асинхронность

Файберы — фундамент event-loop библиотек. Упрощённая схема `await`:

```php
<?php
// Псевдокод того, как async-фреймворк использует Fiber:
function await(callable $asyncTask): mixed
{
    $fiber = Fiber::getCurrent()
        ?? throw new \Error('await вне файбера');

    // регистрируем задачу в event loop, при готовности — resume
    $asyncTask(fn($result) => $fiber->resume($result));

    return Fiber::suspend();   // отдаём управление event loop'у
}

// Тогда прикладной код выглядит синхронно:
// $user = await(fetchUser($id));
// $posts = await(fetchPosts($user));
```

Реальные реализации — **Amp** (`Amp\async`, `Amp\Future`), **ReactPHP**, **Revolt** event loop.

---

## Ограничения

- **Нельзя suspend из {main}** — `Fiber::suspend()` вне файбера бросает `FiberError`.
- **Один файбер = один поток** — fibers не дают параллелизма, только кооперативное переключение в рамках одного процесса/потока.
- **Нельзя приостановиться через границу C-стека** в некоторых случаях (внутри некоторых внутренних колбэков).
- Нужен **планировщик** (event loop) — сам по себе `Fiber` лишь примитив; кто и когда вызовет `resume()`, решает ваш код или библиотека.

```php
<?php
// ❌ Fiber::suspend();   // в {main} → FiberError: Cannot suspend outside of a fiber
```

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое Fiber? | Приостанавливаемый стек выполнения для кооперативной многозадачности (8.1+). |
| Дают ли fibers параллелизм? | Нет — только кооперативное переключение в одном потоке. |
| Как приостановить файбер? | `Fiber::suspend()` изнутри файбера. |
| Как возобновить? | `$fiber->resume($value)` снаружи. |
| Как обмениваются значениями? | `suspend($x)` → наружу в `resume`/`start`; `resume($y)` → внутрь как результат `suspend`. |
| Fiber vs Generator? | Fiber может приостановиться на любой глубине стека; generator — только в своём теле, и «заражает» цепочку. |
| Где используют fibers? | В async-фреймворках (Amp, ReactPHP, Revolt) как основа `await`. |
| Что будет при `suspend` в {main}? | `FiberError` — нельзя приостановиться вне файбера. |
| Как бросить исключение в файбер? | `$fiber->throw($exception)` — возникнет в точке `suspend()`. |
| Как получить результат файбера? | `$fiber->getReturn()` после завершения. |

---

*Источник: [php.net/manual/en/language.fibers.php](https://www.php.net/manual/en/language.fibers.php). Доступно с PHP 8.1+. См. также `language.generators.php`.*
