# DDD-архитектура проекта (Symfony 8 / PHP 8.4)

Документ объясняет, как в этом проекте реализован **Domain-Driven Design** (DDD): какие
есть bounded contexts, как устроены слои, как они связаны, и какие паттерны
применяются. Все примеры — из реального кода в `app/src/`.

---

## 1. Общая картина

Проект построен как **модульный монолит** с разбиением на *bounded contexts*
(ограниченные контексты), каждый из которых внутри разделён на 4 классических
слоя DDD: **Domain → Application → Infrastructure → UI**.

```
app/src/
├── IdentityAccess/     # регистрация, вход, токены, верификация e-mail, сброс пароля
├── Interview/          # сессии интервью (основная сущность продукта)
├── DocumentLibrary/    # загрузка документов, извлечение текста
├── Billing/            # подписки и платежи (Stripe)
├── Usage/              # учёт потребления (usage records)
├── Notification/       # отправка писем (порт Mailer + адаптеры)
├── Admin/              # административные запросы (список юзеров/платежей)
└── Shared/             # общие VO, исключения, порты, инфраструктурные хелперы
```

Каждый контекст — это отдельная «вертикаль» бизнес-логики со своими моделями,
правилами и API. Контексты общаются только через **публичные контракты**
(value objects, репозитории, application-команды), а не через прямой доступ к
внутренностям друг друга.

