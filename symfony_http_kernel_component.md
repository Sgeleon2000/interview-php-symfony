# Symfony HttpKernel Component — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [The HttpKernel Component](https://symfony.com/doc/current/components/http_kernel.html), с пояснениями в комментариях.

`HttpKernel` превращает `Request` в `Response`, диспатча события через EventDispatcher. На нём построены Symfony Framework, Drupal и др.

---

## 1. Установка

```bash
composer require symfony/http-kernel
```

---

## 2. Жизненный цикл Request → Response

```text
                       ┌─────────────────────────────┐
        Request  ───►  │  HttpKernel::handle()       │
                       └─────────────────────────────┘
                                    │
        1) kernel.request           │  — может вернуть Response сразу
                                    ▼
        2) Resolve controller       │  — ControllerResolver: ищет callable
                                    ▼
        3) kernel.controller        │  — финальный шанс заменить controller
                                    ▼
        4) Resolve arguments        │  — ArgumentResolver: какие аргументы передать
                                    ▼
        4.5) kernel.controller_arguments
                                    ▼
        5) Call controller          │  — выполняем callable
                                    ▼
        6) kernel.view (если контроллер вернул НЕ-Response)
                                    ▼
        7) kernel.response          │  — модифицируем Response (заголовки, куки)
                                    ▼
                       ┌─────────────────────────────┐
        Response ◄──── │  $response->send()          │
                       └─────────────────────────────┘
                                    │
        8) kernel.terminate         │  — пост-обработка (отправка email и т.п.)

        ※ В любой момент брошено исключение → kernel.exception
```

---

## 3. Полный рабочий пример: standalone HttpKernel

`public/index.php`

```php
<?php
require __DIR__ . '/../vendor/autoload.php';

use Symfony\Component\EventDispatcher\EventDispatcher;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\RequestStack;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Controller\ArgumentResolver;
use Symfony\Component\HttpKernel\Controller\ControllerResolver;
use Symfony\Component\HttpKernel\EventListener\RouterListener;
use Symfony\Component\HttpKernel\HttpKernel;
use Symfony\Component\Routing\Matcher\UrlMatcher;
use Symfony\Component\Routing\RequestContext;
use Symfony\Component\Routing\Route;
use Symfony\Component\Routing\RouteCollection;

// 1) Описываем маршруты. Ключ '_controller' — это PHP callable, который выполнит контроллер.
$routes = new RouteCollection();
$routes->add('hello', new Route('/hello/{name}', [
    '_controller' => function (Request $request): Response {
        // ArgumentResolver автоматически прокинет $request,
        // а $name доступен через $request->attributes (положил туда RouterListener).
        $name = $request->attributes->get('name');
        return new Response("Hello $name");
    },
]));

// 2) Создаём Request из суперглобальных $_GET/$_POST/$_SERVER и т.д.
$request = Request::createFromGlobals();

// 3) UrlMatcher сопоставляет URL → маршрут (используется внутри RouterListener)
$matcher = new UrlMatcher($routes, new RequestContext());

// 4) EventDispatcher — сердце HttpKernel. Все этапы lifecycle = события.
$dispatcher = new EventDispatcher();
// RouterListener слушает kernel.request и кладёт '_controller' в request->attributes.
$dispatcher->addSubscriber(new RouterListener($matcher, new RequestStack()));

// 5) Резолверы:
$controllerResolver = new ControllerResolver(); // получает callable из request->attributes['_controller']
$argumentResolver   = new ArgumentResolver();   // подбирает аргументы для callable

// 6) Создаём ядро
$kernel = new HttpKernel(
    $dispatcher,
    $controllerResolver,
    new RequestStack(),
    $argumentResolver,
);

// 7) Запускаем lifecycle: Request → Response
$response = $kernel->handle($request);

// 8) Отправляем headers + body клиенту
$response->send();

// 9) kernel.terminate — выполняется ПОСЛЕ отправки ответа (тяжёлые задачи).
//    На PHP-FPM и FrankenPHP клиент уже получит ответ, а PHP продолжит работать.
$kernel->terminate($request, $response);
```

---

## 4. Listener'ы на ВСЕ 8 событий жизненного цикла

`src/EventListener/LifecycleSubscriber.php`

