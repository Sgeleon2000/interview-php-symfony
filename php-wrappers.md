# Протоколы и обёртки потоков в PHP (Wrappers)

> На основе [php.net/manual/ru/wrappers.php](https://www.php.net/manual/ru/wrappers.php). Примеры на PHP 8.x.

**Обёртка (wrapper)** — это код, который сообщает потоковым функциям PHP, как обращаться к определённому протоколу или способу хранения данных. Благодаря обёрткам одни и те же функции (`fopen`, `file_get_contents`, `copy`, `file_exists`) работают и с локальными файлами, и с HTTP-ресурсами, и с архивами, и с памятью.

## Содержание

- [Как работают обёртки](#как-работают-обёртки)
- [file:// — локальная файловая система](#file--локальная-файловая-система)
- [http:// и https:// — HTTP(S)](#http-и-https--https)
- [ftp:// и ftps:// — FTP(S)](#ftp-и-ftps--ftps)
- [php:// — потоки ввода-вывода](#php--потоки-ввода-вывода)
- [php://filter — фильтры](#phpfilter--фильтры)
- [compress.zlib:// и compress.bzip2:// — сжатие](#compresszlib-и-compressbzip2--сжатие)
- [data:// — встроенные данные](#data--встроенные-данные)
- [glob:// — поиск по шаблону](#glob--поиск-по-шаблону)
- [phar:// — PHP-архивы](#phar--php-архивы)
- [Прочие обёртки](#прочие-обёртки)
- [Безопасность](#безопасность)
- [Свои обёртки](#свои-обёртки)

---

## Как работают обёртки

Префикс перед путём (`schema://`) выбирает обёртку. Без префикса используется `file://`.

```php
<?php
file_get_contents('/etc/hosts');               // file:// (по умолчанию)
file_get_contents('file:///etc/hosts');        // явно file://
file_get_contents('https://example.com');      // http(s)://
file_get_contents('php://input');              // php://
file_get_contents('phar://app.phar/index.php'); // phar://
```

Список зарегистрированных обёрток:

```php
<?php
print_r(stream_get_wrappers());
// Array ( [0] => php [1] => file [2] => glob [3] => data
//         [4] => http [5] => https [6] => ftp [7] => ftps
//         [8] => compress.zlib [9] => phar ... )
```

---

## file:// — локальная файловая система

Обёртка по умолчанию. Работает со всеми файловыми функциями.

```php
<?php
$data = file_get_contents('/path/to/file.txt');
$data = file_get_contents('file:///path/to/file.txt');   // эквивалент

// Относительные и абсолютные пути
file_put_contents('output.txt', 'data');
copy('a.txt', 'b.txt');
$exists = file_exists('config.ini');
```

---

## http:// и https:// — HTTP(S)

Чтение ресурсов по HTTP. Только **чтение** (запись не поддерживается). По умолчанию GET; для других методов и заголовков — stream context (см. `php-stream-contexts.md`).

```php
<?php
// Простой GET
$html = file_get_contents('https://example.com');

// Заголовки ответа — в автоматической переменной
print_r($http_response_header);

// POST/заголовки — через контекст
$ctx = stream_context_create(['http' => [
    'method' => 'POST',
    'header' => "Content-Type: application/json\r\n",
    'content' => '{"x":1}',
]]);
$resp = file_get_contents('https://api.example.com', false, $ctx);
```

> Требует включённой директивы `allow_url_fopen`. Поток открывается только для чтения; нельзя «дозаписать» в URL.

---

## ftp:// и ftps:// — FTP(S)

Чтение и запись файлов по FTP. Поддерживает дозапись/возобновление при настройке через контекст.

```php
<?php
// Чтение
$data = file_get_contents('ftp://user:pass@ftp.example.com/file.txt');

// Запись (создаёт/перезаписывает файл на сервере)
file_put_contents('ftp://user:pass@ftp.example.com/upload.txt', $content);
```

---

## php:// — потоки ввода-вывода

Доступ к стандартным потокам и внутренним механизмам PHP. Очень важная обёртка.

| Поток | Назначение |
|-------|-----------|
| `php://input` | сырое тело запроса (для JSON/PUT — вместо `$_POST`) |
| `php://output` | запись в выходной буфер (как `echo`) |
| `php://stdin` / `php://stdout` / `php://stderr` | стандартные потоки (CLI) |
| `php://memory` | поток в памяти |
| `php://temp` | в памяти, с переключением на диск при превышении лимита |
| `php://fd` | доступ к файловому дескриптору (CLI) |
| `php://filter` | применение фильтров к другому потоку |

```php
<?php
// Прочитать тело запроса (JSON API)
$body = file_get_contents('php://input');
$data = json_decode($body, true);

// Поток в памяти — удобно для тестов и временных данных
$fp = fopen('php://memory', 'r+');
fwrite($fp, 'временные данные');
rewind($fp);
echo stream_get_contents($fp);
fclose($fp);

// php://temp с лимитом 5 МБ (потом уйдёт на диск)
$fp = fopen('php://temp/maxmemory:5242880', 'r+');

// Вывод в stderr (CLI) — например, для логов
fwrite(fopen('php://stderr', 'w'), "ошибка\n");
```

> `php://memory` держит всё в ОЗУ; `php://temp` безопаснее для больших объёмов — автоматически сбрасывает на диск.

---

## php://filter — фильтры

Позволяет применять фильтры (преобразования) к потоку на лету: смена кодировки, base64, и т.п.

```php
<?php
// Прочитать файл сразу в верхнем регистре
$data = file_get_contents('php://filter/read=string.toupper/resource=file.txt');

// Цепочка фильтров через |
$data = file_get_contents('php://filter/read=string.toupper|string.rot13/resource=file.txt');

// Конвертация кодировки при чтении
$data = file_get_contents(
    'php://filter/read=convert.iconv.UTF-8/CP1251/resource=file.txt'
);
```

---

## compress.zlib:// и compress.bzip2:// — сжатие

Прозрачное чтение/запись сжатых файлов (gzip, bzip2).

```php
<?php
// Прозрачно читать gzip-файл
$data = file_get_contents('compress.zlib://archive.gz');

// Записать со сжатием
file_put_contents('compress.zlib://out.gz', $content);

// Можно комбинировать с другими обёртками
$fp = fopen('compress.zlib://php://temp', 'r+');
```

> Историческое имя `zlib://` тоже встречается, но рекомендуется `compress.zlib://`.

---

## data:// — встроенные данные

Реализация data-URI (RFC 2397) — данные прямо в «адресе».

```php
<?php
echo file_get_contents('data://text/plain,Hello%20World');   // Hello World

// base64
echo file_get_contents('data://text/plain;base64,SGVsbG8=');  // Hello
```

---

## glob:// — поиск по шаблону

Перебор путей, подходящих под шаблон (через `opendir`/`readdir`).

```php
<?php
$dir = opendir('glob://*.php');
while (($file = readdir($dir)) !== false) {
    echo "$file\n";
}
closedir($dir);

// Чаще используют функцию glob() напрямую:
foreach (glob('/var/log/*.log') as $logFile) {
    echo $logFile;
}
```

---

## phar:// — PHP-архивы

Доступ к файлам внутри **Phar** (PHP Archive) — упакованного приложения/библиотеки в одном файле.

```php
<?php
// Подключить файл из архива
require 'phar://application.phar/src/bootstrap.php';

// Прочитать ресурс из архива
$config = file_get_contents('phar://app.phar/config/app.ini');
```

> Phar похож на JAR в Java — распространение PHP-приложений одним файлом (например, `composer.phar`, `phpunit.phar`).

---

## Прочие обёртки

Доступны при установке соответствующих расширений:

| Обёртка | Назначение | Требует |
|---------|-----------|---------|
| `ssh2://` | файлы/команды по SSH2 | ext-ssh2 |
| `rar://` | файлы внутри RAR-архивов | ext-rar |
| `ogg://` | аудиопотоки OGG | ext-oggvorbis |
| `expect://` | взаимодействие с процессами (PTY) | ext-expect |

```php
<?php
// Примеры (требуют расширений):
$data = file_get_contents('ssh2.sftp://user:pass@host:22/path/file');
$data = file_get_contents('rar://archive.rar#inside.txt');
```

---

## Безопасность

Обёртки URL — частый вектор атак (LFI/RFI, SSRF). Ключевые директивы и правила:

```php
<?php
// php.ini:
// allow_url_fopen  = On/Off  — разрешить открывать URL функциями файлов
// allow_url_include = Off     — ЗАПРЕТИТЬ include/require по URL (всегда Off!)
```

- ⚠️ **Никогда не подставляйте пользовательский ввод в путь** `fopen`/`include` — атакующий может передать `php://filter`, `data://` или `http://` и прочитать/выполнить произвольное.
- `allow_url_include` должна быть **выключена** (это защита от Remote File Inclusion).
- `data://` и `php://filter` использовались в реальных эксплойтах для обхода фильтров — валидируйте схему пути.

```php
<?php
// ❌ ОПАСНО
include $_GET['page'] . '.php';      // ?page=php://filter/... или http://evil

// ✅ Безопасно — белый список
$allowed = ['home', 'about', 'contact'];
$page = in_array($_GET['page'] ?? '', $allowed, true) ? $_GET['page'] : 'home';
include __DIR__ . "/pages/$page.php";
```

---

## Свои обёртки

Можно зарегистрировать собственную обёртку, реализовав класс по контракту `streamWrapper` и зарегистрировав его через `stream_wrapper_register()`.

```php
<?php
class MyWrapper
{
    public $context;
    public function stream_open(string $path, string $mode, int $options, ?string &$openedPath): bool { /* ... */ return true; }
    public function stream_read(int $count): string { /* ... */ return ''; }
    public function stream_write(string $data): int { /* ... */ return strlen($data); }
    public function stream_eof(): bool { /* ... */ return true; }
    public function stream_stat(): array|false { return false; }
    // + stream_close, stream_seek, url_stat и др.
}

stream_wrapper_register('myproto', MyWrapper::class);
$data = file_get_contents('myproto://something');   // вызовет вашу обёртку
```

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое обёртка потока? | Код, описывающий протокол доступа для файловых функций (`schema://`). |
| Обёртка по умолчанию? | `file://`. |
| Как прочитать тело JSON-запроса? | `file_get_contents('php://input')`. |
| `php://memory` vs `php://temp`? | memory — всё в ОЗУ; temp — на диск после лимита. |
| Можно ли писать в `http://`? | Нет — только чтение. |
| Что делает `php://filter`? | Применяет преобразования к потоку (кодировка, base64, регистр). |
| Как читать gzip прозрачно? | `compress.zlib://file.gz`. |
| Чем опасны обёртки в include? | LFI/RFI/SSRF — `php://filter`, `data://`, `http://` в пользовательском вводе. |
| Какая директива защищает от RFI? | `allow_url_include = Off`. |
| Что такое Phar? | PHP-архив: приложение/библиотека в одном файле (как JAR). |
| Как сделать свою обёртку? | Класс по контракту + `stream_wrapper_register()`. |

---

*Источник: [php.net/manual/ru/wrappers.php](https://www.php.net/manual/ru/wrappers.php). Примеры на PHP 8.x. См. также `php-stream-contexts.md`, `php-reserved-variables.md`.*
