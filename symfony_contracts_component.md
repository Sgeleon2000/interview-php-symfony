# Symfony Contracts Component — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [The Contracts Component](https://symfony.com/doc/current/components/contracts.html), плюс примеры использования каждого пакета контрактов.

**Contracts** — набор абстракций (интерфейсы, трейты, normative docblocks), вытащенных из компонентов Symfony. Они позволяют писать слабосвязанный код, который умеет работать с любой реализацией контракта (Symfony, сторонние пакеты).

---

## 1. Установка — пакеты по доменам

```bash
# Каждый контракт — отдельный пакет. Ставишь только то, что нужно проекту:
composer require symfony/cache-contracts
composer require symfony/event-dispatcher-contracts
composer require symfony/deprecation-contracts
composer require symfony/http-client-contracts
composer require symfony/service-contracts
composer require symfony/translation-contracts
```

---

## 2. Принципы Contracts

```text
• Разделение по доменам — каждый контракт в своём sub-namespace.
• Маленькие согласованные наборы интерфейсов / трейтов / docblock'ов.
• Контракт принимают в репозиторий ТОЛЬКО при наличии proven implementation.
• Обратная совместимость с существующими компонентами Symfony гарантируется.
• Пакеты-реализации указывают это в composer.json через provide:
    "provide": { "symfony/cache-implementation": "3.0" }
```

---

## 3. `symfony/service-contracts` — `ServiceSubscriberInterface` и `ResetInterface`

### 3.1. `ServiceSubscriberInterface` — lazy-инъекция сервисов

```php
<?php
namespace App\Service;

use Psr\Container\ContainerInterface;
use Symfony\Contracts\Service\ServiceSubscriberInterface;
use Symfony\Contracts\Service\Attribute\SubscribedService;
use Psr\Log\LoggerInterface;
use Symfony\Component\Routing\RouterInterface;

/**
 * Обычный DI инициализирует ВСЕ зависимости при создании сервиса.
 * Service Subscriber получает контейнер-локатор и достаёт сервисы ТОЛЬКО при
 * реальной необходимости. Полезно, когда из 10 зависимостей нужны 1-2 за запрос.
 */
class HeavyService implements ServiceSubscriberInterface
{
    public function __construct(
        private ContainerInterface $locator, // PSR-11 контейнер-локатор (инжектится автоматически)
    ) {}

    // Объявляем, какие сервисы можем достать из локатора.
    // Тип сервиса = имя метода locator->get(...).
    public static function getSubscribedServices(): array
    {
        return [
            // Базовый синтаксис: 'ключ' => 'тип'
            'logger' => LoggerInterface::class,

            // Через PHP-атрибут (Symfony 6.2+) — компактнее:
            // ключ = тип, имя = имя сервиса (если нужно)
            LoggerInterface::class,
            RouterInterface::class,
        ];
    }

    public function doSomething(): void
    {
        // Сервис будет создан ТОЛЬКО при первом обращении (lazy)
        $logger = $this->locator->get(LoggerInterface::class);
        $logger->info('Lazy-loaded!');
    }
}
```

С атрибутом `#[SubscribedService]` (более новый способ):

```php
<?php
namespace App\Service;

use Psr\Container\ContainerInterface;
use Psr\Log\LoggerInterface;
use Symfony\Contracts\Service\Attribute\SubscribedService;
use Symfony\Contracts\Service\ServiceSubscriberInterface;
use Symfony\Contracts\Service\ServiceSubscriberTrait;

class ModernSubscriber implements ServiceSubscriberInterface
{
    // Трейт сам реализует getSubscribedServices() — читает атрибуты ниже
    use ServiceSubscriberTrait;

    public function __construct(private ContainerInterface $locator) {}

    // Сервис подгрузится при вызове $this->logger()
    #[SubscribedService]
    private function logger(): LoggerInterface
    {
        return $this->locator->get(__FUNCTION__);
    }
}
```

### 3.2. `ResetInterface` — сброс состояния между запросами

```php
<?php
namespace App\Service;

use Symfony\Contracts\Service\ResetInterface;

/**
 * В обычном PHP сервис умирает после запроса. Но в FrankenPHP worker mode,
 * Roadrunner и т.п. процесс живёт между запросами → накопится состояние.
 * Symfony автоматически вызовет reset() после каждого запроса у всех сервисов,
 * реализующих ResetInterface.
 */
class RequestCounter implements ResetInterface
{
    private int $count = 0;
    private array $log = [];

    public function inc(string $url): void
    {
        $this->count++;
        $this->log[] = $url;
    }

    public function reset(): void
    {
        $this->count = 0;
        $this->log   = [];
    }
}
```

---

## 4. `symfony/cache-contracts` — `CacheInterface`

