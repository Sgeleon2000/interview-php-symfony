# Symfony Service Container — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из официальной документации [Symfony Service Container](https://symfony.com/doc/current/service_container.html), с пояснениями в комментариях.

---

## 1. Простой сервис

`src/Service/MessageGenerator.php`

Любой класс в `src/` автоматически становится сервисом благодаря дефолтному конфигу `services.yaml` (`resource: '../src/'`). ID сервиса = FQCN класса.

```php
<?php
namespace App\Service;

use Psr\Log\LoggerInterface;
use Symfony\Component\DependencyInjection\Attribute\When;
use Symfony\Component\DependencyInjection\Attribute\Autoconfigure;

#[Autoconfigure(public: true)] // делает сервис публичным ($container->get() работает)
#[When(env: 'dev')]            // регистрируется ТОЛЬКО в окружении dev (есть и #[WhenNot])
class MessageGenerator
{
    // Внедрение зависимостей через конструктор (constructor injection).
    // Благодаря autowire: true контейнер сам подставит logger по type-hint'у.
    public function __construct(
        private LoggerInterface $logger,        // автоматически найден по типу
        private string $adminEmail,             // скаляр — нельзя autowire, передаётся явно
        private $generateMessageHash,           // callable — внедрён через !closure
    ) {}

    public function getHappyMessage(): string
    {
        $this->logger->info('Генерирую сообщение для ' . $this->adminEmail);
        $hash = ($this->generateMessageHash)(); // вызываем инжектнутый callable
        return "Отлично! [$hash]";
    }
}
```

---

## 2. Сервис с несколькими зависимостями

`src/Service/SiteUpdateManager.php`

```php
<?php
namespace App\Service;

use Symfony\Component\Mailer\MailerInterface;
use Symfony\Component\Mime\Email;

class SiteUpdateManager
{
    // Оба аргумента — сервисы, autowire подставит их по type-hint.
    public function __construct(
        private MessageGenerator $messageGenerator,
        private MailerInterface $mailer,
        private string $adminEmail, // настраивается вручную через arguments / bind
    ) {}

    public function notifyOfSiteUpdate(): bool
    {
        $email = (new Email())
            ->from('admin@example.com')
            ->to($this->adminEmail)
            ->subject('Обновление сайта')
            ->text($this->messageGenerator->getHappyMessage());

        $this->mailer->send($email);
        return true;
    }
}
```

---

## 3. Invokable-сервис для инъекции как closure

`src/Hash/MessageHashGenerator.php`

```php
<?php
namespace App\Hash;

class MessageHashGenerator
{
    public function __invoke(): string
    {
        return substr(md5(uniqid()), 0, 8);
    }
}
```

---

## 4. Функциональный интерфейс + адаптер через `#[AutowireCallable]`

```php
<?php
namespace App\Service;

// Интерфейс с одним методом = "functional interface"
interface MessageFormatterInterface
{
    public function format(string $message, array $parameters): string;
}

class MessageUtils
{
    public function format(string $message, array $parameters): string
    {
        return strtr($message, $parameters);
    }
}

use Symfony\Component\DependencyInjection\Attribute\AutowireCallable;

class Mailer
{
    public function __construct(
        // Symfony сгенерирует адаптер: MessageFormatterInterface::format()
        // будет проксироваться в MessageUtils::format().
        #[AutowireCallable(service: MessageUtils::class, method: 'format')]
        private MessageFormatterInterface $formatter,
    ) {}
}
```

---

## 5. Контроллер — получение сервисов из контейнера

`src/Controller/ProductController.php`

```php
<?php
namespace App\Controller;

use App\Service\MessageGenerator;
use App\Service\SiteUpdateManager;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductController extends AbstractController
{
    #[Route('/products/new')]
    public function new(
        // Type-hint в action — контейнер сам найдёт и передаст нужные сервисы
        MessageGenerator $messageGenerator,
        SiteUpdateManager $siteUpdateManager,
    ): Response {
        $this->addFlash('success', $messageGenerator->getHappyMessage());
        $siteUpdateManager->notifyOfSiteUpdate();

        return new Response('OK');
    }
}
```

---

## 6. Конфигурация контейнера

`config/services.yaml`

