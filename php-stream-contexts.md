# Контексты потоков в PHP (Stream Contexts)

> На основе [php.net/manual/ru/context.php](https://www.php.net/manual/ru/context.php). Примеры на PHP 8.x.

**Контекст потока (stream context)** — это набор параметров и опций, которые настраивают поведение файловых функций и обёрток потоков (stream wrappers). Контекст позволяет, например, задать HTTP-метод и заголовки для `file_get_contents()`, настроить проверку SSL-сертификата или таймаут сокета — без перехода на cURL.

## Содержание

- [Что такое поток и обёртка](#что-такое-поток-и-обёртка)
- [Создание контекста](#создание-контекста)
- [Применение контекста](#применение-контекста)
- [Категории опций](#категории-опций)
- [HTTP-контекст](#http-контекст)
- [SSL-контекст](#ssl-контекст)
- [Socket-контекст](#socket-контекст)
- [FTP, Zip, Zlib, Phar](#ftp-zip-zlib-phar)
- [Параметры контекста (params)](#параметры-контекста-params)
- [Изменение существующего контекста](#изменение-существующего-контекста)

---

## Что такое поток и обёртка

**Поток (stream)** в PHP — это абстракция над любым линейным источником/приёмником данных: файл, сетевое соединение, сжатый архив, память. **Обёртка (wrapper)** описывает протокол доступа: `file://`, `http://`, `https://`, `ftp://`, `php://`, `zip://`, `compress.zlib://`, `phar://`.

```php
<?php
// Все эти функции работают с потоками через обёртки:
file_get_contents('https://example.com');   // обёртка http
file_get_contents('php://input');           // обёртка php
fopen('compress.zlib://file.gz', 'r');       // обёртка zlib
```

Контекст — это «настройки» для конкретной обёртки.

---

## Создание контекста

Контекст создаётся функцией `stream_context_create()`. Структура: `['обёртка' => ['опция' => значение]]`.

```php
<?php
$context = stream_context_create([
    'http' => [
        'method'  => 'POST',
        'header'  => "Content-Type: application/json\r\n",
        'content' => json_encode(['key' => 'value']),
        'timeout' => 10,
    ],
]);
```

---

## Применение контекста

Контекст передаётся последним аргументом в файловые функции, поддерживающие потоки.

```php
<?php
// file_get_contents с контекстом
$response = file_get_contents('https://api.example.com/users', false, $context);

// fopen с контекстом
$handle = fopen('https://example.com', 'r', false, $context);

// Также: file_put_contents, fopen, opendir, copy, stream_socket_client и др.
```

> Многие воспринимают `file_get_contents()` как простую функцию для файлов, но с HTTP-контекстом она становится полноценным мини-HTTP-клиентом.

---

## Категории опций

Опции группируются по обёрткам:

| Обёртка | Раздел мануала | Что настраивает |
|---------|----------------|-----------------|
| `socket` | `context.socket.php` | низкоуровневые опции сокета (привязка, backlog) |
| `http` | `context.http.php` | метод, заголовки, тело, таймаут, редиректы |
| `ftp` | `context.ftp.php` | параметры FTP (overwrite, resume_pos) |
| `ssl` | `context.ssl.php` | проверка сертификатов, SNI, шифры |
| `phar` | `context.phar.php` | сжатие и метаданные Phar-архивов |
| `zip` | `context.zip.php` | пароль для ZIP-архивов |
| `zlib` | `context.zlib.php` | уровень сжатия zlib |
| (общие) | `context.params.php` | `notification` callback и др. |

---

## HTTP-контекст

Самая используемая категория. Основные опции `http`:

```php
<?php
$context = stream_context_create([
    'http' => [
        'method'           => 'POST',          // GET, POST, PUT, DELETE...
        'header'           => implode("\r\n", [
            'Content-Type: application/json',
            'Authorization: Bearer TOKEN',
            'Accept: application/json',
        ]),
        'content'          => json_encode(['name' => 'Анна']),
        'timeout'          => 15,              // таймаут в секундах
        'follow_location'  => 1,               // следовать редиректам (1/0)
        'max_redirects'    => 5,
        'protocol_version' => 1.1,             // важно для keep-alive
        'ignore_errors'    => true,            // читать тело даже при 4xx/5xx
        'user_agent'       => 'MyApp/1.0',
    ],
]);

$body = file_get_contents('https://api.example.com/users', false, $context);

// Заголовки ответа доступны в $http_response_header
print_r($http_response_header);
```

> `ignore_errors => true` особенно важно: без него `file_get_contents` при ответе 404/500 вернёт `false` и не даст прочитать тело ошибки.

---

## SSL-контекст

Настройки TLS/SSL для `https://`, `ftps://` и защищённых сокетов.

```php
<?php
$context = stream_context_create([
    'ssl' => [
        'verify_peer'       => true,    // проверять сертификат сервера (НЕ отключать в проде!)
        'verify_peer_name'  => true,    // проверять соответствие имени хоста
        'allow_self_signed' => false,
        'cafile'            => '/path/to/ca-bundle.crt',  // свой CA
        'ciphers'           => 'HIGH:!aNULL',
        'SNI_enabled'       => true,
        'peer_name'         => 'example.com',
    ],
]);
```

> ⚠️ Отключение `verify_peer`/`verify_peer_name` (`false`) делает соединение уязвимым к MITM-атакам. Это частая ошибка «чтобы заработало» — в продакшене недопустимо.

---

## Socket-контекст

Низкоуровневые опции для сокет-соединений (`tcp://`, `udp://`).

```php
<?php
$context = stream_context_create([
    'socket' => [
        'bindto'  => '0:0',       // привязать к локальному IP:порт
        'backlog' => 128,         // размер очереди соединений (для серверов)
        'tcp_nodelay' => true,    // отключить алгоритм Нейгла
    ],
]);

$socket = stream_socket_client(
    'tcp://example.com:80', $errno, $errstr, 30,
    STREAM_CLIENT_CONNECT, $context
);
```

---

## FTP, Zip, Zlib, Phar

```php
<?php
// FTP — параметры передачи
$ftp = stream_context_create(['ftp' => ['overwrite' => true]]);

// ZIP — пароль для защищённого архива
$zip = stream_context_create(['zip' => ['password' => 'secret']]);
$content = file_get_contents('zip://archive.zip#secret.txt', false, $zip);

// Zlib — уровень сжатия (0-9)
$zlib = stream_context_create(['zlib' => ['level' => 6]]);

// Phar — сжатие архива
$phar = stream_context_create(['phar' => ['compress' => Phar::GZ]]);
```

---

## Параметры контекста (params)

В отличие от опций (зависят от обёртки), **параметры** — общие. Главный из них — `notification`: callback для отслеживания событий потока (прогресс загрузки, редиректы).

```php
<?php
$context = stream_context_create();

stream_context_set_params($context, [
    'notification' => function (
        int $code, int $severity, ?string $message,
        int $messageCode, int $bytesTransferred, int $bytesMax
    ): void {
        switch ($code) {
            case STREAM_NOTIFY_PROGRESS:
                echo "Загружено: $bytesTransferred / $bytesMax\n";
                break;
            case STREAM_NOTIFY_REDIRECTED:
                echo "Редирект на: $message\n";
                break;
            case STREAM_NOTIFY_CONNECT:
                echo "Соединение установлено\n";
                break;
        }
    },
]);

file_get_contents('https://example.com/big-file', false, $context);
```

---

## Изменение существующего контекста

Опции и параметры можно менять после создания:

```php
<?php
$context = stream_context_create();

// Установить опцию обёртки
stream_context_set_option($context, 'http', 'method', 'POST');
stream_context_set_option($context, 'http', 'timeout', 30);

// Установить общий параметр
stream_context_set_params($context, ['notification' => $callback]);

// Прочитать текущие опции
$options = stream_context_get_options($context);

// Контекст по умолчанию для всех операций
$default = stream_context_get_default();
stream_context_set_default(['http' => ['user_agent' => 'MyApp']]);
```

---

## Практический пример: JSON API-запрос без cURL

```php
<?php
function apiPost(string $url, array $data, string $token): array
{
    $context = stream_context_create([
        'http' => [
            'method'        => 'POST',
            'header'        => implode("\r\n", [
                'Content-Type: application/json',
                "Authorization: Bearer $token",
            ]),
            'content'       => json_encode($data),
            'timeout'       => 10,
            'ignore_errors' => true,    // чтобы прочитать тело при ошибке
        ],
    ]);

    $response = file_get_contents($url, false, $context);
    if ($response === false) {
        throw new RuntimeException('Запрос не выполнен');
    }
    return json_decode($response, true) ?? [];
}
```

> Для сложных сценариев (параллельные запросы, тонкий контроль) лучше cURL или HTTP-клиент (Guzzle, Symfony HttpClient — PSR-18). Контексты хороши для простых случаев без зависимостей.

---

## Шпаргалка для собеседования

| Вопрос | Ответ |
|--------|-------|
| Что такое stream context? | Набор опций/параметров, настраивающих поведение обёртки потока. |
| Как создать контекст? | `stream_context_create(['http' => [...]])`. |
| Куда передаётся контекст? | Последним аргументом в `file_get_contents`, `fopen`, `copy` и т.п. |
| Можно ли делать POST через `file_get_contents`? | Да — через HTTP-контекст (`method`, `header`, `content`). |
| Зачем `ignore_errors => true`? | Прочитать тело ответа при кодах 4xx/5xx (иначе вернётся `false`). |
| Чем опасно `verify_peer => false`? | Отключает проверку SSL → уязвимость к MITM. |
| Опции vs параметры контекста? | Опции зависят от обёртки; параметры (`notification`) — общие. |
| Что делает `notification`? | Callback на события потока: прогресс, редирект, соединение. |
| Когда лучше cURL/Guzzle? | Для сложных запросов, параллелизма, тонкого контроля. |

---

*Источник: [php.net/manual/ru/context.php](https://www.php.net/manual/ru/context.php). Примеры на PHP 8.x.*
