# PSR — стандарты PHP-FIG

> На основе [php-fig.org/psr](https://www.php-fig.org/psr/). PSR (PHP Standards Recommendations) — рекомендации, разрабатываемые группой **PHP-FIG** (Framework Interoperability Group) для совместимости PHP-фреймворков и библиотек.

## Что такое PHP-FIG и PSR

**PHP-FIG** — объединение разработчиков популярных проектов (Symfony, Laravel, Composer, Drupal и др.). Цель — договориться об общих интерфейсах и стилях, чтобы код разных библиотек был взаимозаменяемым.

Ключевая идея большинства PSR — **программирование от интерфейса**: ваш код зависит не от конкретной реализации (например, конкретного логгера), а от стандартного интерфейса (`Psr\Log\LoggerInterface`), и любую реализацию можно подменить.

## Содержание

- [Полный список и статусы](#полный-список-и-статусы)
- [Группы стандартов](#группы-стандартов)
- [Стиль кода](#стиль-кода)
  - [PSR-1 — Basic Coding Standard](#psr-1--basic-coding-standard)
  - [PSR-12 — Extended Coding Style](#psr-12--extended-coding-style)
- [Автозагрузка](#автозагрузка)
  - [PSR-4 — Autoloading Standard](#psr-4--autoloading-standard)
- [Интерфейсы](#интерфейсы)
  - [PSR-3 — Logger Interface](#psr-3--logger-interface)
  - [PSR-6 / PSR-16 — Caching](#psr-6--psr-16--caching)
  - [PSR-7 / PSR-15 / PSR-17 / PSR-18 — HTTP](#psr-7--psr-15--psr-17--psr-18--http)
  - [PSR-11 — Container Interface](#psr-11--container-interface)
  - [PSR-13 — Hypermedia Links](#psr-13--hypermedia-links)
  - [PSR-14 — Event Dispatcher](#psr-14--event-dispatcher)
  - [PSR-20 — Clock](#psr-20--clock)
- [Устаревшие и заброшенные](#устаревшие-и-заброшенные)

---

## Полный список и статусы

| № | Название | Статус | Группа |
|---|----------|--------|--------|
| 0 | Autoloading Standard | 🔴 Deprecated | Автозагрузка |
| 1 | Basic Coding Standard | ✅ Accepted | Стиль |
| 2 | Coding Style Guide | 🔴 Deprecated | Стиль |
| 3 | Logger Interface | ✅ Accepted | Интерфейс |
| 4 | Autoloading Standard | ✅ Accepted | Автозагрузка |
| 5 | PHPDoc Standard | 📝 Draft | Документация |
| 6 | Caching Interface | ✅ Accepted | Интерфейс |
| 7 | HTTP Message Interface | ✅ Accepted | Интерфейс |
| 8 | Huggable Interface | ⚫ Abandoned | (шутка) |
| 9 | Security Advisories | ⚫ Abandoned | — |
| 10 | Security Reporting Process | ⚫ Abandoned | — |
| 11 | Container Interface | ✅ Accepted | Интерфейс |
| 12 | Extended Coding Style Guide | ✅ Accepted | Стиль |
| 13 | Hypermedia Links | ✅ Accepted | Интерфейс |
| 14 | Event Dispatcher | ✅ Accepted | Интерфейс |
| 15 | HTTP Server Request Handlers | ✅ Accepted | Интерфейс |
| 16 | Simple Cache | ✅ Accepted | Интерфейс |
| 17 | HTTP Factories | ✅ Accepted | Интерфейс |
| 18 | HTTP Client | ✅ Accepted | Интерфейс |
| 19 | PHPDoc Tags | 📝 Draft | Документация |
| 20 | Clock | ✅ Accepted | Интерфейс |
| 21 | Internationalization | 📝 Draft | Интерфейс |
| 22 | Application Tracing | 📝 Draft | Интерфейс |

**Статусы:**
- ✅ **Accepted** — принят, готов к использованию.
- 📝 **Draft** — в разработке, может измениться.
- 🔴 **Deprecated** — устарел, заменён более новым.
- ⚫ **Abandoned** — заброшен (PSR-8 «Huggable Interface» — первоапрельская шутка).

---

## Группы стандартов

- **Стиль кода:** PSR-1, PSR-12 (PSR-2 устарел в пользу PSR-12).
- **Автозагрузка:** PSR-4 (PSR-0 устарел).
- **Документация:** PSR-5, PSR-19 (оба в статусе Draft).
- **Интерфейсы:** PSR-3, 6, 7, 11, 13, 14, 15, 16, 17, 18, 20.

---

## Стиль кода

### PSR-1 — Basic Coding Standard

Базовые правила, обязательные для совместимого кода:

- Файлы используют только теги `<?php` и `<?=`.
- Кодировка файлов — **UTF-8 без BOM**.
- Файл **либо** объявляет символы (классы, функции, константы), **либо** производит побочные эффекты (вывод, изменение настроек) — но не одновременно.
- Имена классов — `StudlyCaps` (`MyClassName`).
- Имена методов — `camelCase`.
- Константы классов — `UPPER_CASE`.

```php
<?php
// ✅ Правильно: файл только объявляет класс
namespace App\Service;

class UserService
{
    public const MAX_ATTEMPTS = 3;

    public function findUser(int $id): ?User
    {
        // ...
    }
}
```

### PSR-12 — Extended Coding Style

Расширенный гайд по форматированию (заменил устаревший **PSR-2**). Основные правила:

- Отступ — **4 пробела**, не табы.
- Открывающая фигурная скобка класса/метода — на **новой строке**; для управляющих конструкций (`if`, `for`) — на **той же строке**.
- Видимость (`public`/`protected`/`private`) указывается явно для всех свойств и методов.
- `declare(strict_types=1);` — на отдельной строке после `<?php`.
- Одна строка — не длиннее мягкого лимита (рекомендация ~120 символов).
- Списки `use` — после объявления `namespace`.

```php
<?php

declare(strict_types=1);

namespace App\Controller;

use App\Service\UserService;
use Psr\Log\LoggerInterface;

class UserController
{
    public function __construct(
        private readonly UserService $users,
        private readonly LoggerInterface $logger,
    ) {
    }

    public function show(int $id): void
    {
        if ($id <= 0) {
            throw new \InvalidArgumentException('Bad id');
        }
    }
}
```

> На практике форматирование автоматизируют инструментами: **PHP_CodeSniffer** (`phpcs`/`phpcbf`) или **PHP-CS-Fixer**.

---

## Автозагрузка

### PSR-4 — Autoloading Standard

Стандарт сопоставления **пространства имён** с **путём к файлу**. Заменил PSR-0 (тот тащил наследие PEAR с подчёркиваниями в именах).

Правило: полностью квалифицированное имя класса
```
\<Vendor>\<Namespace>\<...>\<ClassName>
```
отображается на файл `<ClassName>.php` внутри каталога, привязанного к префиксу.

**Пример `composer.json`:**
```json
{
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

Тогда:
- `App\Service\UserService` → `src/Service/UserService.php`
- `App\Controller\HomeController` → `src/Controller/HomeController.php`

После изменения — `composer dump-autoload`. Composer генерирует автозагрузчик, и `require 'vendor/autoload.php';` подключает всё.

---

## Интерфейсы

### PSR-3 — Logger Interface

Единый интерфейс логирования (`Psr\Log\LoggerInterface`). Восемь уровней из RFC 5424: `emergency`, `alert`, `critical`, `error`, `warning`, `notice`, `info`, `debug`.

```php
<?php

use Psr\Log\LoggerInterface;

class OrderService
{
    public function __construct(private LoggerInterface $logger) {}

    public function place(int $orderId): void
    {
        // Плейсхолдеры в фигурных скобках + контекст
        $this->logger->info('Order {id} placed', ['id' => $orderId]);

        try {
            // ...
        } catch (\Throwable $e) {
            $this->logger->error('Order failed', ['exception' => $e]);
        }
    }
}
```

Любая реализация (Monolog и др.) подменяется без изменения `OrderService`.

### PSR-6 / PSR-16 — Caching

Два стандарта кэширования:

- **PSR-6 (Caching Interface)** — мощный, через объекты-«элементы» (`CacheItemPoolInterface`, `CacheItemInterface`). Поддерживает отложенное сохранение, метаданные.
- **PSR-16 (Simple Cache)** — упрощённый, «ключ-значение» (`Psr\SimpleCache\CacheInterface`): `get`, `set`, `delete`, `has`.

```php
<?php

use Psr\SimpleCache\CacheInterface;

class WeatherService
{
    public function __construct(private CacheInterface $cache) {}

    public function getTemperature(string $city): float
    {
        $key = "weather.$city";
        if ($this->cache->has($key)) {
            return $this->cache->get($key);
        }
        $temp = $this->fetchFromApi($city);
        $this->cache->set($key, $temp, ttl: 3600);
        return $temp;
    }
}
```

### PSR-7 / PSR-15 / PSR-17 / PSR-18 — HTTP

Семейство стандартов для работы с HTTP:

- **PSR-7 (HTTP Message Interface)** — интерфейсы HTTP-сообщений: `RequestInterface`, `ResponseInterface`, `ServerRequestInterface`, `StreamInterface`, `UriInterface`. Сообщения **иммутабельны** — методы `with*()` возвращают новый объект.
- **PSR-17 (HTTP Factories)** — фабрики для создания PSR-7 объектов (`RequestFactoryInterface`, `ResponseFactoryInterface` и т.д.).
- **PSR-15 (HTTP Handlers)** — интерфейсы middleware (`MiddlewareInterface`) и обработчиков (`RequestHandlerInterface`).
- **PSR-18 (HTTP Client)** — единый интерфейс HTTP-клиента (`ClientInterface::sendRequest()`).

```php
<?php

use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;

// PSR-15 middleware
class AuthMiddleware implements MiddlewareInterface
{
    public function process(
        ServerRequestInterface $request,
        RequestHandlerInterface $handler
    ): ResponseInterface {
        if (!$request->hasHeader('Authorization')) {
            // вернуть 401 (через PSR-17 фабрику)
        }
        return $handler->handle($request);
    }
}
```

```php
<?php

use Psr\Http\Client\ClientInterface;
use Psr\Http\Message\RequestFactoryInterface;

// PSR-18 клиент + PSR-17 фабрика
$request = $requestFactory->createRequest('GET', 'https://api.example.com');
$response = $client->sendRequest($request); // ResponseInterface (PSR-7)
echo $response->getStatusCode();
```

### PSR-11 — Container Interface

Интерфейс **DI-контейнера** (`Psr\Container\ContainerInterface`) с двумя методами:

```php
<?php

use Psr\Container\ContainerInterface;

interface ContainerInterface
{
    public function get(string $id): mixed;   // вернуть сервис
    public function has(string $id): bool;     // есть ли сервис
}
```

Позволяет фреймворкам и библиотекам доставать зависимости из любого совместимого контейнера (Symfony DI, PHP-DI, Laravel Container).

### PSR-13 — Hypermedia Links

Интерфейсы для представления гиперссылок (`LinkInterface`) — для API в стиле HATEOAS / HAL.

### PSR-14 — Event Dispatcher

Стандарт событийной системы. Ключевые интерфейсы:

- `EventDispatcherInterface` — диспетчер, метод `dispatch(object $event)`.
- `ListenerProviderInterface` — поставщик слушателей для события.
- `StoppableEventInterface` — событие, распространение которого можно остановить.

```php
<?php

use Psr\EventDispatcher\EventDispatcherInterface;

class UserRegistered
{
    public function __construct(public readonly int $userId) {}
}

class RegistrationService
{
    public function __construct(private EventDispatcherInterface $dispatcher) {}

    public function register(string $email): void
    {
        // ... создать пользователя
        $this->dispatcher->dispatch(new UserRegistered(42));
    }
}
```

### PSR-20 — Clock

Интерфейс для получения текущего времени (`Psr\Clock\ClockInterface::now(): DateTimeImmutable`). Позволяет **подменять время в тестах** — вместо прямого вызова `new DateTime()`.

```php
<?php

use Psr\Clock\ClockInterface;

class SubscriptionService
{
    public function __construct(private ClockInterface $clock) {}

    public function isExpired(\DateTimeImmutable $expiresAt): bool
    {
        return $this->clock->now() > $expiresAt;
    }
}

// В тесте можно подставить FrozenClock с фиксированной датой
```

---

## Устаревшие и заброшенные

| PSR | Что с ним |
|-----|-----------|
| **PSR-0** | Устарел. Заменён PSR-4 (без подчёркиваний-разделителей в стиле PEAR). |
| **PSR-2** | Устарел. Заменён PSR-12. |
| **PSR-8** | Заброшен — «Huggable Interface» был первоапрельской шуткой. |
| **PSR-9 / PSR-10** | Заброшены — стандарты публикации информации о безопасности. |

**Драфты (в разработке):** PSR-5 (PHPDoc), PSR-19 (PHPDoc Tags), PSR-21 (Internationalization), PSR-22 (Application Tracing).

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое PSR? | Рекомендации PHP-FIG для совместимости фреймворков. |
| Чем PSR-4 отличается от PSR-0? | PSR-4 проще, без подчёркиваний-разделителей PEAR; PSR-0 устарел. |
| Какой PSR заменил PSR-2? | PSR-12 (Extended Coding Style). |
| Стандарт логирования? | PSR-3. |
| Стандарт DI-контейнера? | PSR-11 (`get`, `has`). |
| Стандарты HTTP? | PSR-7 (сообщения), 15 (middleware), 17 (фабрики), 18 (клиент). |
| Чем PSR-6 отличается от PSR-16? | PSR-6 — через pool/item, сложнее; PSR-16 — простой «ключ-значение». |
| Зачем PSR-20 (Clock)? | Чтобы подменять текущее время в тестах. |
| Главная идея PSR-интерфейсов? | Зависеть от абстракции, а не от конкретной реализации. |

---

*Источник: [php-fig.org/psr](https://www.php-fig.org/psr/). Примеры на PHP 8.x.*