```php
<?php
namespace App\EventListener;

use Symfony\Component\EventDispatcher\EventSubscriberInterface;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\Event\ControllerArgumentsEvent;
use Symfony\Component\HttpKernel\Event\ControllerEvent;
use Symfony\Component\HttpKernel\Event\ExceptionEvent;
use Symfony\Component\HttpKernel\Event\FinishRequestEvent;
use Symfony\Component\HttpKernel\Event\RequestEvent;
use Symfony\Component\HttpKernel\Event\ResponseEvent;
use Symfony\Component\HttpKernel\Event\TerminateEvent;
use Symfony\Component\HttpKernel\Event\ViewEvent;
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\KernelEvents;

class LifecycleSubscriber implements EventSubscriberInterface
{
    // Маппинг: event name → метод. Можно указать приоритет: [['onKernelRequest', 100]]
    public static function getSubscribedEvents(): array
    {
        return [
            KernelEvents::REQUEST              => 'onKernelRequest',
            KernelEvents::CONTROLLER           => 'onKernelController',
            KernelEvents::CONTROLLER_ARGUMENTS => 'onControllerArguments',
            KernelEvents::VIEW                 => 'onKernelView',
            KernelEvents::RESPONSE             => 'onKernelResponse',
            KernelEvents::FINISH_REQUEST       => 'onFinishRequest',
            KernelEvents::EXCEPTION            => 'onKernelException',
            KernelEvents::TERMINATE            => 'onKernelTerminate',
        ];
    }

    // ------------- 1) kernel.request -------------
    // Цель: добавить инфу к Request, инициализация, ИЛИ вернуть Response сразу
    // (security, locale, routing). Если задать Response — propagation останавливается.
    public function onKernelRequest(RequestEvent $event): void
    {
        if (!$event->isMainRequest()) {
            return; // не реагируем на sub-requests (важный паттерн!)
        }

        $request = $event->getRequest();
        $request->setLocale($request->headers->get('Accept-Language', 'en'));

        // Пример: гард доступа
        if ($request->getPathInfo() === '/forbidden') {
            $event->setResponse(new Response('Access denied', 403));
            // setResponse() → пропускаем сразу к kernel.response
        }
    }

    // ------------- 3) kernel.controller -------------
    // Цель: инициализация перед вызовом контроллера; можно подменить controller.
    public function onKernelController(ControllerEvent $event): void
    {
        $controller = $event->getController();
        // $event->setController($newController); // подмена

        // Доступ к атрибутам (PHP-атрибутам класса/метода контроллера):
        $attributes = $event->getAttributes(); // ['App\Attribute\Foo' => [FooInstance]]
    }

    // ------------- 4.5) kernel.controller_arguments -------------
    // Цель: модифицировать массив аргументов перед вызовом контроллера.
    public function onControllerArguments(ControllerArgumentsEvent $event): void
    {
        $args = $event->getArguments();
        // $event->setArguments($args);
    }

    // ------------- 6) kernel.view -------------
    // Срабатывает ТОЛЬКО если контроллер вернул НЕ Response (например, массив).
    // Цель: превратить return value в Response.
    public function onKernelView(ViewEvent $event): void
    {
        $result = $event->getControllerResult(); // что вернул controller

        if (is_array($result)) {
            // setResponse → пропускаем к kernel.response (как и в request)
            $event->setResponse(new JsonResponse($result));
        }
    }

    // ------------- 7) kernel.response -------------
    // Цель: модифицировать готовый Response (заголовки, куки, контент).
    public function onKernelResponse(ResponseEvent $event): void
    {
        $response = $event->getResponse();
        $response->headers->set('X-Powered-By', 'My HttpKernel App');
    }

    // ------------- kernel.finish_request -------------
    // Срабатывает в конце КАЖДОГО запроса (включая sub-requests).
    // Полезно для cleanup в long-running процессах.
    public function onFinishRequest(FinishRequestEvent $event): void
    {
        // ...
    }

    // ------------- 9) kernel.exception -------------
    // Цель: поймать исключение и сформировать Response.
    public function onKernelException(ExceptionEvent $event): void
    {
        $exception = $event->getThrowable();

        // Можно понять, идёт ли уже terminate (исключение в kernel.terminate)
        if ($event->isKernelTerminating()) {
            return;
        }

        if ($exception instanceof NotFoundHttpException) {
            $event->setResponse(new Response('Not found, sorry', 404));
            // setResponse → propagation остановлен
        }
    }

    // ------------- 8) kernel.terminate -------------
    // Срабатывает ПОСЛЕ отправки ответа клиенту (PHP-FPM / FrankenPHP).
    // Цель: тяжёлые задачи без задержки ответа (отправка email, аналитика).
    public function onKernelTerminate(TerminateEvent $event): void
    {
        // mail(...); // например
    }
}
```