```php
<?php
namespace App\Service;

use Psr\Cache\CacheItemInterface;
use Symfony\Contracts\Cache\CacheInterface;
use Symfony\Contracts\Cache\ItemInterface;

/**
 * CacheInterface (Symfony) ≠ CacheItemPoolInterface (PSR-6).
 * Symfony-вариант избавляет от boilerplate "get → check isHit → save".
 * Метод get() сам разруливает cache miss через колбэк.
 */
class WeatherService
{
    // CacheInterface inject'ится автоматически (autowire подставит реализацию).
    public function __construct(private CacheInterface $cache) {}

    public function getForecast(string $city): array
    {
        // 1-й аргумент: ключ кэша
        // 2-й: колбэк, который выполнится ТОЛЬКО при cache miss
        return $this->cache->get('weather.' . $city, function (ItemInterface $item) use ($city) {
            $item->expiresAfter(3600); // TTL 1 час
            $item->tag(['weather', 'city-' . $city]); // теги для инвалидации (если реализация поддерживает)

            // Тяжёлый запрос — кэшируется
            return ['city' => $city, 'temp' => random_int(-20, 35)];
        });
        // При cache hit колбэк не вызывается — сразу возвращается значение.
    }

    public function invalidate(string $city): void
    {
        // delete() — стандартный метод из PSR-6 CacheItemPoolInterface,
        // CacheInterface наследует его поведение.
        $this->cache->delete('weather.' . $city);
    }
}
```

---

## 5. `symfony/event-dispatcher-contracts` — `EventDispatcherInterface` и базовый `Event`

```php
<?php
namespace App\Service;

use Symfony\Contracts\EventDispatcher\Event;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

/**
 * Symfony\Contracts\EventDispatcher\EventDispatcherInterface — ТОЛЬКО dispatch().
 * Используй его в сервисах, если нужно только диспатчить события.
 *
 * Symfony\Component\EventDispatcher\EventDispatcherInterface — полный API
 * (addListener, removeListener, getListeners, ...). Используй ТОЛЬКО если
 * нужно манипулировать listener'ами в рантайме.
 */
class OrderEvent extends Event // базовый Event из contracts: stopPropagation()/isPropagationStopped()
{
    public function __construct(public readonly int $orderId) {}
}

class OrderService
{
    public function __construct(private EventDispatcherInterface $dispatcher) {}

    public function place(int $orderId): void
    {
        // Имя события опционально — по умолчанию = FQCN класса события
        $this->dispatcher->dispatch(new OrderEvent($orderId));
        // $this->dispatcher->dispatch($event, 'order.placed'); // с явным именем
    }
}
```

---

## 6. `symfony/http-client-contracts` — `HttpClientInterface` и `ResponseInterface`

```php
<?php
namespace App\Service;

use Symfony\Contracts\HttpClient\HttpClientInterface;
use Symfony\Contracts\HttpClient\ResponseInterface;

/**
 * Type-hint на интерфейс из contracts — твой код работает с любой реализацией:
 * symfony/http-client (curl, native), мок-клиентом в тестах, Guzzle-адаптером и т.д.
 */
class GitHubClient
{
    public function __construct(private HttpClientInterface $http) {}

    public function fetchUser(string $login): array
    {
        // Запрос неблокирующий — реальная отправка при чтении ответа
        $response = $this->http->request('GET', "https://api.github.com/users/$login", [
            'headers' => ['Accept' => 'application/vnd.github+json'],
            'timeout' => 5.0,
        ]);

        // Методы из ResponseInterface
        $statusCode = $response->getStatusCode();
        $headers    = $response->getHeaders();
        $body       = $response->getContent();      // строка
        $array      = $response->toArray();         // JSON → array

        return $array;
    }

    // ChunkInterface для streaming через stream()
    public function downloadStream(string $url): \Generator
    {
        $response = $this->http->request('GET', $url);
        foreach ($this->http->stream($response) as $chunk) {
            yield $chunk->getContent();
        }
    }
}
```

---

## 7. `symfony/translation-contracts` — `TranslatorInterface` и `LocaleAwareInterface`

```php
<?php
namespace App\Service;

use Symfony\Contracts\Translation\LocaleAwareInterface;
use Symfony\Contracts\Translation\TranslatorInterface;

class Greeter
{
    // Принимаем интерфейс из contracts — реализация может быть какой угодно
    // (symfony/translation, gettext-адаптер, мок в тестах).
    public function __construct(private TranslatorInterface $translator) {}

    public function greet(string $name, string $locale = null): string
    {
        return $this->translator->trans(
            id: 'hello.user',                  // ключ перевода
            parameters: ['%name%' => $name],   // подстановки
            domain: 'messages',                // домен (опц.)
            locale: $locale,                   // если null — текущая локаль
        );
    }

    public function switchLocale(string $locale): void
    {
        // LocaleAwareInterface — отдельный контракт для смены локали в рантайме
        if ($this->translator instanceof LocaleAwareInterface) {
            $this->translator->setLocale($locale);
        }
    }
}
```

