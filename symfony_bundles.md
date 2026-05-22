# Symfony Bundle System — короткий пример всех возможностей

Этот файл содержит компактный пример, покрывающий всё из документации [The Bundle System](https://symfony.com/doc/current/bundles.html), с пояснениями в комментариях.

> ⚠️ **ВАЖНО:** Начиная с Symfony 4.0 **НЕ рекомендуется** организовывать код своего приложения через bundle'ы. Bundle'ы нужны **только для шаринга** кода и функциональности между приложениями (плагины, переиспользуемые библиотеки). Свой код пиши прямо в `src/` без bundle-структуры.

---

## 1. Включение bundle'ов в приложении

`config/bundles.php`

```php
<?php
// Список ВСЕХ активных bundle'ов в приложении.
// Ключ — FQCN bundle-класса, значение — массив окружений, где он включён.
return [
    // 'all' → включён во всех окружениях (dev, prod, test, любых других)
    Symfony\Bundle\FrameworkBundle\FrameworkBundle::class => ['all' => true],

    // Включён ТОЛЬКО в окружении 'dev' (полезно для дебаг-инструментов)
    Symfony\Bundle\DebugBundle\DebugBundle::class => ['dev' => true],

    // Включён В НЕСКОЛЬКИХ окружениях, но НЕ в 'prod'
    Symfony\Bundle\WebProfilerBundle\WebProfilerBundle::class => ['dev' => true, 'test' => true],

    // Наш собственный bundle — включён везде
    Acme\BlogBundle\AcmeBlogBundle::class => ['all' => true],
];
```

При использовании **Symfony Flex** этот файл редактируется автоматически: рецепты включают/выключают bundle при `composer require` / `composer remove`. Вручную лезть туда обычно не нужно.

---

## 2. Минимальный bundle-класс

`src/AcmeBlogBundle.php`

```php
<?php
namespace Acme\BlogBundle;

use Symfony\Component\HttpKernel\Bundle\AbstractBundle;

/**
 * Минимальный bundle — пустой класс, унаследованный от AbstractBundle.
 * Этого достаточно, чтобы Symfony распознал и подключил bundle.
 *
 * AbstractBundle (Symfony 6.1+) — современный базовый класс, который сам
 * умеет регистрировать конфиги, extension'ы, compiler passes без бойлерплейта.
 *
 * Для совместимости со старыми версиями Symfony нужно наследоваться от
 * Symfony\Component\HttpKernel\Bundle\Bundle.
 */
class AcmeBlogBundle extends AbstractBundle
{
    // Класс может быть пустым — но это мощная точка расширения:
    // здесь можно настроить namespace конфига, build container, etc.
}
```

### Конвенции именования

```text
✓ AcmeBlogBundle      — стандартная форма: <Vendor> + <Name> + "Bundle"
✓ BlogBundle          — можно короче, если "vendor" не нужен (для приватных bundle)
✗ acme_blog_bundle    — НЕТ, только StudlyCaps
✗ Bundle\Acme\Blog    — namespace должен быть <Vendor>\<Name>Bundle
```

---

## 3. Структура директорий bundle'а

```text
my-bundle/
├── assets/              ← Исходники веб-ассетов (TS, JS, Sass, CSS),
│                          а также Stimulus-контроллеры. НЕ в public/.
│
├── config/              ← Конфигурация bundle'а
│   ├── services.php       (DI-определения)
│   └── routes.php         (роуты)
│
├── public/              ← Скомпилированные / готовые ассеты
│                          (картинки, итоговые CSS/JS).
│                          Копируются в public/ приложения через
│                          `php bin/console assets:install`.
│
├── src/                 ← ВСЯ PHP-логика bundle'а
│   ├── AcmeBlogBundle.php          (главный класс — корень src/)
│   ├── Controller/
│   │   └── CategoryController.php
│   ├── DependencyInjection/        (Extension, Configuration)
│   └── EventSubscriber/
│
├── templates/           ← Twig-шаблоны, обычно по контроллерам
│   └── category/
│       └── show.html.twig
│
├── tests/               ← Все тесты bundle'а
│
├── translations/        ← Файлы переводов: <domain>.<locale>.<format>
│   ├── AcmeBlogBundle.en.xlf
│   └── AcmeBlogBundle.ru.xlf
│
├── composer.json        ← Метаданные пакета и autoload (см. ниже)
└── README.md
```

---

## 4. `composer.json` bundle'а с PSR-4 autoload

`composer.json`

```json
{
    "name": "acme/blog-bundle",
    "type": "symfony-bundle",
    "description": "Acme Blog Bundle",
    "license": "MIT",
    "require": {
        "php": ">=8.2",
        "symfony/framework-bundle": "^7.0|^8.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^11.0",
        "symfony/phpunit-bridge": "^7.0|^8.0"
    },
    "autoload": {
        "psr-4": {
            "Acme\\BlogBundle\\": "src/"
        }
    },
    "autoload-dev": {
        "psr-4": {
            "Acme\\BlogBundle\\Tests\\": "tests/"
        }
    }
}
```

Ключевое:
- `"type": "symfony-bundle"` — позволяет Symfony Flex автоматически зарегистрировать bundle при `composer require`.
- `"psr-4"` — namespace bundle'а указывает на `src/`. Главный класс лежит прямо в корне `src/` (`src/AcmeBlogBundle.php`).
- `"autoload-dev"` — отдельный namespace для тестов, чтобы они не попали в боевой пакет.

---

## 5. Локальная разработка bundle'а: Path Repository

Когда bundle ещё **не опубликован** (не на Packagist), но нужно протестировать его в реальном Symfony-приложении.

### Шаг 1 — добавить локальный путь в `composer.json` приложения

```json
{
    "repositories": [
        {
            "type": "path",
            "url": "/path/to/your/AcmeBlogBundle"
        }
    ],
    "require": {
        "acme/blog-bundle": "*"
    }
}
```

### Шаг 2 — установить bundle обычной командой

```bash
composer require acme/blog-bundle
```

### Что произойдёт

```text
• Composer создаст SYMLINK из vendor/acme/blog-bundle → /path/to/your/AcmeBlogBundle
• Любая правка в исходниках bundle'а МГНОВЕННО видна в приложении
• Не нужно ничего пересобирать или переустанавливать
```

### Шаг 3 — включить bundle

`config/bundles.php`

```php
<?php
return [
    // ...
    Acme\BlogBundle\AcmeBlogBundle::class => ['all' => true],
];
```

---

## 6. Локальная разработка УЖЕ опубликованного bundle'а

Когда bundle **уже на Packagist**, но нужно править его локально и сразу видеть изменения в приложении.

```bash
# Удаляем установленную через composer версию из vendor
rm -rf vendor/acme/blog-bundle/

# Создаём симлинк на свою локальную копию
ln -s ~/Projects/AcmeBlogBundle/ vendor/acme/blog-bundle
```

Symfony теперь использует твою локальную копию. Можно править код, гонять тесты, видеть изменения мгновенно.

**Возврат к "нормальному" состоянию:**

```bash
# Удаляем симлинк и переустанавливаем пакет из Packagist
rm vendor/acme/blog-bundle
composer install
# или
composer require acme/blog-bundle
```

---

## 7. Сводная таблица директорий

| Директория      | Назначение                                                                |
| --------------- | ------------------------------------------------------------------------- |
| `assets/`       | Исходники веб-ассетов: TypeScript, JavaScript, Sass, CSS, Stimulus и т.п. |
| `config/`       | Конфигурация bundle'а — services, routes                                  |
| `public/`       | Готовые статические ассеты, копируются в `public/` приложения             |
| `src/`          | Вся PHP-логика bundle'а (контроллеры, сервисы, DI, ивенты и т.д.)         |
| `templates/`    | Twig-шаблоны, по соглашению — по подпапкам контроллеров                   |
| `tests/`        | Тесты (PHPUnit, etc.)                                                     |
| `translations/` | Файлы переводов в формате `<domain>.<locale>.<format>` (xlf, yaml, php)   |

---

## 8. Сравнение `AbstractBundle` vs `Bundle`

```text
AbstractBundle (Symfony 6.1+)
   ✓ Современный способ создания bundle.
   ✓ Минимум бойлерплейта — сам подгружает services/, configures container.
   ✓ Удобные методы: configure(), loadExtension(), build().
   ✗ Несовместим со старыми Symfony (<6.1).

Bundle (классический)
   ✓ Совместим с любыми версиями Symfony.
   ✗ Нужно отдельно создавать Extension, Configuration, etc.
   ✗ Больше шаблонного кода.

ПРАВИЛО:
   • Новый bundle, поддерживающий только Symfony 6.1+   → AbstractBundle
   • Bundle, поддерживающий старые версии Symfony       → Bundle
```

---

## 9. Главное про bundle'ы

```text
✗ НЕ ИСПОЛЬЗУЙ bundle'ы для организации кода СВОЕГО приложения.
   Пиши логику прямо в src/ без AcmeAppBundle, BlogBundle и т.п.
   (См. https://symfony.com/doc/current/best_practices.html)

✓ ИСПОЛЬЗУЙ bundle'ы только для:
   • переиспользуемой логики между несколькими приложениями
   • публичных пакетов (на Packagist / GitHub)
   • плагинов с собственной конфигурацией и расширениями
   • интеграции сторонних библиотек с Symfony
```

---

## 10. Связанные темы (для углубления)

| Тема                                  | Ссылка                                                                           |
| ------------------------------------- | -------------------------------------------------------------------------------- |
| Переопределение частей bundle'а       | https://symfony.com/doc/current/bundles/override.html                            |
| Best Practices для reusable bundle    | https://symfony.com/doc/current/bundles/best_practices.html                      |
| Дружелюбная конфигурация bundle       | https://symfony.com/doc/current/bundles/configuration.html                       |
| Загрузка сервисов внутри bundle       | https://symfony.com/doc/current/bundles/extension.html                           |
| Конфигурирование других bundle'ов     | https://symfony.com/doc/current/bundles/prepend_extension.html                   |