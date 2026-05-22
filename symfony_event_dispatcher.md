# Symfony Events and Event Listeners — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [Events and Event Listeners](https://symfony.com/doc/current/event_dispatcher.html), с пояснениями в комментариях.

В Symfony во время выполнения приложения постоянно диспатчатся события. Слушать их можно через **Event Listener** или **Event Subscriber**. Также можно создавать свои события и диспатчить их.

---

## 1. Event Listener (3 способа регистрации)

### 1.1. Listener с тегом `kernel.event_listener` через config

`src/EventListener/ExceptionListener.php`

```php
<?php
namespace App\EventListener;

use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\Exception\HttpExceptionInterface;

class ExceptionListener
{
    // __invoke вызовется по умолчанию, если в теге не задан 'method'.
    public function __invoke(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();
        $message = sprintf(
            'My Error says: %s with code: %s',
            $exception->getMessage(),
            $exception->getCode()
        );

        $response = new Response($message);
        // ВАЖНО: text/plain — сообщение может содержать пользовательский ввод (XSS!)
        $response->headers->set('Content-Type', 'text/plain; charset=utf-8');

        // HttpExceptionInterface несёт status code и headers
        if ($exception instanceof HttpExceptionInterface) {
            $response->setStatusCode($exception->getStatusCode());
            $response->headers->replace($exception->getHeaders());
        } else {
            $response->setStatusCode(Response::HTTP_INTERNAL_SERVER_ERROR);
        }

        // Передаём готовый Response в систему — propagation останавливается
        $event->setResponse($response);
    }
}
```

`config/services.yaml`

```yaml
services:
    App\EventListener\ExceptionListener:
        tags:
            # Минимальная регистрация: вызовется __invoke()
            - { name: kernel.event_listener }

            # Полная форма с опциями
            - name: kernel.event_listener
              event: kernel.exception          # какое событие слушать
              method: onKernelException        # какой метод вызвать (опц.)
              priority: 10                     # порядок: больше = раньше (-256..256)
              # Если в метод не вкручен type-hint события, 'event' нужен,
              # чтобы Symfony знал тип $event-аргумента.
```

Логика выбора метода Symfony:
1. Если задан `method` в теге — вызывается он.
2. Иначе, если задан `event` — вызывается `on` + PascalCase имя события (например, `onKernelException`).
3. Иначе вызывается `__invoke()`.
4. Иначе — исключение.

---

### 1.2. Listener через PHP-атрибут `#[AsEventListener]` (на классе)

`src/EventListener/MyListener.php`

```php
<?php
namespace App\EventListener;

use App\Event\CustomEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener] // самый простой вариант — событие определяется по type-hint в __invoke
final class MyListener
{
    public function __invoke(CustomEvent $event): void
    {
        // ...
    }
}
```

Несколько атрибутов на одном классе:

```php
<?php
namespace App\EventListener;

use App\Event\CustomEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

// Если method не указан — вызовется on + PascalCase имени события.
// Для 'foo' → onFoo()
#[AsEventListener(event: CustomEvent::class, method: 'onCustomEvent')]
#[AsEventListener(event: 'foo', priority: 42)]
#[AsEventListener(event: 'bar', method: 'onBarEvent')]
final class MyMultiListener
{
    public function onCustomEvent(CustomEvent $event): void { /* ... */ }
    public function onFoo(): void { /* ... */ }
    public function onBarEvent(): void { /* ... */ }
}
```

---

### 1.3. Listener через атрибут на МЕТОДЕ

```php
<?php
namespace App\EventListener;

use App\Event\CustomEvent;
use App\Event\AnotherCustomEvent;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

final class MyMultiListener
{
    // event не нужен — Symfony возьмёт его из type-hint
    #[AsEventListener]
    public function onCustomEvent(CustomEvent $event): void {}

    // Union-type → подписка сразу на оба события
    #[AsEventListener]
    public function onMultipleCustomEvent(CustomEvent|AnotherCustomEvent $event): void {}

    // Если тип-хинта нет — нужно указать event
    #[AsEventListener(event: 'foo', priority: 42)]
    public function onFoo(): void {}

    #[AsEventListener(event: 'bar')]
    public function onBarEvent(): void {}
}
```

---

## 2. Event Subscriber