---

## 5. Кастомный ControllerResolver

`src/Kernel/MyControllerResolver.php`

```php
<?php
namespace App\Kernel;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Controller\ControllerResolverInterface;

/**
 * Задача: по Request вернуть PHP callable (контроллер), который сгенерирует Response.
 * Дефолтный ControllerResolver читает ключ '_controller' из request->attributes
 * (туда его кладёт RouterListener).
 */
class MyControllerResolver implements ControllerResolverInterface
{
    public function getController(Request $request): callable|false
    {
        $controller = $request->attributes->get('_controller');

        if (!$controller) {
            return false; // controller не найден → исключение
        }

        // Если строка вида "App\Controller\HomeController::index" — превращаем в callable
        if (is_string($controller) && str_contains($controller, '::')) {
            [$class, $method] = explode('::', $controller, 2);
            return [new $class(), $method];
        }

        return $controller; // уже callable (Closure, [$obj, 'method'], invokable)
    }
}
```

---

## 6. Кастомный ValueResolver (расширяет ArgumentResolver)

`src/Kernel/UserAgentValueResolver.php`

```php
<?php
namespace App\Kernel;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpKernel\Controller\ValueResolverInterface;
use Symfony\Component\HttpKernel\ControllerMetadata\ArgumentMetadata;

/**
 * ArgumentResolver итерирует аргументы контроллера и ищет, кто умеет их разрешать.
 * Дефолт умеет: атрибуты Request, type-hint Request, variadic. Расширим — добавим
 * автоматическую инъекцию User-Agent в аргумент с именем $userAgent.
 */
class UserAgentValueResolver implements ValueResolverInterface
{
    public function resolve(Request $request, ArgumentMetadata $argument): iterable
    {
        if ($argument->getName() !== 'userAgent' || $argument->getType() !== 'string') {
            return []; // не наш случай — следующий резолвер попробует
        }

        yield $request->headers->get('User-Agent', 'unknown');
    }
}

// В контроллере:
// public function index(string $userAgent): Response { ... }
```

---

## 7. Сброс состояния (ResetInterface) — для FrankenPHP worker mode

`src/Service/StatefulService.php`

```php
<?php
namespace App\Service;

use Symfony\Contracts\Service\ResetInterface;

/**
 * PHP по умолчанию stateless — каждый запрос свежий процесс.
 * Но в FrankenPHP worker mode процесс живёт между запросами, и сервисы
 * могут накапливать состояние → memory leaks.
 * Реализуем ResetInterface — kernel сам вызовет reset() после каждого запроса.
 */
class StatefulService implements ResetInterface
{
    private array $cache = [];
    private int $counter = 0;

    public function add(string $key, mixed $value): void
    {
        $this->cache[$key] = $value;
        $this->counter++;
    }

    public function reset(): void
    {
        // Очищаем всё накопленное между запросами
        $this->cache   = [];
        $this->counter = 0;
    }
}
```

---

## 8. Sub-requests (под-запросы)

`src/Controller/PageController.php`

```php
<?php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\HttpKernelInterface;
use Symfony\Component\Routing\Attribute\Route;

class PageController extends AbstractController
{
    #[Route('/page')]
    public function page(HttpKernelInterface $kernel): Response
    {
        // Готовим sub-request — отдельный Request, не от пользователя.
        $subRequest = new Request();
        $subRequest->attributes->set('_controller', 'App\Controller\WidgetController::render');
        $subRequest->attributes->set('widgetId', 42);

        // Если ответ должен быть в другом формате (по умолчанию html):
        $subRequest->attributes->set('_format', 'json');

        // ВАЖНО: второй аргумент HttpKernelInterface::SUB_REQUEST
        // Многие listener'ы (security и т.п.) реагируют только на MAIN_REQUEST.
        // В своих listener'ах проверяй $event->isMainRequest().
        $widgetResponse = $kernel->handle($subRequest, HttpKernelInterface::SUB_REQUEST);

        return new Response('Главная страница + виджет: ' . $widgetResponse->getContent());
    }
}
```