```yaml
# Параметры контейнера (плоские key-value, точка — конвенция Symfony)
parameters:
    app.admin_email: 'manager@example.com'

services:
    # ---- Дефолты, применяются ко всем сервисам этого файла ----
    _defaults:
        autowire: true       # автоматически подставлять зависимости по type-hint
        autoconfigure: true  # авто-тегирование (event subscriber, twig extension и т.д.)
        # bind — передать значения в ЛЮБОЙ сервис по имени/типу аргумента
        bind:
            $adminEmail: '%app.admin_email%'                       # по имени
            Psr\Log\LoggerInterface: '@monolog.logger.request'     # по типу
            Psr\Log\LoggerInterface $requestLogger: '@monolog.logger.request' # имя + тип

    # ---- Массовая регистрация: все классы из src/ становятся сервисами ----
    App\:
        resource: '../src/'
        exclude:
            - '../src/{DependencyInjection,Entity,Kernel.php}'

    # ---- Явная (manual) конфигурация одного сервиса ----
    App\Service\MessageGenerator:
        arguments:
            $logger: '@monolog.logger.request'        # '@' = ID сервиса (не строка!)
            $adminEmail: '%app.admin_email%'          # %...% = параметр контейнера
            $generateMessageHash: !closure '@App\Hash\MessageHashGenerator' # инъекция callable

    # ---- Передача разных типов значений в аргументы ----
    App\Service\SomeService:
        arguments:
            - 'Foo'                                   # строка
            - true                                    # bool
            - 7                                       # int
            - !php/const PDO::FETCH_NUM               # PHP-константа
            - '@?optional.service'                    # null, если сервис не существует
            - !abstract 'будет задано CompilerPass'   # абстрактный аргумент

    # ---- Два разных сервиса из одного класса + alias ----
    site_update_manager.superadmin:
        class: App\Service\SiteUpdateManager
        autowire: false
        arguments:
            - '@App\Service\MessageGenerator'
            - '@mailer'
            - 'superadmin@example.com'

    site_update_manager.normal_users:
        class: App\Service\SiteUpdateManager
        autowire: false
        arguments:
            - '@App\Service\MessageGenerator'
            - '@mailer'
            - 'contact@example.com'

    # Alias: по type-hint SiteUpdateManager будет приходить superadmin-версия
    App\Service\SiteUpdateManager: '@site_update_manager.superadmin'

    # ---- Публичный сервис (можно получить через $container->get()) ----
    App\Service\PublicService:
        public: true

    # ---- Адаптер функционального интерфейса через конфиг ----
    app.message_formatter:
        class: App\Service\MessageFormatterInterface
        from_callable: [!service { class: App\Service\MessageUtils }, 'format']

    # ---- Несколько определений для одного namespace (через ключ namespace) ----
    command_handlers:
        namespace: App\Domain\
        resource: '../src/Domain/*/CommandHandler'
        tags: [command_handler]

# ---- Сервисы, регистрируемые только в определённом окружении ----
when@dev:
    services:
        _defaults:            # ВАЖНО: _defaults НЕ наследуются — задавать заново
            autowire: true
            autoconfigure: true
        App\Service\DevOnlyService: ~

when@test:
    services:
        # remove — убрать сервис из контейнера в этом окружении
        App\Service\PublicService: null
```

---

## 7. Полезные консольные команды

```bash
php bin/console debug:autowiring                   # список всех типов, доступных для autowire
php bin/console debug:autowiring logger            # фильтр (например, все логгеры)
php bin/console debug:container                    # все сервисы контейнера
php bin/console lint:container                     # проверка корректности конфигурации
php bin/console lint:container --resolve-env-vars  # + проверка env переменных
```

---

## Что покрыто из документации

- Получение сервисов через type-hint в контроллере
- Создание своих сервисов
- Автоматическая загрузка через `resource` / `exclude`
- Ограничение по окружению через `#[When]` / `#[WhenNot]` / `when@dev`
- Dependency injection через конструктор
- Ручная настройка аргументов
- Передача скаляров / констант / коллекций
- Ссылки на сервисы через `@` и `service()`
- Параметры контейнера (`%...%`)
- Выбор конкретного сервиса (`$logger: '@monolog.logger.request'`)
- Удаление сервиса
- Инъекция closure через `!closure`
- Биндинг аргументов (`bind`)
- Абстрактные аргументы (`!abstract`)
- Опции `autowire` и `autoconfigure`
- Публичные / приватные сервисы
- Алиасы
- Множественные определения через `namespace`
- Адаптеры функциональных интерфейсов через `#[AutowireCallable]` / `from_callable`