### Используемый технологический стек
- **Symfony 8.0**, **PHP 8.4** (`readonly`-классы, enum'ы, named arguments).
- **Doctrine ORM 3** — персистентность (атрибутный маппинг прямо на агрегатах).
- **Symfony Messenger** — шина команд/запросов (сейчас в `sync`-режиме).
- **Symfony Security** — аутентификация (форма + Bearer-токены устройств).
- **Symfony UX (Live Components)** — реактивный фронтенд.

---

## 2. Структура слоёв внутри одного контекста

Каждый контекст повторяет одну и ту же раскладку каталогов (пример —
`IdentityAccess/`):

```
IdentityAccess/
├── Domain/                 # ЯДРО. Не зависит ни от чего внешнего.
│   ├── Model/              # агрегаты и сущности (User, DeviceToken, ...)
│   ├── ValueObject/        # UserId, HashedPassword, Role, ...
│   ├── Repository/         # ИНТЕРФЕЙСЫ репозиториев (порты)
│   ├── Service/            # доменные сервисы
│   ├── Event/              # доменные события
│   └── Exception/          # доменные исключения (EmailAlreadyTakenException)
│
├── Application/            # СЦЕНАРИИ использования (оркестрация).
│   ├── Command/            # команды-мутации + их хендлеры (CQRS write)
│   ├── Query/              # запросы-чтения + их хендлеры (CQRS read)
│   ├── DTO/                # объекты для передачи данных наружу (View-модели)
│   ├── Service/            # ИНТЕРФЕЙСЫ прикладных портов (PasswordHasher)
│   └── EventSubscriber/    # реакция на события
│
├── Infrastructure/         # АДАПТЕРЫ к внешнему миру.
│   ├── Persistence/Doctrine/Repository/   # реализации репозиториев
│   ├── Hashing/                            # Argon2idPasswordHasher
│   └── Security/                           # мосты к Symfony Security
│
└── UI/                     # ТОЧКИ ВХОДА.
    ├── Http/Controller/    # REST + web-контроллеры
    ├── LiveComponent/      # Symfony UX компоненты
    ├── Cli/                # консольные команды
    └── Form/               # формы
```

### Правило зависимостей (Dependency Rule)

Зависимости направлены **только внутрь**, к домену:

```
UI ──▶ Application ──▶ Domain ◀── Infrastructure
                          ▲
                          └──── Infrastructure реализует интерфейсы домена
```

- **Domain** не знает ни про Doctrine, ни про Symfony, ни про HTTP. Это чистый PHP
  с бизнес-правилами.
- **Application** знает про Domain, оркестрирует сценарии, но не знает про
  конкретную БД или фреймворк — работает через интерфейсы (порты).
- **Infrastructure** реализует интерфейсы из Domain/Application (паттерн
  **Ports & Adapters / Hexagonal**).
- **UI** — тонкий слой: принимает запрос, вызывает Application, отдаёт ответ.

---

## 3. Domain Layer — ядро

### 3.1. Value Objects (объекты-значения)

Неизменяемые (`readonly`), сравниваются по значению, **сами себя валидируют** в
конструкторе. Это делает невозможным существование «невалидного» значения.

**`Email`** (общий, в `Shared/Domain/ValueObject/Email.php`):

```php
final readonly class Email
{
    public string $value;

    public function __construct(string $value)
    {
        $value = strtolower(trim($value));
        if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
            throw new InvalidEmailException($value);   // невалидный e-mail не родится
        }
        $this->value = $value;
    }

    public function equals(self $other): bool { return $this->value === $other->value; }
    public function __toString(): string { return $this->value; }
}
```

**`UserId`** — типизированный идентификатор на базе UUID v7 с фабрикой:

```php
final readonly class UserId
{
    public string $value;

    public function __construct(string $value)
    {
        if (!Uuid::isValid($value)) {
            throw new \InvalidArgumentException(sprintf('Invalid UserId: "%s".', $value));
        }
        $this->value = $value;
    }

    public static function generate(): self { return new self(Uuid::v7()->toRfc4122()); }
    public function equals(self $other): bool { return $this->value === $other->value; }
}
```

Зачем типизированные ID, а не просто `string`: нельзя случайно передать `DocumentId`
туда, где ждут `UserId` — компилятор/IDE поймает ошибку. UUID v7 даёт сортируемость
по времени создания.

**Enum'ы как VO** — для конечных наборов состояний:

```php
enum Role: string { case User = 'user'; case Admin = 'admin'; }
```

Аналогично: `SessionStatus`, `SubscriptionStatus`, `Plan`, `DocumentType`,
`ExtractionStatus`, `UsageKind`.

### 3.2. Aggregates / Entities (агрегаты и сущности)

Агрегат — корневая сущность, охраняющая инварианты. Ключевые приёмы в проекте:

1. **Приватный конструктор + именованные фабрики** — объект нельзя создать в
   невалидном состоянии, только через осмысленный бизнес-метод
   (`User::register()`, `InterviewSession::create()`, `Document::upload()`).
2. **Поведение, а не сеттеры** — состояние меняется доменными методами с правилами
   (`changePassword()`, `verifyEmail()`, `attachDocument()`, `delete()`).
3. **Геттеры возвращают Value Objects**, хотя внутри хранятся примитивы (этого
   требует Doctrine-маппинг).

Пример — `User` (`IdentityAccess/Domain/Model/User.php`):

```php
#[ORM\Entity]
#[ORM\Table(name: 'users')]
class User
{
    #[ORM\Id, ORM\Column(type: 'uuid', unique: true)]
    private string $id;                              // примитив для Doctrine
    // ... email, passwordHash, role, createdAt, emailVerifiedAt

    private function __construct(/* VO-аргументы */) { /* ... */ }

    // Фабрика: единственный «правильный» способ создать пользователя
    public static function register(
        UserId $id, Email $email, HashedPassword $passwordHash, \DateTimeImmutable $createdAt,
    ): self {
        return new self($id, $email, $passwordHash, Role::User, $createdAt);
    }

    public function id(): UserId { return new UserId($this->id); }   // отдаёт VO
    public function email(): Email { return new Email($this->email); }

    // Поведение с инвариантом, а не голый сеттер
    public function changePassword(HashedPassword $newHash): void { $this->passwordHash = $newHash->hash; }
    public function verifyEmail(\DateTimeImmutable $at): void { $this->emailVerifiedAt = $at; }
    public function isEmailVerified(): bool { return $this->emailVerifiedAt !== null; }
}
```

Более богатый агрегат — `InterviewSession` (`Interview/Domain/Model/`). Здесь видно
защиту инвариантов и доменные операции:

```php
public function setTitle(string $title): void
{
    $title = trim($title);
    if ($title === '') { throw new \InvalidArgumentException('Title is required.'); }
    if (strlen($title) > 200) { throw new \InvalidArgumentException('Title is too long.'); }
    $this->title = $title;
}

public function attachDocument(string $documentId, \DateTimeImmutable $now): void
{
    if (!in_array($documentId, $this->documentIds, true)) {   // идемпотентность
        $this->documentIds[] = $documentId;
        $this->updatedAt = $now;
    }
}

public function delete(\DateTimeImmutable $at): void
{
    $this->deletedAt ??= $at;            // soft-delete, повторный вызов не перезапишет
}

public function clone(InterviewSessionId $newId, \DateTimeImmutable $now): self { /* копия */ }
```

Важная деталь: **время передаётся снаружи** (`\DateTimeImmutable $now`), а не берётся
через `new \DateTimeImmutable()` внутри агрегата. Это делает домен детерминированным
и тестируемым.

### 3.3. Repository — порты (интерфейсы)

В домене лежит только **интерфейс** репозитория. Это порт: домен говорит «мне нужно
уметь сохранять и искать User», но не знает как.

```php
// IdentityAccess/Domain/Repository/UserRepository.php
interface UserRepository
{
    public function save(User $user): void;
    public function findByEmail(Email $email): ?User;
    public function findById(UserId $id): ?User;
}
```

Методы оперируют **доменными типами** (`User`, `Email`, `UserId`) — никаких
`array` или `int id`.

### 3.4. Доменные исключения

Бизнес-ошибки выражены типами: `EmailAlreadyTakenException`,
`InvalidEmailException`, `NotFoundException`, `AccessDeniedException`. UI-слой ловит
их и переводит в HTTP-коды (см. §6).

---

## 4. Application Layer — сценарии (CQRS)

Приложение реализует **CQRS** (Command Query Responsibility Segregation):
- **Command** — изменяет состояние (write side), возвращает максимум id.
- **Query** — читает данные (read side), возвращает DTO/View.

Каждый use-case оформлен как пара **Command/Query (DTO) + Handler**, лежащая в
своей папке:

```
Application/Command/RegisterUser/
├── RegisterUserCommand.php    # неизменяемый DTO с входными данными
└── RegisterUserHandler.php    # логика сценария
```

### 4.1. Command (входной DTO)

Просто переносит данные, без поведения:

```php
final readonly class RegisterUserCommand
{
    public function __construct(
        public string $email,
        public string $plainPassword,
    ) {}
}
```

### 4.2. Handler (оркестратор сценария)

Handler — `readonly` сервис с `__invoke()`. Он **координирует** домен и порты, но сам
бизнес-правила держит минимальными — основная логика живёт в агрегатах.

```php
final readonly class RegisterUserHandler
{
    public function __construct(
        private UserRepository $users,                          // порт (интерфейс)
        private PasswordHasher $passwordHasher,                 // порт
        private EmailVerificationTokenRepository $verificationTokens,
        private Mailer $mailer,                                 // порт из Shared
    ) {}

    public function __invoke(RegisterUserCommand $command): UserId
    {
        $email = new Email($command->email);                   // VO валидирует ввод

        if ($this->users->findByEmail($email) !== null) {
            throw new EmailAlreadyTakenException($email);       // доменное исключение
        }
        if (strlen($command->plainPassword) < 8) {
            throw new \InvalidArgumentException('Password must be at least 8 characters.');
        }

        $now = new \DateTimeImmutable();
        $user = User::register(                                 // создание через фабрику
            id: UserId::generate(),
            email: $email,
            passwordHash: $this->passwordHasher->hash($command->plainPassword),
            createdAt: $now,
        );
        $this->users->save($user);

        // побочный эффект: токен верификации + письмо (через порты)
        $plainToken = bin2hex(random_bytes(32));
        $this->verificationTokens->save(EmailVerificationToken::issue($plainToken, $user->id(), $now));
        $this->mailer->send($email->value, 'email_verification', ['token' => $plainToken]);

        return $user->id();
    }
}
```

Обратите внимание: handler зависит **только от интерфейсов** (`UserRepository`,
`PasswordHasher`, `Mailer`) — он не знает про Doctrine, Argon2 или SMTP.

### 4.3. Query + DTO (read side)

Запрос отдаёт **View-DTO**, а не доменный агрегат — наружу не утекает внутренняя
модель, и читающий код не может случайно мутировать домен. Здесь же — проверка
прав доступа.

```php
final readonly class GetSessionHandler
{
    public function __construct(private InterviewSessionRepository $sessions) {}

    public function __invoke(GetSessionQuery $query): SessionView
    {
        $session = $this->sessions->findById($query->id);
        if ($session === null) {
            throw new NotFoundException('Interview session');
        }
        if (!$session->userId()->equals($query->requestingUserId)) {  // авторизация на уровне данных
            throw new AccessDeniedException();
        }
        return SessionView::fromAggregate($session);                   // маппинг в DTO
    }
}
```

`SessionView` — плоский неизменяемый DTO с фабрикой из агрегата и сериализацией:

```php
final readonly class SessionView
{
    public function __construct(public string $id, public string $title, /* ... */) {}

    public static function fromAggregate(InterviewSession $s): self { /* читает геттеры агрегата */ }
    public function toArray(): array { /* snake_case для JSON-ответа */ }
}
```

### 4.4. Application-сервисы как порты

Кроме репозиториев, прикладной слой объявляет **технические порты** — интерфейсы для
вещей, которые делает инфраструктура:
- `IdentityAccess/Application/Service/PasswordHasher` — хеширование паролей;
- `DocumentLibrary/Application/Service/FileStorage` — хранилище файлов;
- `DocumentLibrary/Application/Service/TextExtractor` — извлечение текста;
- `Shared/Application/Notification/Mailer` — отправка писем.

Пример порта `Mailer` (`Shared/Application/Notification/Mailer.php`):

```php
interface Mailer
{
    /** @param array<string,scalar> $params */
    public function send(string $to, string $template, array $params): void;
}
```

---

## 5. Infrastructure Layer — адаптеры

Здесь живут конкретные реализации портов. Они **зависят** от домена, а домен — нет.

### 5.1. Doctrine-репозитории

```php
final readonly class DoctrineUserRepository implements UserRepository
{
    public function __construct(private EntityManagerInterface $em) {}

    public function save(User $user): void
    {
        $this->em->persist($user);
        $this->em->flush();
    }

    public function findByEmail(Email $email): ?User
    {
        return $this->em->getRepository(User::class)->findOneBy(['email' => $email->value]);
    }

    public function findById(UserId $id): ?User
    {
        return $this->em->getRepository(User::class)->find($id->value);
    }
}
```

> **Замечание по чистоте архитектуры.** В этом проекте Doctrine-атрибуты маппинга
> (`#[ORM\Entity]`, `#[ORM\Column]`) проставлены **прямо на доменных агрегатах**
> (`User`, `InterviewSession`). Это прагматичный компромисс (меньше кода, чем
> отдельные XML/PHP-маппинги), но формально это «протекание» инфраструктурной
> детали в домен. Полностью «чистый» вариант — выносить маппинг в
> `Infrastructure/Persistence/Doctrine/Mapping/` (каталоги под это уже
> зарезервированы `.gitkeep`-файлами). Домен изолируют от этого тем, что снаружи он
> всегда работает через VO и не зависит от Doctrine в логике.

### 5.2. Другие адаптеры

- `Infrastructure/Hashing/Argon2idPasswordHasher` — реализует `PasswordHasher`.
- `DocumentLibrary/Infrastructure/Storage/LocalFilesystemStorage` — реализует
  `FileStorage` (пишет в `var/uploads`).
- `DocumentLibrary/Infrastructure/TextExtraction/PlainTextExtractor` — реализует
  `TextExtractor`.
- `Notification/Infrastructure/Email/LogMailer` и `SymfonyMailer` — два адаптера под
  один порт `Mailer` (можно переключать конфигом).
- `Billing/Infrastructure/Stripe/StripeSignatureVerifier` — проверка подписи
  вебхука.
- `IdentityAccess/Infrastructure/Security/*` — мост к Symfony Security.

### 5.3. Мост к фреймворку (Security)

Домен ничего не знает про `Symfony\...\UserInterface`. Адаптер `SecurityUser`
переводит доменную модель в то, что понимает Symfony Security:

```php
final readonly class SecurityUser implements UserInterface, PasswordAuthenticatedUserInterface
{
    public function __construct(
        public string $id, public string $email, public string $passwordHash,
        public string $role, public bool $emailVerified,
    ) {}

    public function getRoles(): array
    {
        $roles = ['ROLE_USER'];
        if ($this->role === 'admin')   { $roles[] = 'ROLE_ADMIN'; }
        if ($this->emailVerified)      { $roles[] = 'ROLE_EMAIL_VERIFIED'; }
        return $roles;
    }
    // getPassword / getUserIdentifier / eraseCredentials
}
```

---

## 6. UI Layer — точки входа

Контроллеры **тонкие**: распарсить запрос → собрать Command/Query → вызвать handler →
перевести результат/исключения в HTTP.

```php
final class RegisterController
{
    public function __construct(private readonly RegisterUserHandler $handler) {}

    #[Route(path: '/auth/sign-up', name: 'auth_sign_up', methods: ['POST'])]
    public function __invoke(Request $request): JsonResponse
    {
        $payload = json_decode($request->getContent(), true, flags: JSON_THROW_ON_ERROR);
        // ... извлечь email/password ...

        try {
            $userId = ($this->handler)(new RegisterUserCommand($email, $password));
        } catch (InvalidEmailException|\InvalidArgumentException $e) {
            return new JsonResponse(['error' => $e->getMessage()], Response::HTTP_BAD_REQUEST);
        } catch (EmailAlreadyTakenException $e) {
            return new JsonResponse(['error' => $e->getMessage()], Response::HTTP_CONFLICT);  // 409
        }

        return new JsonResponse(['id' => $userId->value, 'email' => $email], Response::HTTP_CREATED);
    }
}
```

**Маппинг доменных исключений на HTTP-коды** — единственная «бизнес-логика» в UI:
- `InvalidEmailException` / `\InvalidArgumentException` → `400 Bad Request`
- `EmailAlreadyTakenException` → `409 Conflict`
- `NotFoundException` → `404`
- `AccessDeniedException` → `403`

### Получение текущего пользователя

Хелпер `Shared/Infrastructure/Http/RequestUserId` достаёт `UserId` из контекста
Security, чтобы контроллеры передавали его в Query/Command:

```php
final readonly class RequestUserId
{
    public function __construct(private Security $security) {}

    public function get(): UserId
    {
        $user = $this->security->getUser();
        if (!$user instanceof SecurityUser) {
            throw new \RuntimeException('No authenticated user.');
        }
        return new UserId($user->id);
    }
}
```

### Live Components

Для интерактивного фронта используются Symfony UX Live Components
(`DocumentsListComponent`, `SessionsListComponent`) — они вызывают те же
Application-handlers, что и REST-контроллеры.

---

## 7. Связывание зависимостей (DI)

Магия Ports & Adapters происходит в `config/services.yaml`. Сначала всё из `src/`
регистрируется как сервисы, **кроме чистых доменных артефактов** (их нельзя
автосоздавать через контейнер):

```yaml
App\:
    resource: '../src/'
    exclude:
        - '../src/Kernel.php'
        - '../src/*/Domain/Model/'         # агрегаты создаются фабриками, не DI
        - '../src/*/Domain/ValueObject/'   # VO создаются вручную
        - '../src/*/Domain/Event/'
        - '../src/*/Domain/Exception/'
        - '../src/*/Application/DTO/'       # DTO — данные, не сервисы
        - '../src/Shared/Domain/ValueObject/'
        # ...
```

Затем **каждый порт связывается со своим адаптером** через `alias`:

```yaml
# Домен просит UserRepository — контейнер подставит Doctrine-реализацию
App\IdentityAccess\Domain\Repository\UserRepository:
    alias: App\IdentityAccess\Infrastructure\Persistence\Doctrine\Repository\DoctrineUserRepository

App\IdentityAccess\Application\Service\PasswordHasher:
    alias: App\IdentityAccess\Infrastructure\Hashing\Argon2idPasswordHasher

# Один порт Mailer — здесь выбран адаптер LogMailer (можно сменить на SymfonyMailer)
App\Shared\Application\Notification\Mailer:
    alias: App\Notification\Infrastructure\Email\LogMailer

App\DocumentLibrary\Application\Service\FileStorage:
    alias: App\DocumentLibrary\Infrastructure\Storage\LocalFilesystemStorage
```

Именно эти `alias`-строки — «розетки», в которые вставляются адаптеры. Чтобы
заменить реализацию (например, локальное хранилище на S3), достаточно поменять
одну строку, не трогая домен и приложение.

---

## 8. Шина команд/запросов (Messenger)

Подключён Symfony Messenger как инфраструктура для CQRS. Сейчас транспорт —
**синхронный** (`sync://`), т.е. handler выполняется в том же процессе:

```yaml
# config/packages/messenger.yaml
framework:
    messenger:
        transports:
            sync: 'sync://'
```

На практике в части контроллеров handler'ы вызываются **напрямую**
(`($this->handler)(new RegisterUserCommand(...))`), что для sync-режима
эквивалентно отправке в шину. Каталоги `Shared/*/Bus/` зарезервированы под
выделенные command/query/event-шины, когда понадобится async (например, вынести
обработку в очередь).

Где async уже «напрашивается» — извлечение текста из документов. Сейчас оно
**синхронное** прямо в `UploadDocumentHandler`, с явной пометкой в коде:

```php
// Inline (sync) text extraction for MVP. Async via Messenger would be the next step.
if ($this->extractor->supports($command->mimeType, $command->originalName)) {
    try {
        $document->markExtractionReady(
            $this->extractor->extract($command->contents, $command->mimeType, $command->originalName),
        );
    } catch (\Throwable) {
        $document->markExtractionFailed();   // состояние извлечения — часть домена
    }
} else {
    $document->markExtractionUnsupported();
}
```

Состояние извлечения (`ExtractionStatus`) моделируется как часть агрегата
`Document` — это пример того, как технический процесс отражён доменными терминами.

---

## 9. Межконтекстное взаимодействие

Контексты **не лезут в чужие таблицы и модели напрямую**. Связи строятся так:

1. **Через общие Value Objects.** `UserId` определён в `IdentityAccess`, но
   используется как ссылка в `Interview`, `DocumentLibrary`, `Billing`, `Usage`.
   Это «общий язык» (shared kernel в минимальном виде).

2. **Через ссылки по id, а не по объектам.** `InterviewSession` хранит
   `documentIds` как `list<string>` (json-колонка), а не коллекцию объектов
   `Document` из чужого контекста. Контексты слабо связаны.

3. **Через вызов чужих репозиториев-портов** в прикладном слое, когда нужно. Пример —
   `Billing` при обработке вебхука Stripe ищет пользователя через
   `IdentityAccess\Domain\Repository\UserRepository`:

```php
// Billing/Application/Webhook/StripeEventDispatcher.php
final readonly class StripeEventDispatcher
{
    public function __construct(
        private SubscriptionRepository $subscriptions,
        private PaymentRepository $payments,
        private UserRepository $users,          // порт чужого контекста IdentityAccess
    ) {}

    public function dispatch(array $event): void
    {
        match ((string)($event['type'] ?? '')) {
            'customer.subscription.created',
            'customer.subscription.updated' => $this->onSubscriptionUpsert(...),
            'invoice.paid'                  => $this->onInvoicePaid(...),
            // ...
        };
    }
}
```

Здесь `StripeEventDispatcher` — это **anti-corruption layer**: он переводит сырой
JSON от Stripe в доменные мутации (`Subscription::syncFromStripe()`,
`Payment::markPaid()`), не пуская формат Stripe внутрь домена.

---

## 10. Карта bounded contexts

| Контекст | Назначение | Ключевые агрегаты / VO |
|---|---|---|
| **IdentityAccess** | Регистрация, вход, токены устройств, верификация e-mail, сброс пароля | `User`, `DeviceToken`, `EmailVerificationToken`, `PasswordResetToken`; `UserId`, `HashedPassword`, `Role` |
| **Interview** | Сессии интервью (ядро продукта): CRUD, клонирование, привязка документов | `InterviewSession`; `InterviewSessionId`, `SessionStatus` |
| **DocumentLibrary** | Загрузка файлов, извлечение текста, лимиты | `Document`; `DocumentId`, `DocumentType`, `ExtractionStatus` |
| **Billing** | Подписки и платежи через Stripe (вебхуки) | `Subscription`, `Payment`; `Plan`, `SubscriptionStatus` |
| **Usage** | Учёт потребления за период | `UsageRecord`; `UsageKind` |
| **Notification** | Отправка писем (порт + адаптеры, шаблоны) | порт `Mailer`, `TemplateRegistry` |
| **Admin** | Административные read-сценарии | `ListUsersHandler`, `ListPaymentsHandler` |
| **Shared** | Общие VO/исключения/порты, инфраструктурные хелперы | `Email`, `NotFoundException`, `AccessDeniedException`, `Mailer`, `RequestUserId` |

---

## 11. Поток данных: пример сквозного сценария

Регистрация пользователя «сверху вниз»:

```
HTTP POST /auth/sign-up
   │
   ▼
[UI]  RegisterController
   │  парсит JSON → new RegisterUserCommand(email, password)
   ▼
[Application]  RegisterUserHandler::__invoke()
   │  • new Email() ........................... валидация ввода (Domain VO)
   │  • users->findByEmail() .................. порт UserRepository
   │  • User::register() ...................... фабрика агрегата (Domain)
   │  • passwordHasher->hash() ................ порт PasswordHasher
   │  • users->save() ......................... порт UserRepository
   │  • verificationTokens->save() ........... порт
   │  • mailer->send() ........................ порт Mailer
   ▼
[Infrastructure]  DoctrineUserRepository / Argon2idPasswordHasher / LogMailer
   │  реальные адаптеры: БД, хеш, лог-письмо
   ▼
[Application] возвращает UserId
   ▼
[UI]  RegisterController → JSON 201 Created {id, email}
        (ошибки: InvalidEmail/InvalidArgument→400, EmailAlreadyTaken→409)
```

---

## 12. Итог: какие паттерны DDD здесь реально применены

✅ **Bounded Contexts** — 8 изолированных модулей по бизнес-областям.
✅ **Layered / Hexagonal architecture** — Domain ← Application ← UI, Infrastructure
   реализует порты.
✅ **Ports & Adapters** — интерфейсы в Domain/Application, реализации в
   Infrastructure, связка через `alias` в `services.yaml`.
✅ **Value Objects** — самовалидирующиеся неизменяемые `Email`, `UserId`, enum-статусы.
✅ **Aggregates** с приватными конструкторами, фабриками и поведенческими методами,
   защищающими инварианты.
✅ **Repository pattern** — доменные интерфейсы + Doctrine-реализации.
✅ **CQRS** — раздельные Command/Query + handlers; запросы возвращают DTO-View.
✅ **Application Services (use-case handlers)** — тонкая оркестрация.
✅ **Anti-Corruption Layer** — `StripeEventDispatcher` изолирует формат Stripe.
✅ **Domain Exceptions** → маппинг на HTTP в UI.
✅ **Shared Kernel (минимальный)** — `UserId`, `Email`, общие исключения в `Shared`.
✅ **Dependency Inversion / DI** — всё связано через автоконфигурацию + alias'ы.

### Зоны для дальнейшего «ужесточения» (по коду видно намерение)
- Вынести Doctrine-маппинг из агрегатов в `Infrastructure/.../Mapping/` (каталоги
  уже зарезервированы) — убрать ORM-атрибуты из домена.
- Задействовать доменные события (`Domain/Event/`) и event-subscribers вместо прямых
  побочных эффектов в handler'ах (письма, usage).
- Перевести часть сценариев на async-транспорт Messenger (извлечение текста, письма)
  через выделенные шины в `Shared/*/Bus/`.

---

*Документ сгенерирован на основе анализа исходного кода в `app/src/` и
`app/config/`.*