---

## 9. Locating Resources — логические пути бандлов

```php
<?php
// HttpKernel позволяет ссылаться на ресурсы бандлов логическим путём,
// независимо от того, где бандл физически установлен.

// Логический путь:  '@FooBundle/Resources/config/services.xml'
// Физический путь:  '/var/www/.../vendor/acme/foo-bundle/Resources/config/services.xml'

$physicalPath = $kernel->locateResource('@FooBundle/Resources/config/services.xml');

// Используется внутри Symfony для импорта конфигов, шаблонов, переводов и т.п.
```

---

## 10. Кастомное исключение для 404

`src/Exception/PageNotFoundException.php`

```php
<?php
namespace App\Exception;

use Symfony\Component\HttpKernel\Exception\HttpException;

/**
 * HttpExceptionInterface → ErrorListener возьмёт status code и headers
 * автоматически. Бросаешь в контроллере → клиент получит правильный 404.
 */
class PageNotFoundException extends HttpException
{
    public function __construct(string $message = 'Page not found')
    {
        parent::__construct(statusCode: 404, message: $message, headers: [
            'X-Reason' => 'not-found',
        ]);
    }
}
```

---

## 11. Таблица событий жизненного цикла

| Событие                       | Константа                            | Объект события              | Назначение                                                          |
| ----------------------------- | ------------------------------------ | --------------------------- | ------------------------------------------------------------------- |
| `kernel.request`              | `KernelEvents::REQUEST`              | `RequestEvent`              | Инициализация Request, ранний выход с Response (security, routing)  |
| `kernel.controller`           | `KernelEvents::CONTROLLER`           | `ControllerEvent`           | Подмена/инициализация контроллера до вызова                         |
| `kernel.controller_arguments` | `KernelEvents::CONTROLLER_ARGUMENTS` | `ControllerArgumentsEvent`  | Модификация массива аргументов контроллера                          |
| `kernel.view`                 | `KernelEvents::VIEW`                 | `ViewEvent`                 | Превратить НЕ-Response (массив, объект) в Response                  |
| `kernel.response`             | `KernelEvents::RESPONSE`             | `ResponseEvent`             | Модифицировать готовый Response (headers, cookies, content)         |
| `kernel.finish_request`       | `KernelEvents::FINISH_REQUEST`       | `FinishRequestEvent`        | Cleanup в конце каждого запроса (включая sub-requests)              |
| `kernel.terminate`            | `KernelEvents::TERMINATE`            | `TerminateEvent`            | Тяжёлая работа ПОСЛЕ отправки ответа (PHP-FPM / FrankenPHP)         |
| `kernel.exception`            | `KernelEvents::EXCEPTION`            | `ExceptionEvent`            | Поймать исключение и сформировать error Response                    |

---

## 12. Важные нюансы

```text
• setResponse() в kernel.request, kernel.view, kernel.exception
  ⇒ propagation останавливается, listener'ы с меньшим приоритетом НЕ выполнятся.

• Controller обязан вернуть ЧТО-ТО. return null → исключение.

• Если controller вернул не Response, и никто в kernel.view не задал Response → исключение.

• kernel.terminate работает корректно только на:
   - PHP-FPM (использует fastcgi_finish_request)
   - FrankenPHP
  На других SAPI ответ дойдёт до клиента ТОЛЬКО после завершения terminate-слушателей.

• ExceptionEvent::isKernelTerminating() — true, если исключение брошено
  во время kernel.terminate (иначе можно уйти в бесконечный цикл).

• В sub-request listener'ы вроде Security часто не должны срабатывать.
  Всегда проверяй $event->isMainRequest() в начале.

• В FrankenPHP worker mode реализуй ResetInterface в stateful-сервисах,
  иначе будут memory leaks между запросами.
```

---

## 13. Регистрация subscriber'а в Symfony Framework

`config/services.yaml`

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true  # благодаря этому EventSubscriberInterface
                             # автоматически регистрируется на свои события

    App\:
        resource: '../src/'

    # Можно явно — но при autoconfigure не требуется:
    # App\EventListener\LifecycleSubscriber:
    #     tags: [kernel.event_subscriber]

    # Кастомный ValueResolver — должен быть с тегом controller.argument_value_resolver
    App\Kernel\UserAgentValueResolver:
        tags:
            - { name: controller.argument_value_resolver, priority: 100 }
```