В отличие от listener, **subscriber сам знает, какие события слушает**. Реализует `EventSubscriberInterface::getSubscribedEvents()`.

`src/EventSubscriber/ExceptionSubscriber.php`

```php
<?php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;

class ExceptionSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        // Один subscriber может слушать ОДНО событие несколькими методами.
        // Формат: [имя_метода, приоритет]. Чем больше priority — тем раньше.
        // Приоритеты СУММИРУЮТСЯ со ВСЕМИ listener'ами и subscriber'ами.
        return [
            ExceptionEvent::class => [
                ['processException', 10],   // вызовется первым
                ['logException', 0],        // вторым
                ['notifyException', -10],   // последним
            ],
            // Можно также использовать строковое имя события:
            // 'kernel.exception' => 'processException',
            // 'kernel.exception' => ['processException', 10],
        ];
    }

    public function processException(ExceptionEvent $event): void { /* ... */ }
    public function logException(ExceptionEvent $event): void { /* ... */ }
    public function notifyException(ExceptionEvent $event): void { /* ... */ }
}
```

Благодаря `autoconfigure: true` Symfony сам добавит тег `kernel.event_subscriber`. Иначе тег нужно прописать вручную.

---

## 3. Main vs Sub-request — типичная проверка в listener'ах

```php
<?php
namespace App\EventListener;

use Symfony\Component\HttpKernel\Event\RequestEvent;

class RequestListener
{
    public function onKernelRequest(RequestEvent $event): void
    {
        // Одна страница может породить main-request + несколько sub-request'ов
        // (например, при embed-контроллерах в шаблонах).
        // Часто логика должна срабатывать только на main-request.
        if (!$event->isMainRequest()) {
            return;
        }
        // ...
    }
}
```

---

## 4. Listeners vs Subscribers — что выбрать?

```text
Listener:
  + Гибче — bundle может включать/выключать его через конфиг.
  − Знание о событиях лежит в конфигурации сервиса (тег или атрибут).

Subscriber:
  + Легче переиспользовать — все события описаны прямо в классе.
  + Symfony внутри использует subscriber'ы.
  − Менее гибкий для динамического вкл./выкл. из конфига bundle'а.
```

---

## 5. Event Aliases — FQCN вместо строкового имени

```php
<?php
namespace App\EventSubscriber;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\RequestEvent;

class RequestSubscriber implements EventSubscriberInterface
{
    public static function getSubscribedEvents(): array
    {
        return [
            // Используем FQCN класса события — это alias для 'kernel.request'.
            // Маппинг происходит при компиляции контейнера.
            RequestEvent::class => 'onKernelRequest',
        ];
    }

    public function onKernelRequest(RequestEvent $event): void { /* ... */ }
}
```

Расширение списка алиасов для своих событий через compiler pass:

```php
<?php
// src/Kernel.php
namespace App;

use App\Event\MyCustomEvent;
use Symfony\Component\DependencyInjection\ContainerBuilder;
use Symfony\Component\EventDispatcher\DependencyInjection\AddEventAliasesPass;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    protected function build(ContainerBuilder $container): void
    {
        // Регистрируем алиас: FQCN класса → строковое имя события.
        // Можно регистрировать pass несколько раз — список будет дополняться.
        $container->addCompilerPass(new AddEventAliasesPass([
            MyCustomEvent::class => 'my_custom_event',
        ]));
    }
}
```

---

## 6. Debugging — консольные команды

```bash
# Все события и их listener'ы
php bin/console debug:event-dispatcher

# Listener'ы конкретного события
php bin/console debug:event-dispatcher kernel.exception

# Частичное совпадение по имени
php bin/console debug:event-dispatcher kernel    # → kernel.exception, kernel.response, ...
php bin/console debug:event-dispatcher Security  # → Symfony\...\CheckPassportEvent

# У security своя шина событий per-firewall
php bin/console debug:event-dispatcher --dispatcher=security.event_dispatcher.main
```

---

## 7. Before/After-фильтры — практический пример с токеном

### 7.1. Параметры с токенами

`config/services.yaml`

```yaml
parameters:
    tokens:
        client1: pass1
        client2: pass2

services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'

    # Передаём параметр tokens в subscriber
    App\EventSubscriber\TokenSubscriber:
        arguments:
            $tokens: '%tokens%'
```

