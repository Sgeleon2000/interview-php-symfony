# Предопределённые переменные в PHP (Superglobals)

> На основе [php.net/manual/ru/reserved.variables.php](https://www.php.net/manual/ru/reserved.variables.php). Примеры на PHP 8.x.

PHP предоставляет набор **предопределённых переменных** — они доступны в любом скрипте. Большинство из них — **суперглобальные** (superglobals): видны в любой области видимости без объявления `global`.

## Содержание

- [Что такое суперглобалы](#что-такое-суперглобалы)
- [$GLOBALS](#globals)
- [$_SERVER](#_server)
- [$_GET](#_get)
- [$_POST](#_post)
- [$_REQUEST](#_request)
- [$_FILES](#_files)
- [$_COOKIE](#_cookie)
- [$_SESSION](#_session)
- [$_ENV](#_env)
- [Переменные CLI: $argc и $argv](#переменные-cli-argc-и-argv)
- [Прочие: $http_response_header](#прочие-http_response_header)
- [Безопасность](#безопасность)

---

## Что такое суперглобалы

Суперглобальные переменные — это встроенные массивы, доступные **везде** без `global`:

```php
<?php
function handler() {
    echo $_GET['page'];      // работает без global $_GET
    echo $_SERVER['HTTP_HOST'];
}
```

Полный список суперглобалов: `$GLOBALS`, `$_SERVER`, `$_GET`, `$_POST`, `$_FILES`, `$_REQUEST`, `$_SESSION`, `$_ENV`, `$_COOKIE`.

> Суперглобалы нельзя использовать как имена переменных-параметров функций; их нельзя переопределить.

---

## $GLOBALS

Ассоциативный массив всех переменных **глобальной области**. Ключ — имя переменной.

```php
<?php
$x = 10;

function show() {
    echo $GLOBALS['x'];      // доступ к глобальной $x изнутри функции
}
show();                      // 10

// Эквивалент использования global:
function inc() {
    $GLOBALS['x']++;         // = global $x; $x++;
}
```

> ⚠️ В PHP 8.1 запись в `$GLOBALS` целиком ограничена (нельзя `$GLOBALS = [...]`), но изменение отдельных элементов работает. Использование `$GLOBALS` — признак плохой архитектуры; предпочитайте передачу через аргументы/DI.

---

## $_SERVER

Информация о сервере и окружении выполнения: заголовки, пути, адреса. Содержимое зависит от веб-сервера.

```php
<?php
$_SERVER['REQUEST_METHOD'];   // 'GET', 'POST', ...
$_SERVER['REQUEST_URI'];      // '/users?page=2'
$_SERVER['HTTP_HOST'];        // 'example.com'
$_SERVER['REMOTE_ADDR'];      // IP клиента
$_SERVER['HTTP_USER_AGENT'];  // строка User-Agent
$_SERVER['SCRIPT_NAME'];      // '/index.php'
$_SERVER['SERVER_NAME'];      // имя сервера
$_SERVER['QUERY_STRING'];     // 'page=2'
$_SERVER['DOCUMENT_ROOT'];    // корень сайта на диске
$_SERVER['HTTPS'] ?? null;    // непусто, если HTTPS
```

> Все заголовки запроса доступны как `HTTP_*` (например, `Accept-Language` → `$_SERVER['HTTP_ACCEPT_LANGUAGE']`).

---

## $_GET

Параметры из строки запроса URL (`?key=value`). Используется для GET-запросов.

```php
<?php
// URL: /search?q=php&page=2
$query = $_GET['q'] ?? '';        // 'php'
$page  = (int) ($_GET['page'] ?? 1);  // 2

// Массивы в URL: ?tags[]=a&tags[]=b
$tags = $_GET['tags'] ?? [];      // ['a', 'b']
```

> Данные всегда строки (или массивы строк). Приводите типы и валидируйте.

---

## $_POST

Данные, переданные методом HTTP POST (обычно из форм с `Content-Type: application/x-www-form-urlencoded` или `multipart/form-data`).

```php
<?php
// <form method="post"><input name="email"></form>
$email = $_POST['email'] ?? '';

// Вложенные имена: <input name="user[name]">
$name = $_POST['user']['name'] ?? '';
```

> JSON-тело запроса (`application/json`) в `$_POST` **не** попадает — его читают через `file_get_contents('php://input')` и `json_decode()`.

```php
<?php
$data = json_decode(file_get_contents('php://input'), true);
```

---

## $_REQUEST

Объединение `$_GET`, `$_POST` и `$_COOKIE` (порядок и состав зависят от `request_order` в php.ini).

```php
<?php
$value = $_REQUEST['key'] ?? null;   // из GET, POST или COOKIE
```

> ⚠️ Не рекомендуется в безопасном коде: неясно, откуда пришли данные, и cookie может «перебить» параметр запроса. Лучше явно использовать `$_GET`/`$_POST`.

---

## $_FILES

Информация о файлах, загруженных через HTTP POST (форма с `enctype="multipart/form-data"`).

```php
<?php
// <input type="file" name="avatar">
$file = $_FILES['avatar'];

$file['name'];      // оригинальное имя файла
$file['type'];      // MIME-тип (от клиента — не доверять!)
$file['tmp_name'];  // временный путь на сервере
$file['error'];     // код ошибки (UPLOAD_ERR_OK == 0)
$file['size'];      // размер в байтах

if ($file['error'] === UPLOAD_ERR_OK) {
    move_uploaded_file($file['tmp_name'], '/uploads/' . basename($file['name']));
}
```

> MIME-тип из `$_FILES['...']['type']` задаёт клиент — проверяйте реальный тип через `finfo`. Всегда используйте `move_uploaded_file()`, а не `copy()`.

---

## $_COOKIE

HTTP cookies, переданные браузером.

```php
<?php
$theme = $_COOKIE['theme'] ?? 'light';

// Установка cookie (до любого вывода!) — функцией setcookie:
setcookie('theme', 'dark', [
    'expires'  => time() + 86400,
    'httponly' => true,    // недоступна из JS (защита от XSS)
    'secure'   => true,    // только по HTTPS
    'samesite' => 'Strict',
]);
// Новое значение появится в $_COOKIE при СЛЕДУЮЩЕМ запросе
```

---

## $_SESSION

Переменные сессии — сохраняются между запросами одного пользователя. Требует `session_start()`.

```php
<?php
session_start();                      // обязательно перед использованием

$_SESSION['user_id'] = 42;            // записать
$userId = $_SESSION['user_id'] ?? null;  // прочитать

unset($_SESSION['user_id']);          // удалить одну переменную
session_destroy();                    // уничтожить всю сессию
```

> Данные сессии хранятся на сервере, клиенту отдаётся только идентификатор (обычно в cookie `PHPSESSID`).

---

## $_ENV

Переменные окружения, переданные процессу PHP. Наполнение зависит от настройки `variables_order` в php.ini.

```php
<?php
$path = $_ENV['PATH'] ?? '';
$dbHost = $_ENV['DB_HOST'] ?? 'localhost';

// Более надёжно читать окружение через getenv():
$dbHost = getenv('DB_HOST') ?: 'localhost';
```

> В реальных проектах переменные окружения чаще читают через библиотеки (`symfony/dotenv`, `vlucas/phpdotenv`) и `getenv()`, т.к. `$_ENV` может быть пустым при определённой конфигурации.

---

## Переменные CLI: $argc и $argv

Доступны при запуске скрипта из командной строки (CLI).

```php
<?php
// php script.php arg1 arg2
echo $argc;          // 3 — количество аргументов (включая имя скрипта)
print_r($argv);
/* Array (
    [0] => script.php
    [1] => arg1
    [2] => arg2
) */

$command = $argv[1] ?? 'help';
```

> `$argv[0]` всегда содержит имя самого скрипта. Эти переменные **не** являются суперглобальными — доступны в глобальной области (внутри функции нужен `global` или `$_SERVER['argv']`).

---

## Прочие: $http_response_header

Заполняется автоматически при HTTP-запросах через обёртки потоков (`file_get_contents`, `fopen`). Содержит заголовки ответа.

```php
<?php
$body = file_get_contents('https://example.com');
print_r($http_response_header);   // массив строк заголовков ответа
/* [0] => HTTP/1.1 200 OK
   [1] => Content-Type: text/html
   ... */
```

> Переменная `$php_errormsg` (последнее сообщение об ошибке) — **устарела** и удалена в PHP 8.0. Вместо неё используйте `error_get_last()`.

---

## Безопасность

Данные из запроса — это **недоверенный ввод**. Ключевые правила:

```php
<?php
// 1. Всегда валидируйте и фильтруйте
$id = filter_input(INPUT_GET, 'id', FILTER_VALIDATE_INT);
$email = filter_input(INPUT_POST, 'email', FILTER_VALIDATE_EMAIL);

// 2. Экранируйте при выводе (защита от XSS)
echo htmlspecialchars($_GET['name'] ?? '', ENT_QUOTES);

// 3. Используйте подготовленные выражения для БД (защита от SQL-инъекций)
$stmt = $pdo->prepare('SELECT * FROM users WHERE id = ?');
$stmt->execute([$_GET['id']]);
```

Важные нюансы безопасности из документации:
- **`$_SERVER['PHP_SELF']`** может содержать ввод пользователя — для путей используйте `$_SERVER['SCRIPT_NAME']`.
- **`$_SERVER`** в целом нельзя слепо доверять (часть значений — заголовки от клиента).
- **`HTTP_X_FORWARDED_FOR`** и подобные заголовки легко подделать — для проверки реального IP используйте `$_SERVER['REMOTE_ADDR']`.
- **Точки и пробелы** в именах GET/POST-полей автоматически заменяются на `_` (поле `user.name` → `$_POST['user_name']`).

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое суперглобалы? | Встроенные массивы, доступные везде без `global`. |
| Перечисли суперглобалы. | `$GLOBALS`, `$_SERVER`, `$_GET`, `$_POST`, `$_FILES`, `$_REQUEST`, `$_SESSION`, `$_ENV`, `$_COOKIE`. |
| Откуда брать тело JSON-запроса? | `file_get_contents('php://input')` — не из `$_POST`. |
| Что в `$_REQUEST` и почему его избегают? | GET+POST+COOKIE; неясен источник данных, cookie может перекрыть параметр. |
| Как безопасно загрузить файл? | Проверить `error`, реальный MIME через `finfo`, `move_uploaded_file()`. |
| Чем `$_SESSION` отличается от `$_COOKIE`? | Сессия хранится на сервере (клиенту — только ID); cookie — у клиента. |
| Можно ли доверять `$_SERVER`? | Нет — многие значения приходят от клиента (заголовки, PHP_SELF). |
| Как получить реальный IP? | `$_SERVER['REMOTE_ADDR']` (X-Forwarded-For подделывается). |
| Что такое `$argv`/`$argc`? | Аргументы командной строки в CLI; `$argv[0]` — имя скрипта. |
| Заменитель `$php_errormsg`? | `error_get_last()` (сама переменная удалена в 8.0). |

---

*Источник: [php.net/manual/ru/reserved.variables.php](https://www.php.net/manual/ru/reserved.variables.php). Примеры на PHP 8.x. См. также `php-variables.md`, `php-errors.md`.*