---

## 8. `symfony/deprecation-contracts` — единый способ помечать deprecations

```php
<?php
namespace App\Service;

class LegacyApi
{
    public function oldMethod(): void
    {
        // trigger_deprecation($package, $version, $message, ...$args)
        // Единая точка во всей экосистеме Symfony и сторонних пакетов.
        // Без зависимостей в рантайме — функция определяется через autoload.
        trigger_deprecation(
            'acme/my-package',         // пакет, где deprecation объявлен
            '2.5',                     // версия, начиная с которой
            'Method "%s::oldMethod()" is deprecated, use "newMethod()" instead.',
            self::class,
        );

        // ... legacy-логика
    }

    public function newMethod(): void
    {
        // современный код
    }
}
```

---

## 9. Сводная таблица пакетов Contracts

| Пакет                                | Главные абстракции                                                | Назначение                                                                 |
| ------------------------------------ | ----------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `symfony/cache-contracts`            | `CacheInterface`, `ItemInterface`, `CacheTrait`                   | Кэширование с колбэком, без boilerplate'а PSR-6                            |
| `symfony/event-dispatcher-contracts` | `EventDispatcherInterface`, `Event`                               | Диспатч событий (минимальный API, только `dispatch()`)                     |
| `symfony/deprecation-contracts`      | функция `trigger_deprecation()`                                   | Единый способ выкидывать deprecation-уведомления                           |
| `symfony/http-client-contracts`      | `HttpClientInterface`, `ResponseInterface`, `ChunkInterface`      | HTTP-клиент с lazy/streaming-семантикой                                    |
| `symfony/service-contracts`          | `ServiceSubscriberInterface`, `ServiceProviderInterface`, `ResetInterface`, `#[SubscribedService]` | Lazy service locator, сброс состояния в long-running runtimes |
| `symfony/translation-contracts`      | `TranslatorInterface`, `LocaleAwareInterface`, `TranslatableInterface` | Перевод и локализация                                                  |

---

## 10. Когда использовать Contracts vs Component vs PSR

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ PSR (php-fig)                                                            │
│   Универсальный стандарт всего PHP-сообщества.                           │
│   Цель: интероперабельность ВСЕХ фреймворков.                            │
│   Пример: PSR-3 LoggerInterface, PSR-6 CacheItemPoolInterface,           │
│           PSR-11 ContainerInterface, PSR-18 ClientInterface              │
├──────────────────────────────────────────────────────────────────────────┤
│ Symfony Contracts                                                        │
│   Абстракции, "обкатанные" в Symfony, но без привязки к фреймворку.     │
│   Часто строятся ПОВЕРХ PSR (там, где это уместно).                     │
│   Цель: дать богатый удобный API + интероперабельность.                 │
│   Пример: CacheInterface (расширяет PSR-6), TranslatorInterface         │
├──────────────────────────────────────────────────────────────────────────┤
│ Symfony Components                                                       │
│   Конкретные реализации контрактов с полной функциональностью.          │
│   Пример: symfony/cache, symfony/translation, symfony/event-dispatcher  │
└──────────────────────────────────────────────────────────────────────────┘

ПРАВИЛО ТИПИЗАЦИИ В СВОИХ СЕРВИСАХ:
  • В аргументах конструктора → type-hint на Contracts/PSR интерфейс.
  • В реализациях (тестовых моках, своих классах) → можно зависеть от
    конкретного компонента, если нужно его API.
```

---

## 11. Реализация своего контракта в пакете

`composer.json` своего пакета, который реализует один из контрактов:

```json
{
    "name": "acme/super-cache",
    "require": {
        "symfony/cache-contracts": "^3.0"
    },
    "provide": {
        "symfony/cache-implementation": "3.0"
    }
}
```

Конвенция `symfony/*-implementation` нужна, чтобы пакеты, требующие "какую-нибудь реализацию контракта", могли указать это в `require`:

```json
{
    "require": {
        "symfony/cache-implementation": "^3.0"
    }
}
```

Composer тогда найдёт любой пакет, который заявил `provide` для этого виртуального имени.

---

## 12. Главное правило

```php
// ✗ ПЛОХО — жёсткая привязка к конкретной реализации
public function __construct(
    private \Symfony\Component\Cache\Adapter\RedisAdapter $cache,
) {}

// ✓ ХОРОШО — type-hint на контракт, любая реализация подойдёт
public function __construct(
    private \Symfony\Contracts\Cache\CacheInterface $cache,
) {}

// ✓ ИДЕАЛЬНО (когда подходит) — type-hint на PSR
public function __construct(
    private \Psr\Cache\CacheItemPoolInterface $cache,
) {}
```