### 7.2. Маркерный интерфейс для контроллеров

`src/Controller/TokenAuthenticatedController.php`

```php
<?php
namespace App\Controller;

// Пустой "маркерный" интерфейс — контроллеры реализуют его,
// чтобы subscriber знал: "вот этот контроллер требует токен".
interface TokenAuthenticatedController
{
}
```

`src/Controller/FooController.php`

```php
<?php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class FooController extends AbstractController implements TokenAuthenticatedController
{
    #[Route('/foo/bar')]
    public function bar(): Response
    {
        return new Response('Secret data!');
    }
}
```

### 7.3. Subscriber: before + after в одном месте

`src/EventSubscriber/TokenSubscriber.php`

```php
<?php
namespace App\EventSubscriber;

use App\Controller\TokenAuthenticatedController;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpKernel\Event\ControllerEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\Exception\AccessDeniedHttpException;
use Symfony\Component\HttpKernel\KernelEvents;

class TokenSubscriber implements EventSubscriberInterface
{
    public function __construct(
        private array $tokens, // приходит из %tokens%
    ) {}

    // ===== BEFORE-фильтр: kernel.controller =====
    // Срабатывает ПЕРЕД вызовом контроллера на КАЖДОМ запросе.
    public function onKernelController(ControllerEvent $event): void
    {
        $controller = $event->getController();

        // Если контроллер — [$instance, 'methodName'], берём только сам объект.
        if (is_array($controller)) {
            $controller = $controller[0];
        }

        // Реагируем только на контроллеры с маркерным интерфейсом
        if ($controller instanceof TokenAuthenticatedController) {
            $token = $event->getRequest()->query->get('token');

            if (!in_array($token, $this->tokens, true)) {
                // Исключение → kernel.exception → 403
                throw new AccessDeniedHttpException('This action needs a valid token!');
            }

            // Помечаем запрос как прошедший токен-аутентификацию.
            // attributes — место для произвольных request-scoped данных.
            $event->getRequest()->attributes->set('auth_token', $token);
        }
    }

    // ===== AFTER-фильтр: kernel.response =====
    // Срабатывает ПОСЛЕ того, как контроллер вернул Response.
    public function onKernelResponse(ResponseEvent $event): void
    {
        // Был ли запрос помечен как токен-аутентифицированный?
        if (!$token = $event->getRequest()->attributes->get('auth_token')) {
            return;
        }

        $response = $event->getResponse();

        // Добавляем хэш контента с "солью" из токена в header
        $hash = sha1($response->getContent() . $token);
        $response->headers->set('X-CONTENT-HASH', $hash);
    }

    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::CONTROLLER => 'onKernelController', // BEFORE
            KernelEvents::RESPONSE   => 'onKernelResponse',   // AFTER
        ];
    }
}
```

---

## 8. Кастомные события — пользовательский dispatching

### 8.1. Класс события

`src/Event/BeforeSendMailEvent.php`

```php
<?php
namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

// Наследование от Event даёт методы stopPropagation()/isPropagationStopped()
class BeforeSendMailEvent extends Event
{
    public function __construct(
        private string $subject,
        private string $message,
    ) {}

    public function getSubject(): string         { return $this->subject; }
    public function setSubject(string $s): void  { $this->subject = $s; }
    public function getMessage(): string         { return $this->message; }
    public function setMessage(string $m): void  { $this->message = $m; }
}
```

`src/Event/AfterSendMailEvent.php`

```php
<?php
namespace App\Event;

use Symfony\Contracts\EventDispatcher\Event;

class AfterSendMailEvent extends Event
{
    public function __construct(private mixed $returnValue) {}

    public function getReturnValue(): mixed         { return $this->returnValue; }
    public function setReturnValue(mixed $v): void  { $this->returnValue = $v; }
}
```

### 8.2. Сервис, который диспатчит события до и после метода

`src/Mailer/CustomMailer.php`

