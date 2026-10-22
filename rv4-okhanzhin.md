[okhanzhin/task-tracker](https://github.com/okhanzhin/task-tracker)

## ХОРОШО

1. Правило «статус и `completedAt` меняются вместе» проверяется не только в коде, но и в самой базе через `CHECK`-ограничение `chk_tasks_completed_at`

2. Внутренний API для отчётов закрыт отдельным сервисным токеном с правом `internal:reports:read`. Данные отдаются постранично, и на каждую страницу уходит всего два запроса к базе

3. Модели Kafka-сообщений лежат отдельно в `task-tracker-common/model/messaging`. Консьюмеры не завязаны на пакеты отправителя, и на контракты есть тесты

4. В email-sender настроены повторные попытки и очередь для неотправленных писем (DLT). Сообщения с ошибками в данных сразу уходят туда, без лишних повторов

5. Работа с GigaChat вынесена в отдельный сервис. В промпте есть защита от команд, которые кто-то может спрятать в названии задачи

6. Если отчёт для одного пользователя упал, шедулер продолжает рассылку остальным

## ЗАМЕЧАНИЯ

### [task-tracker-backend] корневой пакет

1. Токен для шедулера выпускается закомментированным кодом в `TaskTrackerApplication`

Чтобы получить `SCHEDULER_SERVICE_TOKEN`, нужно раскомментировать `CommandLineRunner`, запустить бэкенд и скопировать токен из консоли. Токен действует 90 дней, а как выпустить новый, нигде не написано. Когда срок истечёт, отчёты просто перестанут приходить

**Рекомендация:**

Сделай отдельный `CommandLineRunner`, который включается профилем `issue-service-token`, и опиши в README, как его запустить

### [task-tracker-backend] пакет /annotation

1. `PasswordMatchesValidator#isValid()` падает, если пароль не передан

Проверка совпадения паролей запускается, даже когда поле `password` пустое. На запрос регистрации без пароля сервер отвечает 500 вместо 400

**Рекомендация:**

Сравнивай через `Objects.equals(request.getPassword(), request.getConfirmPassword())`

### [task-tracker-backend] пакет /controller

1. Нет API-интерфейсов и OpenAPI

Маппинги и логика контроллера лежат в одном классе, а все методы возвращают `ResponseEntity<?>`, так что по сигнатуре не видно, что придёт в ответе. API описан только в README и в Postman, и эти описания уже не совпадают с кодом

**Рекомендация:**

Подключи springdoc и для каждого контроллера заведи интерфейс `*Api` с маппингами, `@Operation` и конкретными типами ответов:

```java
@RequestMapping("/api/v1/tasks")
public interface TaskApi {

    @Operation(summary = "Редактирование задачи")
    @PatchMapping("/{taskId}")
    TaskResponse updateTask(
            @PathVariable long taskId,
            @AuthenticationPrincipal SecurityUser principal,
            @Valid @RequestBody UpdateTaskRequest request
    );
}
```

2. `logout()` на самом деле ничего не делает

Метод только ставит пустой заголовок в ответ и чистит `SecurityContextHolder`. Токен после логаута продолжает работать до конца срока, и если его кто-то утащил, выход из аккаунта никак не поможет

**Рекомендация:**

Добавляй в токен `jti` при генерации в `JwtServiceImpl`, на логауте сохраняй его в таблицу отозванных токенов вместе с `exp`, а в `JwtRequestFilter` отклоняй токены, чей `jti` есть в этой таблице. Старые записи чисти по `expires_at`, после истечения срока они уже не нужны

```java
@Transactional
public void logout(String token) {
    Claims claims = jwtService.getClaims(token);
    revokedTokenRepository.save(new RevokedToken(claims.getId(), claims.getExpiration().toInstant()));
}
```

```java
if (revokedTokenRepository.existsById(claims.getId())) {
    throw new BadCredentialsException("Token revoked");
}
```

### [task-tracker-backend] пакет /exception

1. `handleGenericException()` отвечает 500 на ошибки клиента

Общий обработчик ловит всё подряд: кривой JSON, неподдерживаемый метод, несуществующий адрес. Получается, что если клиент отправил кривой JSON, ему отвечают «ошибка сервера»

**Рекомендация:**

Добавь обработчик `HttpMessageNotReadableException` с ответом 400, а в общем обработчике бери статус из исключения, если оно реализует `org.springframework.web.ErrorResponse`

### [task-tracker-backend] пакет /mapper

1. Email сохраняется в исходном регистре в `UserMapper#map()`

Можно зарегистрировать `User@Mail.ru` и `user@mail.ru` как два разных аккаунта. А войти получится, только если набрать email в том же регистре, что при регистрации

**Рекомендация:**

Приводи email к нижнему регистру в `UserMapper#map()` и в `AuthenticationServiceImpl#authenticate()`

2. Мапперы написаны руками и смешаны с бизнес-логикой

`TaskMapper` и `UserMapper` вручную перекладывают поля через билдеры, и при добавлении нового поля его легко забыть в одном из методов. 
Заодно в мапперы уехала логика, которой там не место: `UserMapper#map()` хеширует пароль и ставит статус `ACTIVE`, а `TaskMapper#map()` решает, когда ставить `completedAt`, и для этого тянет `Clock`

**Рекомендация:**

Перейди на MapStruct, а хеширование пароля и выставление статуса и `completedAt` перенеси в сервис и сущность

```java
@Mapper(componentModel = "spring")
public interface TaskMapper {

    TaskResponse toResponse(Task task);

    @Mapping(target = "taskId", source = "id")
    TaskData toTaskData(ReportTaskView view);
}
```

### [task-tracker-backend] пакет /model/dto

1. `SecurityUser` лежит среди DTO

**Рекомендация:**

Перенеси его в `/security`

2. Блокировка пользователя не работает

`SecurityUser` не переопределяет `isEnabled()`, поэтому пользователь со статусом `BLOCKED` входит как обычно

**Рекомендация:**

Возвращай `isActive` из `isEnabled()` и проверяй это в `JwtRequestFilter`

3. `UserDTO` содержит хеш пароля, а `TaskDTO` вообще не используется

`UserServiceImpl#createUser()` возвращает DTO вместе с хешем пароля, хотя дальше он нужен только для генерации токена

**Рекомендация:**

Удали `TaskDTO` и `UserDTO`, а из `createUser()` возвращай `SecurityUser`

### [task-tracker-backend] пакет /model/entity

1. Сущности называются `UserDAO` и `TaskDAO`

DAO — это класс для доступа к базе, а у тебя так называются сами сущности. Это путает

**Рекомендация:**

Переименуй их в `User` и `Task`

2. `@Data` на сущностях

`toString` и `hashCode` тянут за собой связанные `tasks` и `owner`. Если загружены обе стороны, получится бесконечная рекурсия. Ещё через открытые сеттеры можно поменять `completed`, не тронув `completedAt`

**Рекомендация:**

Замени `@Data` на `@Getter`, удали неиспользуемое `UserDAO#tasks` и перенеси смену статуса в сущность. Нельзя использовать ломбоковский @Data на сущностях спринга, это знаментая ошибка которая может сыграть злую шутку.

### [task-tracker-backend] пакет /producer

1. `EmailTaskProducer#send()` не проверяет, ушло ли сообщение

Если Kafka недоступна, регистрация пройдёт, письмо не уйдёт, и по логам этого не узнать

**Рекомендация:**

Добавь `whenComplete` с логированием ошибки, как в `SummarizationResponseProducer`

### [task-tracker-backend] пакет /security

1. `JwtRequestFilter` ловит не все ошибки аутентификации

Если пользователя из токена уже удалили, или у токена неизвестный тип, исключение пролетает мимо `catch`, и клиент получает непонятный ответ вместо 401. Кроме того, `try` охватывает `filterChain.doFilter()`, поэтому любые `IllegalArgumentException` из следующих фильтров тоже превращаются в 401

**Рекомендация:**

Оставь в `try` только разбор токена и лови там `JwtException | AuthenticationException`

### [task-tracker-backend] пакет /service

1. `getAllTasks()` делает лишний запрос и возвращает задачи в случайном порядке

Пользователь загружается из базы только для того, чтобы передать его в `findDistinctByOwner`. Порядок задач не задан, и на фронте они могут перемешиваться

**Рекомендация:**

Используй `findAllByOwnerIdOrderByIdAsc(ownerId)`

2. Письмо уходит после сохранения пользователя только по счастливой случайности

Сейчас всё работает, потому что `save()` сам завершает транзакцию. Но если кто-то добавит `@Transactional` на `createUser()`, письмо начнёт уходить ещё до сохранения пользователя

**Рекомендация:**

Публикуй событие регистрации и отправляй письмо из `@TransactionalEventListener(phase = AFTER_COMMIT)`

### [task-tracker-backend] пакет /service/impl

1. `JwtServiceImpl` читает настройки через `@Value`

**Рекомендация:**

Создай `JwtProperties` в `config/properties` с `@Validated` и внедри его в `JwtServiceImpl`

### [task-tracker-backend] пакет /utils

1. `ReportRequestValidator` проверяет параметры вручную

Для `limit` и `afterUserId` хватило бы обычных аннотаций, а вместо этого написаны ручные проверки и три отдельных исключения

**Рекомендация:**

Поставь `@Min`, `@Max` и `@Positive` на параметры `InternalReportController`, а валидатор и лишние исключения удали

2. В `/utils` сложено то, что относится к конкретным частям кода

`JwtClaims` нужен security, а `ConstrainViolationUtils` и текст приветственного письма — регистрации

**Рекомендация:**

Разнеси эти классы по пакетам, где они используются, и удали `/utils`

### [task-tracker-backend] resources/db/changelog

1. Настройка `APP_SCHEMA` ни на что не влияет

В миграциях и в `@Table` жёстко прописана схема `tasks_tracker`, а `APP_SCHEMA` указывает на `task_tracker`, которую никто не создаёт

**Рекомендация:**

Убери имя схемы из `@Table` и SQL-файлов и бери его из `APP_SCHEMA`

2. Код опирается на имя `users_email_key`, которое база придумала сама

**Рекомендация:**

Задай имя ограничения явно (`CONSTRAINT uk_users_email UNIQUE (email)`) и используй его в коде

### [task-tracker-scheduler] пакет /client

1. Одна упавшая страница в `TaskTrackerApiClient#getDailyReportData()` срывает весь отчёт

Повторных попыток нет. Если бэкенд моргнул на третьей странице, исключение улетает из `DailyReportRequestSource#forEach()` наверх, и все пользователи с этой страницы и дальше остаются без отчёта до следующей ночи

**Рекомендация:**

Подключи `spring-retry` и повесь `@Retryable` с backoff на `getDailyReportData()`

### [task-tracker-scheduler] пакет /config

1. `KafkaProducerConfig` и `KafkaConsumerConfig` повторяют то, что Spring Boot делает сам

Boot уже создаёт `KafkaTemplate` и фабрику листенеров по `spring.kafka.*` и сам подхватывает бин `DefaultErrorHandler`. При этом шаблон назван `summarizationRequestKafkaTemplate`, а через него же уходят письма

**Рекомендация:**

Удали оба класса и используй автоконфигурацию

### [task-tracker-scheduler] пакет /mapper

1. `ReportEmailMapper` строит `messageId` из `requestId`, но толку от этого нет

`requestId` каждый запуск генерируется заново случайным UUID, а email-sender по `messageId` дубли не отсекает. Выглядит как защита от повторной отправки, а на деле ничего не защищает

**Рекомендация:**

Генерируй `messageId` через `UUID.randomUUID()`, как при регистрации на бэкенде

### [task-tracker-scheduler] пакет /rpc

1. Синхронный вызов через Kafka

`PendingSummarizationRequestRegistry` превращает Kafka в синхронный вызов: шедулер отправляет запрос в `SUMMARIZATION_REQUESTS`, складывает `CompletableFuture` в мапу в памяти, а `DailyReportJob#processUser()` ждёт ответ через `join()` до 180 секунд на каждого пользователя. Пользователи обрабатываются строго по одному, а если шедулер перезапустится посреди рассылки, все ожидающие ответы потеряются. Ради этого понадобились реестр, таймауты, отдельный топик ответов и консьюмер в шедулере

**Рекомендация:**

Сделай цепочку без ответов: шедулер кладёт `SummarizationRequest` в Kafka и больше ничего не ждёт, а summarization-service сам публикует готовое письмо в `EMAIL_SENDING_TASKS`. Реестр, `SUMMARIZATION_RESPONSES` и `SummarizationResponseListener` после этого можно удалить

### [task-tracker-scheduler] пакет /service

1. Время запуска и часовой пояс отчёта читаются через `@Value`

`DailyReportScheduler` и `ReportEmailMapper` каждый сам читают `app.reports.daily.zone`, и в обоих классах продублировано значение по умолчанию

**Рекомендация:**

Создай `DailyReportProperties` в `config/properties` и используй его в обоих классах

2. Одно и то же проверяется по несколько раз

Период `from < to` проверяют `DailyReportJob#run()`, `DailyReportRequestSource#forEach()`, `ReportEmailMapper`, и ещё раз его проверяет summarization-service. `SummarizationRequestMapper` проверяет ответ бэкенда и кидает `TaskTrackerClientException`, хотя к HTTP-клиенту отношения не имеет. Из-за этих проверок сама логика в классах теряется

**Рекомендация:**

Проверяй ответ бэкенда один раз в `DailyReportRequestSource#validatePage()`, а `requireNonNull` и повторные проверки периода в остальных классах убери

3. `DailyReportScheduler` лежит в `/service`

Это точка входа по крону, а не сервис

**Рекомендация:**

Перенеси его в пакет `/scheduler`

### [task-tracker-scheduler] пакет /service/impl

1. Если GigaChat не ответил, отчёт не отправляется

`UserReportDeliveryServiceImpl` при ошибке суммаризации просто пишет её в лог, и пользователь остаётся без письма. А ведь все данные для отчёта уже есть в `SummarizationRequest`

**Рекомендация:**

Собирай письмо по простому шаблону из `SummarizationRequest`, если суммаризация упала. После перехода на цепочку из замечания про `/rpc` эта логика переедет в summarization-service

### [email-sender-service] пакет /config

1. `KafkaConsumerConfig` теряет настройки листенера из `application.yml`

Фабрика создаётся через `new` без `ConcurrentKafkaListenerContainerFactoryConfigurer`, поэтому `spring.kafka.listener.concurrency: 3` и `ack-mode: record` не применяются. По факту письма читает один поток и подтверждает их пачками, хотя в конфиге написано другое

**Рекомендация:**

Удали `KafkaConsumerConfig`, Spring Boot сам соберёт фабрику по `spring.kafka.listener.*` и подхватит твой `DefaultErrorHandler`

### [email-sender-service] пакет /config/properties

1. `MailProperties` называется так же, как класс в Spring Boot

**Рекомендация:**

Переименуй его в `MailSenderProperties`

### [email-sender-service] пакет /mapper

1. `EmailNotificationMapper` вручную копирует поля в класс-близнец

`EmailNotification` повторяет `EmailSendingTask` поле в поле, и маппер просто перекладывает четыре поля руками

**Рекомендация:**

Замени `EmailNotificationMapper` на интерфейс MapStruct

### [email-sender-service] пакет /utils

1. `EmailSendingTaskValidator` проверяет поля вручную

Правила для сообщения спрятаны внутри консьюмера. При этом соседний `SummarizationRequest` уже описан через аннотации

**Рекомендация:**

Опиши ограничения прямо на полях `EmailSendingTask` и проверяй их через `jakarta.validation.Validator`

### [email-sender-service] resources

1. Нет `ErrorHandlingDeserializer`

Если сообщение не разбирается как JSON, оно не попадёт в DLT, хотя очередь для неотправленных писем у тебя уже настроена

**Рекомендация:**

Укажи `ErrorHandlingDeserializer` с делегатом `JsonDeserializer`

2. В resources лежит пустой `log.txt`

**Рекомендация:**

Удали его

### [summarization-service] пакет /service

1. `ReportPromptBuilder` передаёт в отчёт все задачи

По ТЗ в письме должно быть не больше пяти названий в каждой группе, а сейчас, если задач 80, все 80 и попадут в письмо

**Рекомендация:**

Количество считай по всем задачам, а в промпт передавай только первые пять названий

### [summarization-service] пакет /utils

1. Системный промпт лежит в `utils/Constants`

**Рекомендация:**

Перенеси его в `resources/prompts/daily-report-system.txt`

### [summarization-service] resources/application.yml

1. Нет `ErrorHandlingDeserializer`

**Рекомендация:**

Подключи его так же, как в email-sender

### [task-tracker-common] пакет /model/messaging/summarization

1. Период в `SummarizationRequest` передаётся как `LocalDateTime`

Часовой пояс теряется. В итоге в теме письма время по Москве, а в тексте по UTC, и пользователь видит два разных времени

**Рекомендация:**

Замени поля на `Instant` и передавай часовой пояс отдельным полем

### [инфраструктура] docker-compose.yml и .env.example

1. Не совпадает имя базы

Postgres создаёт базу `tasks-db`, а бэкенд по `.env.example` подключается к `some-db`. Если запустить проект по README, бэкенд не поднимется

**Рекомендация:**

Укажи `POSTGRES_DB: ${POSTGRES_DB}` в сервисе `postgresql`

2. У summarization-service включён SSL-debug

Из-за `-Djavax.net.debug` на каждый запрос к GigaChat в логи сыпятся тысячи строк

**Рекомендация:**

Удали `JAVA_TOOL_OPTIONS` из compose, хватит того, что задано в Dockerfile

3. Остались ненужные переменные

`KAFKA_ZOOKEEPER_CONNECT` и `KAFKA_TOPIC` нигде не используются, а значение `LIQUIBASE_SCHEMA` init-скрипт не учитывает

**Рекомендация:**

Удали эти переменные

4. В `.env.example` `JWT_SECRET` записан не в Base64

Если запустить проект с этим примером, бэкенд упадёт при старте

**Рекомендация:**

Поставь заглушку в формате Base64 и добавь подсказку `openssl rand -base64 32`

5. Нет фронта

По ТЗ фронт должен быть и в репозитории, и в compose

**Рекомендация:**

Добавь его отдельным сервисом в compose

## РЕКОМЕНДАЦИИ

1. Сначала сверяй API с ТЗ, потом добавляй что-то сверху
2. Основные функции не должны ломаться, если упал внешний сервис
3. Описывай контракты между сервисами явно: время передавай в `Instant`, правила проверки — аннотациями
4. Хотя бы раз запусти проект с нуля по README и посмотри, все ли настройки работают
5. Правила, которые касаются сущности, держи в самой сущности
6. Добавь тесты на неправильные запросы: почти все 500-е как раз там

## ИТОГ

Сильная работа, заметно выше учебного минимума. Архитектура продумана, сервисы разделены по делу, видно, что ты думал о сбоях и тестах

Слабые места в основном на стыках: API не совпадает с ТЗ, часть настроек работает не так, как задумано, а ежедневный отчёт целиком зависит от GigaChat. Начни с JWT-секрета в репозитории, это единственная серьёзная дыра. Потом приведи API в соответствие с ТЗ
](https://github.com/okhanzhin/task-tracker)](https://github.com/okhanzhin/task-tracker)
