# Symfony Kernel Configuration — короткий пример всех параметров

Этот файл содержит компактный пример, покрывающий все опции из документации [Configuring in the Kernel](https://symfony.com/doc/current/reference/configuration/kernel.html), с пояснениями в комментариях.

Все эти параметры доступны в контейнере как `%kernel.имя%` и могут быть переопределены через методы класса `Kernel` или через `services.yaml`.

---

## 1. Кастомный Kernel с переопределением всех методов

`src/Kernel.php`

```php
<?php
namespace App;

use Symfony\Bundle\FrameworkBundle\Kernel\MicroKernelTrait;
use Symfony\Component\HttpKernel\Kernel as BaseKernel;

class Kernel extends BaseKernel
{
    use MicroKernelTrait;

    // -------------------------------------------------------------------------
    // kernel.project_dir — абсолютный путь к корню проекта
    // По умолчанию: каталог, где лежит composer.json
    // Используется для построения путей относительно корня проекта.
    // -------------------------------------------------------------------------
    public function getProjectDir(): string
    {
        // ВАЖНО: без trailing slash. Например: '/var/www/example.com'
        return \dirname(__DIR__);
    }

    // -------------------------------------------------------------------------
    // kernel.cache_dir — путь к директории кэша (read-write)
    // По умолчанию: %kernel.project_dir%/var/cache/%kernel.environment%
    // Сюда приложение пишет данные во время работы.
    // -------------------------------------------------------------------------
    public function getCacheDir(): string
    {
        return $this->getProjectDir() . '/var/cache/' . $this->environment;
    }

    // -------------------------------------------------------------------------
    // kernel.build_dir — путь к директории сборки (read-only)
    // По умолчанию: совпадает с getCacheDir().
    // Используется, чтобы отделить read-only кэш (скомпилированный контейнер)
    // от read-write кэша (cache pools). Полезно для Docker, AWS Lambda и т.п.
    // Можно задать через переменную окружения APP_BUILD_DIR.
    // -------------------------------------------------------------------------
    public function getBuildDir(): string
    {
        return $this->getProjectDir() . '/var/build/' . $this->environment;
    }

    // -------------------------------------------------------------------------
    // kernel.logs_dir — путь к директории логов
    // По умолчанию: %kernel.project_dir%/var/log
    // -------------------------------------------------------------------------
    public function getLogDir(): string
    {
        return $this->getProjectDir() . '/var/log';
    }

    // -------------------------------------------------------------------------
    // kernel.share_dir — путь к общей (shared) cache-директории
    // По умолчанию: совпадает с getCacheDir().
    // -------------------------------------------------------------------------
    public function getShareDir(): string
    {
        return $this->getProjectDir() . '/var/share';
    }

    // -------------------------------------------------------------------------
    // kernel.charset — кодировка символов приложения
    // По умолчанию: 'UTF-8'
    // -------------------------------------------------------------------------
    public function getCharset(): string
    {
        return 'UTF-8'; // можно вернуть 'ISO-8859-1' и т.д.
    }

    // -------------------------------------------------------------------------
    // kernel.container_class — уникальное имя класса скомпилированного контейнера
    // По умолчанию: 'App_KernelDevDebugContainer' (зависит от namespace, env, debug)
    // Важно при работе с multiple kernels — у каждого должен быть свой ID.
    // -------------------------------------------------------------------------
    public function getContainerClass(): string
    {
        // Стандартное поведение можно сохранить:
        return parent::getContainerClass();
        // Или задать свой ID:
        // return sprintf('AcmeKernel%s', random_int(10_000, 99_999));
    }
}
```

---

## 2. Использование параметров kernel в сервисах

`src/Service/PathHelper.php`

```php
<?php
namespace App\Service;

class PathHelper
{
    public function __construct(
        // Все параметры контейнера доступны через '%kernel.*%' в конфиге,
        // а здесь — внедряются как обычные строки/булы.
        private string $projectDir,      // %kernel.project_dir%
        private string $cacheDir,        // %kernel.cache_dir%
        private string $logsDir,         // %kernel.logs_dir%
        private string $environment,     // %kernel.environment%
        private string $runtimeEnv,      // %kernel.runtime_environment%
        private bool $debug,             // %kernel.debug%
        private string $charset,         // %kernel.charset%
        private string $secret,          // %kernel.secret%
        private string $defaultLocale,   // %kernel.default_locale%
        private array $enabledLocales,   // %kernel.enabled_locales%
        private array $bundles,          // %kernel.bundles%
        private array $bundlesMetadata,  // %kernel.bundles_metadata%
        private bool $isWeb,             // %kernel.runtime_mode.web%
        private bool $isCli,             // %kernel.runtime_mode.cli%
        private bool $isWorker,          // %kernel.runtime_mode.worker%
    ) {}

    public function describe(): array
    {
        return [
            'project'  => $this->projectDir,
            'env'      => $this->environment,         // dev/prod/test (конфиг)
            'runtime'  => $this->runtimeEnv,          // staging/production (место деплоя)
            'debug'    => $this->debug,               // вкл./выкл. debug
            'locale'   => $this->defaultLocale,
            'locales'  => $this->enabledLocales,
            'bundles'  => array_keys($this->bundles), // список зарегистрированных бандлов
            'mode'     => match (true) {
                $this->isWorker => 'worker',          // FrankenPHP и подобное
                $this->isWeb    => 'web',
                $this->isCli    => 'cli',
                default         => 'unknown',
            },
        ];
    }
}
```

---

## 3. Конфигурация контейнера

`config/services.yaml`

```yaml
parameters:
    # ---- kernel.container_build_time ----
    # Для reproducible builds: чтобы container.build_time не менялся при каждой
    # компиляции, задайте фиксированное значение (timestamp).
    # Связанные авто-параметры: container.build_hash, container.build_time, container.build_id
    kernel.container_build_time: '1234567890'

services:
    _defaults:
        autowire: true
        autoconfigure: true

    App\:
        resource: '../src/'
        exclude: '../src/{DependencyInjection,Entity,Kernel.php}'

    # ---- Передача всех kernel-параметров в сервис ----
    App\Service\PathHelper:
        arguments:
            $projectDir:      '%kernel.project_dir%'        # корень проекта
            $cacheDir:        '%kernel.cache_dir%'          # var/cache/<env>
            $logsDir:         '%kernel.logs_dir%'           # var/log
            $environment:     '%kernel.environment%'        # dev/prod/test (config)
            $runtimeEnv:      '%kernel.runtime_environment%' # APP_RUNTIME_ENV (deploy)
            $debug:           '%kernel.debug%'              # bool
            $charset:         '%kernel.charset%'            # UTF-8
            $secret:          '%kernel.secret%'             # APP_SECRET
            $defaultLocale:   '%kernel.default_locale%'     # framework.default_locale
            $enabledLocales:  '%kernel.enabled_locales%'    # framework.enabled_locales
            $bundles:         '%kernel.bundles%'            # ['FrameworkBundle' => FQCN, ...]
            $bundlesMetadata: '%kernel.bundles_metadata%'   # [name => ['path', 'namespace']]
            $isWeb:           '%kernel.runtime_mode.web%'   # web-окружение?
            $isCli:           '%kernel.runtime_mode.cli%'   # CLI-окружение?
            $isWorker:        '%kernel.runtime_mode.worker%' # long-running (FrankenPHP)?

# ---- framework-параметры, которые мапятся в kernel.* ----
framework:
    secret: '%env(APP_SECRET)%'      # → kernel.secret
    default_locale: 'en'             # → kernel.default_locale
    enabled_locales: ['en', 'ru']    # → kernel.enabled_locales
    error_controller: null           # → kernel.error_controller
    http_method_override: false      # → kernel.http_method_override
    allowed_http_method_override: [] # → kernel.allowed_http_method_override
    trust_x_sendfile_type_header: false # → kernel.trust_x_sendfile_type_header
    trusted_headers:                 # → kernel.trusted_headers
        - 'x-forwarded-for'
        - 'x-forwarded-host'
        - 'x-forwarded-proto'
        - 'x-forwarded-port'
        - 'x-forwarded-prefix'
    trusted_hosts: ['^example\.com$'] # → kernel.trusted_hosts
    trusted_proxies: '127.0.0.1,10.0.0.0/8' # → kernel.trusted_proxies
```

---

## 4. Переменные окружения

`.env`

```bash
# kernel.environment — какой набор конфига использовать (dev/prod/test)
APP_ENV=prod

# kernel.runtime_environment — где приложение реально запущено
# Позволяет запускать с конфигом prod в разных местах (staging/production)
APP_RUNTIME_ENV=staging

# kernel.runtime_mode — режим работы (web=1&worker=0, web=1&worker=1 и т.д.)
APP_RUNTIME_MODE=web=1

# kernel.debug — режим отладки (true в dev, false в prod)
APP_DEBUG=0

# kernel.secret — секрет приложения для CSRF-токенов, подписей и т.д.
APP_SECRET=s3cr3t

# kernel.build_dir — кастомный путь к build-директории (для read-only ФС)
APP_BUILD_DIR=/tmp/app-build

# APP_PROJECT_DIR — read-only переменная с путём к проекту
# Перезаписать её нельзя, для кастомизации используйте Kernel::getProjectDir()
```

---

## 5. Получение параметров в коде (контроллер)

```php
<?php
namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpKernel\KernelInterface;
use Symfony\Component\Routing\Attribute\Route;

class DebugController extends AbstractController
{
    #[Route('/debug/kernel')]
    public function info(
        KernelInterface $kernel,                                 // сам Kernel-объект
        #[Autowire('%kernel.project_dir%')] string $projectDir,  // через атрибут
        #[Autowire('%kernel.environment%')] string $env,
        #[Autowire('%kernel.debug%')] bool $debug,
        #[Autowire('%kernel.bundles%')] array $bundles,
    ): JsonResponse {
        return new JsonResponse([
            // Через KernelInterface — методы класса
            'projectDir'    => $kernel->getProjectDir(),
            'cacheDir'      => $kernel->getCacheDir(),
            'buildDir'      => $kernel->getBuildDir(),
            'logDir'        => $kernel->getLogDir(),
            'environment'   => $kernel->getEnvironment(),
            'debug'         => $kernel->isDebug(),
            'charset'       => $kernel->getCharset(),
            'containerClass'=> $kernel->getContainerClass(),
            'bundles'       => array_keys($kernel->getBundles()),

            // Через #[Autowire] — параметры контейнера
            'fromAutowire' => compact('projectDir', 'env', 'debug', 'bundles'),
        ]);
    }
}
```

---

## 6. Полный список kernel-параметров

| Параметр                                | Тип       | По умолчанию                                              | Описание                                                          |
| --------------------------------------- | --------- | --------------------------------------------------------- | ----------------------------------------------------------------- |
| `kernel.build_dir`                      | `string`  | `getCacheDir()`                                           | Read-only директория сборки (контейнер). `APP_BUILD_DIR`          |
| `kernel.bundles`                        | `array`   | `[]`                                                      | `['FrameworkBundle' => FQCN, ...]`                                |
| `kernel.bundles_metadata`               | `array`   | `[]`                                                      | `[name => ['path', 'namespace']]`                                 |
| `kernel.cache_dir`                      | `string`  | `var/cache/<env>`                                         | Read-write кэш                                                    |
| `kernel.charset`                        | `string`  | `UTF-8`                                                   | Кодировка                                                         |
| `kernel.container_build_time`           | `string`  | `time()`                                                  | Timestamp компиляции (для reproducible builds — фиксируется)      |
| `kernel.container_class`                | `string`  | `App_KernelDevDebugContainer`                             | Имя класса контейнера (важно для multiple kernels)                |
| `kernel.debug`                          | `boolean` | (аргумент бута)                                           | Режим debug                                                       |
| `kernel.default_locale`                 | `string`  | `framework.default_locale`                                | Локаль по умолчанию                                               |
| `kernel.enabled_locales`                | `array`   | `framework.enabled_locales`                               | Список локалей                                                    |
| `kernel.environment`                    | `string`  | (аргумент бута)                                           | Конфиг-окружение (dev/prod/test)                                  |
| `kernel.error_controller`               | mixed     | `framework.error_controller`                              | Контроллер ошибок                                                 |
| `kernel.http_method_override`           | `bool`    | `framework.http_method_override`                          | Поддержка `_method`                                               |
| `kernel.allowed_http_method_override`   | `array`   | `framework.allowed_http_method_override`                  | Разрешённые HTTP-методы для override                              |
| `kernel.logs_dir`                       | `string`  | `var/log`                                                 | Логи                                                              |
| `kernel.project_dir`                    | `string`  | каталог `composer.json`                                   | Корень проекта                                                    |
| `kernel.runtime_environment`            | `string`  | `%env(default:kernel.environment:APP_RUNTIME_ENV)%`       | Куда задеплоено (staging/production)                              |
| `kernel.runtime_mode`                   | `string`  | `%env(...:APP_RUNTIME_MODE)%`                             | Query string: `web=1&worker=0`                                    |
| `kernel.runtime_mode.web`               | `bool`    | вычисляется                                               | Web-окружение?                                                    |
| `kernel.runtime_mode.cli`               | `bool`    | `!kernel.runtime_mode.web`                                | CLI-окружение?                                                    |
| `kernel.runtime_mode.worker`            | `bool`    | вычисляется                                               | Long-running (FrankenPHP)?                                        |
| `kernel.secret`                         | `string`  | `%env(APP_SECRET)%`                                       | Секрет приложения                                                 |
| `kernel.share_dir`                      | `string`  | `getCacheDir()`                                           | Shared cache directory                                            |
| `kernel.trust_x_sendfile_type_header`   | `bool`    | `framework.trust_x_sendfile_type_header`                  | Доверять X-Sendfile-Type заголовку                                |
| `kernel.trusted_headers`                | `array`   | `framework.trusted_headers`                               | Доверенные заголовки                                              |
| `kernel.trusted_hosts`                  | `array`   | `framework.trusted_hosts`                                 | Доверенные хосты (regex)                                          |
| `kernel.trusted_proxies`                | `string`  | `framework.trusted_proxies`                               | Доверенные прокси (IP/CIDR)                                       |

---

## 7. Ключевая разница: `environment` vs `runtime_environment`

```text
kernel.environment          → КАКОЙ КОНФИГ загружать (dev/prod/test)
                              Определяет, какие config/packages/<env>/*.yaml применятся.

kernel.runtime_environment  → ГДЕ приложение реально работает (staging/production)
                              Никак не влияет на загрузку конфига, только информативно.

Пример: можно запустить с APP_ENV=prod (полный prod-конфиг)
        и APP_RUNTIME_ENV=staging (чтобы код понимал, что это staging-сервер).
```