```php
<?php
namespace App\Mailer;

use App\Event\AfterSendMailEvent;
use App\Event\BeforeSendMailEvent;
use Symfony\Contracts\EventDispatcher\EventDispatcherInterface;

class CustomMailer
{
    public function __construct(
        // ВАЖНО: type-hint интерфейс, не конкретный EventDispatcher.
        // Symfony\Contracts\EventDispatcher\EventDispatcherInterface — только dispatch()
        // Symfony\Component\EventDispatcher\EventDispatcherInterface — ещё addListener()/removeListener() и т.д.
        private EventDispatcherInterface $dispatcher,
    ) {}

    public function send(string $subject, string $message): mixed
    {
        // ----- BEFORE -----
        $event = new BeforeSendMailEvent($subject, $message);
        // 2-й аргумент — строковое имя события. Если не передать, используется FQCN класса.
        $this->dispatcher->dispatch($event, 'mailer.pre_send');

        // Listener'ы могли изменить данные
        $subject = $event->getSubject();
        $message = $event->getMessage();

        // ----- РЕАЛЬНАЯ РАБОТА -----
        $returnValue = sprintf('Отправлено: [%s] %s', $subject, $message);

        // ----- AFTER -----
        $event = new AfterSendMailEvent($returnValue);
        $this->dispatcher->dispatch($event, 'mailer.post_send');

        // Listener мог изменить return value
        return $event->getReturnValue();
    }
}
```

### 8.3. Subscriber на кастомное событие

`src/EventSubscriber/MailPostSendSubscriber.php`

```php
<?php
namespace App\EventSubscriber;

use App\Event\AfterSendMailEvent;
use Symfony\Component\EventDispatcher\EventSubscriberInterface;

class MailPostSendSubscriber implements EventSubscriberInterface
{
    public function onMailerPostSend(AfterSendMailEvent $event): void
    {
        $returnValue = $event->getReturnValue();
        // Модифицируем результат
        $event->setReturnValue($returnValue . ' [post-processed]');
    }

    public static function getSubscribedEvents(): array
    {
        return [
            'mailer.post_send' => 'onMailerPostSend',
        ];
    }
}
```

---

## 9. Остановка propagation

```php
<?php
use Symfony\Contracts\EventDispatcher\Event;

class MyEvent extends Event {}

// В listener'е:
public function __invoke(MyEvent $event): void
{
    // После этого вызова listener'ы с меньшим приоритетом НЕ выполнятся
    $event->stopPropagation();

    // Проверить можно так:
    // $event->isPropagationStopped();
}
```

Также propagation останавливается автоматически, когда listener задаёт ответ через `setResponse()` для событий `kernel.request`, `kernel.view`, `kernel.exception`.

---

## 10. Логика выбора метода в listener (тег `kernel.event_listener`)

```text
БЕЗ атрибута 'event' в теге:
  1. Если задан 'method' → вызывается он.
  2. Иначе → __invoke().
  3. Иначе → исключение.

С атрибутом 'event' в теге:
  1. Если задан 'method' → вызывается он.
  2. Иначе → 'on' + PascalCase имени события (например, onKernelException).
  3. Иначе → __invoke().
  4. Иначе → исключение.
```

---

## 11. Сводная таблица способов регистрации

| Способ                                | Где описано        | Когда удобен                                                |
| ------------------------------------- | ------------------ | ----------------------------------------------------------- |
| Тег `kernel.event_listener` в YAML    | `services.yaml`    | Когда нужна гибкая настройка из bundle / по конфигу         |
| `#[AsEventListener]` на классе        | сам класс          | Маленький listener — конфиг и код вместе                    |
| `#[AsEventListener]` на методе        | сам метод          | Один класс слушает много событий разными методами           |
| `EventSubscriberInterface`            | сам класс          | Подписка на несколько событий с разными приоритетами        |
| `$dispatcher->dispatch($event)`       | где угодно         | Диспатч СВОИХ событий из бизнес-логики                      |

---

## 12. Шпаргалка по приоритетам

```text
• Приоритет — целое число (положительное или отрицательное), по умолчанию 0.
• Чем БОЛЬШЕ число → тем РАНЬШЕ выполнится listener.
• Внутренние Symfony-listener'ы обычно в диапазоне -256..256, но твои могут быть любыми.
• Приоритеты СУММИРУЮТСЯ для ВСЕХ listener'ов и subscriber'ов на это событие.
• Чтобы посмотреть итоговый порядок: php bin/console debug:event-dispatcher <event>
```