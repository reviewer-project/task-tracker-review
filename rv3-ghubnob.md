[ghubnob/task-tracker](https://github.com/ghubnob/task-tracker)

## ХОРОШО

1. **Разделение REST-контракта и реализации.** `AuthApi`/`TaskApi` в пакете `controller/api` несут весь mapping, `@Valid`, `@Operation` и `@ApiResponse` прямо на методах интерфейса, а `AuthController`/`TaskController` только делегируют в сервис. Отдельного пакета `documentation`/`swagger` под аннотации нет — документация не оторвана от контракта.

2. **Приветственное письмо публикуется после коммита.** `UserEventListener#onUserRegistered()` подписан на `@TransactionalEventListener(phase = AFTER_COMMIT)`, поэтому Kafka-сообщение уходит только после успешного сохранения пользователя. Это ровно то место, где часто ошибаются, и здесь оно сделано верно.

3. **Регистрация защищена от гонки по уникальности.** `UserService#register()` сначала проверяет `existsByUsername`/`existsByEmail`, а `ExceptionsHandler` дополнительно перехватывает `DataIntegrityViolationException` от constraint'ов БД и превращает её в 409 — параллельная регистрация одинакового email не даст два аккаунта и не свалится в 500.

4. **У backend есть настоящие интеграционные тесты.** `FlowIntegrationTest` поднимает Postgres и Kafka через Testcontainers и проверяет регистрацию, создание/обновление/удаление задачи, 401 без токена, 403 при попытке чужой задачи и 400 на невалидных данных — это выходит за рамки голого соответствия ТЗ.

## ЗАМЕЧАНИЯ

[backend] пакет /auth/jwt

1. `JwtTokenProvider#isTokenValid()` не проверяет ничего нового — обе его проверки к этому моменту уже гарантированно истинны

```java
public boolean isTokenValid(String token, UserDetails user) {
    final String username = extractUsername(token);
    return (username.equals(user.getUsername()) && !isTokenExpired(token));
}
```

Подпись и срок действия токена реально проверяются в `extractAllClaims()` (`Jwts.parser().verifyWith(key).build().parseSignedClaims(token)`) — `jjwt` сам бросает `JwtException` на невалидной подписи или истёкшем `exp`. Но в `JwtAuthenticationFilter` `isTokenValid()` вызывается уже после того, как `extractUsername(jwt)` успешно распарсил тот же токен — то есть до `isTokenValid()` доходят только токены, которые уже прошли проверку подписи и точно не просрочены. Из-за этого `!isTokenExpired(token)` внутри метода не может стать `false` в этом месте вызова, а `username.equals(user.getUsername())` — тавтология, потому что `user` получен через `loadUserByUsername(username)` с тем же самым `username`. Метод с говорящим именем "проверка валидности" на деле не принимает решений и просто ещё дважды парсит уже распарсенный токен.

**Рекомендация:**

Парси токен один раз за запрос и убери `isTokenValid`/`isTokenExpired` — единственным сигналом валидности пусть будет успешный или неуспешный парсинг:

```java
public Claims parseAndValidate(String token) {
    try {
        return Jwts.parser().verifyWith(key).build().parseSignedClaims(token).getPayload();
    } catch (JwtException e) {
        throw new IllegalArgumentException("Invalid JWT token", e);
    }
}
```

и в `JwtAuthenticationFilter` бери `username` из `claims.getSubject()` того же вызова, без повторных парсингов.

[backend] пакет /configuration

2. Origin'ы CORS заданы через `@Value`, а не через типизированный проперти-класс

`SecurityConfig` уже содержит `@Value("${app.cors.allowed_origins}") List<String> allowedOrigins`, при этом для JWT в проекте есть отдельный валидируемый `JWTProperties`. Для CORS такого класса нет, и настройка держится на голом `@Value` прямо в конфиг-классе.

**Рекомендация:**

Вынеси `allowed_origins` в отдельный проперти-класс (например, `CorsProperties` рядом с `JWTProperties`) по уже принятому в проекте образцу.

[backend] пакет /dto/request

3. `RegisterRequest.email` и `AuthRequest.email` помечены `@Email` без обязательности поля

`@Email` у Hibernate Validator считает `null` и пустую строку валидными значениями — обязательность поля должна обеспечиваться отдельной аннотацией. Сейчас у обоих DTO её нет, поэтому запрос без поля `email` или с `email: ""` пройдёт валидацию и упадёт уже на `NOT NULL`-constraint в БД, вернув 409/500 вместо понятной 400-ошибки валидации.

**Рекомендация:**

Добавь `@NotBlank` рядом с `@Email` в `RegisterRequest` и `AuthRequest`.

4. `TaskUpdateRequest.completed` — примитив `boolean`, поэтому пропущенное поле не отличить от `false`

```java
public record TaskUpdateRequest(..., boolean completed) {}
```

`completed` объявлен как примитив, а не `Boolean`, поэтому на нём нельзя поставить `@NotNull`, и Jackson по умолчанию не падает ни на отсутствующем поле `completed` в JSON, ни на явном `"completed": null` — в обоих случаях тихо подставляется `false`. Раз `TaskService#updateTask()` всегда применяет `req.completed()` для установки или сброса `completedAt`, клиент, случайно не передавший `completed`, молча снимет отметку о выполнении и обнулит `completedAt` у уже завершённой задачи — без единой 400-ошибки валидации.

**Рекомендация:**

Замени тип поля на `Boolean` и добавь `@NotNull(message = "Completed status must be specified!")`, чтобы отсутствие или `null` в этом поле отклонялись валидацией, а не тихо трактовались как `false`.

[backend] пакет /service

6. `TaskService` и `UserService` — конкретные классы без интерфейсов, хотя их напрямую вызывают контроллеры

`TaskController` работает с `TaskService`, а `AuthController` — с `UserService` напрямую, без прослойки в виде интерфейса. Оба класса как раз попадают под тот случай, когда интерфейс оправдан: это не внутренний форматтер или маппер, а слой, к которому обращается контроллер и который задаёт понятный набор бизнес-операций (создание/обновление/удаление задачи, регистрация/логин). Сейчас, чтобы подменить реализацию в тесте или добавить альтернативную стратегию, нужно менять сигнатуру самого класса, а не контракта. Этот же пункт отмечался в прошлом ревью проекта и до сих пор не был закрыт.

**Рекомендация:**

Выдели `TaskService` и `UserService` в интерфейсы (`TaskService`/`TaskServiceImpl`, `UserService`/`UserServiceImpl`), чтобы контроллеры зависели от контракта, а не от конкретной реализации — по тому же принципу, что уже применён для `TaskApi`/`AuthApi`.

7. `KafkaService` — название не говорит о том, за что класс отвечает

```java
@Service
public class KafkaService {
    public void sendMessage(EmailMessageDto msg) { ... }
}
```

Класс называется по технологии (Kafka), а не по своей бизнес-роли — а роль у него ровно одна: публикация задачи на отправку email-уведомления в `EMAIL_SENDING_TASKS`. Если backend когда-нибудь заведёт другого Kafka-продюсера для другой цели, имя `KafkaService` перестанет быть уникальным по смыслу и придётся придумывать что-то вроде `KafkaService2`.

**Рекомендация:**

Переименуй класс так, чтобы имя отражало бизнес-роль, а не транспорт — например, `EmailNotificationPublisher`, оставив тот же единственный метод отправки.

8. `TaskService#updateTask()` перезаписывает `completedAt` при каждом сохранении с `completed = true`

```java
if (req.completed()) {
    task.setCompleted(true);
    task.setCompletedAt(Instant.now());
} else {
    task.setCompleted(false);
    task.setCompletedAt(null);
}
```

`completedAt` выставляется в `Instant.now()` всегда, когда в запросе `completed = true`, а не только в момент перехода из невыполненной в выполненную. Поскольку `TaskUpdateRequest` — это full-replace PATCH (title, text и completed передаются всегда), обычное редактирование заголовка уже выполненной задачи (с тем же `completed = true`) молча сдвигает `completedAt` на текущий момент и стирает реальное время завершения.

**Рекомендация:**

Сравнивай `req.completed()` с текущим `task.isCompleted()` и трогай `completedAt` только при фактическом переходе состояния:

```java
if (req.completed() != task.isCompleted()) {
    task.setCompletedAt(req.completed() ? Instant.now() : null);
}
task.setCompleted(req.completed());
```

9. `TaskService#updateTask()`/`deleteTask()` по-разному отвечают на несуществующую и на чужую задачу

Сейчас `findById` без учёта пользователя даёт `TaskNotExistException` → 404, если задачи вообще нет, и `TaskAccessDeniedException` → 403, если она принадлежит другому пользователю. Различие в коде ответа позволяет по одному запросу понять, существует ли конкретный `taskId`, даже если он чужой. Для учебного проекта это не критично, но раз задачи ссылаются по числовому id, перебор становится тривиальным.

**Рекомендация:**

Ищи задачу сразу с учётом пользователя (`findByIdAndUserId`) и бросай `TaskNotExistException` в обоих случаях — и когда задачи нет, и когда она принадлежит другому.

10. `UserService#register()`/`DatabaseUserDetailsService#loadUserByUsername()` не нормализуют email

Email сохраняется и ищется как есть, без приведения регистра. `existsByEmail`/`findByEmail` и уникальный constraint в БД чувствительны к регистру, поэтому `Test@Mail.com` и `test@mail.com` благополучно создадут два разных аккаунта, хотя по смыслу это один и тот же адрес.

**Рекомендация:**

Нормализуй email (trim + toLowerCase), чтобы и проверка уникальности, и поиск при логине всегда работали с одним и тем же значением.

[common] пакет /constant

11. `KafkaTopics` — множественное число в имени класса, а сам топик — жёсткая Java-константа

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class KafkaTopics {
    public static final String EMAIL_SENDING_TASKS = "EMAIL_SENDING_TASKS";
}
```

По конвенции Java классы называют в единственном числе — `KafkaTopics` по имени больше похож на пакет с несколькими константами, чем на тип. Кроме названия, есть более практичный момент: имя топика зашито Java-константой на этапе компиляции, и чтобы завести другое имя топика под другое окружение (например, `dev.email_sending_tasks` вместо `prod.email_sending_tasks`), нужно пересобирать все три сервиса — тогда как cron, SMTP и JWT в проекте уже вынесены в `application.yaml`/`.properties` именно ради такой гибкости.

**Рекомендация:**

Замени константу на проперти `app.kafka.topics.email-sending-tasks` в каждом из трёх сервисов, а согласованность между backend, scheduler и mail-sender обеспечь так же, как сейчас обеспечена согласованность `KAFKA_SERVERS`, — одной переменной окружения в `compose.yaml`, проброшенной с одинаковым значением во все три сервиса.

[mail-sender] пакет /service

13. `MailService` держит `app.mail.from` через `@Value` и не проверяет обязательные SMTP-настройки при старте

`from` внедряется напрямую через `@Value` в бизнес-сервис, а host/port/username/password идут через стандартные `spring.mail.*` без какой-либо валидации на старте. Если `MAIL_USERNAME`/`MAIL_SMTP_PASS`/`MAIL_FROM` не заданы в окружении, сервис поднимется штатно и упадёт только в момент первой попытки отправки письма.

**Рекомендация:**

Заведи в `mail-sender` типизированный `MailProperties` (по аналогии с `JWTProperties` в backend) с `@NotBlank` на обязательных полях и используй его в `MailService` вместо `@Value`.

14. `MailService#send()` — ручной маппинг `EmailMessageDto` в `SimpleMailMessage` прямо в методе отправки

Сборка письма через `set*` лежит внутри `send()` вперемешку с логированием и вызовом `mailSender.send()`, из-за чего метод отвечает сразу за две вещи: сборку письма и его отправку.

**Рекомендация:**

Заведи `MailMessageMapper` через MapStruct с `@Mapping(source = "recipient", target = "to")` — `subject`/`text` совпадают по имени и замапятся сами:

```java
@Mapper(componentModel = "spring")
public interface MailMessageMapper {
    @Mapping(source = "recipient", target = "to")
    SimpleMailMessage toMailMessage(EmailMessageDto message);
}
```

15. `MailService` типизирует поле как `JavaMailSender`, хотя использует только метод `MailSender`

`send()` вызывает `mailSender.send(SimpleMailMessage)` — этот метод объявлен в `MailSender`, `JavaMailSender` его только наследует. Ничего специфичного для `JavaMailSender` (`createMimeMessage()`, `MimeMessageHelper`, HTML-тело, вложения) в классе не используется. Более широкий интерфейс в поле не даёт сервису никаких возможностей сверх того, что даёт `MailSender`, зато заявляет более широкий контракт, чем реально нужен.

**Рекомендация:**

Сузь тип поля до `MailSender` — это тот же бин `JavaMailSenderImpl`, который создаёт autoconfiguration, Spring подставит его без изменений в конфигурации. На `JavaMailSender` возвращайся осознанно, когда `mail-sender` реально понадобятся его возможности — HTML-контент или вложения через `MimeMessageHelper`.

[mail-sender] пакет /listener

16. `EmailSendingListener` — имя описывает транспорт (Kafka listener), а не роль класса

```java
@Component
public class EmailSendingListener {
    final MailService mailService;

    @KafkaListener(topics = KafkaTopics.EMAIL_SENDING_TASKS)
    public void handle(EmailMessageDto message) { ... }
}
```

Та же логика, что и в замечании про `KafkaService` (см. п.7): суффикс `Listener` говорит о том, что класс подписан на `@KafkaListener`, а не о том, зачем он это делает. Единственная задача класса — принять задачу на отправку письма из Kafka и передать её в `MailService`.

**Рекомендация:**

Переименуй в `EmailConsumer` (или `EmailTaskConsumer`/`EmailNotificationConsumer`) — по аналогии с переименованием `KafkaService` → `EmailNotificationPublisher` из п.7: имя должно называть бизнес-роль, а не механизм подписки.

17. `EmailSendingListener#handle()` — сообщение в `log.error` набрано капсом, не в стиле остальных логов проекта

```java
log.error("ERROR HANDLING EMAIL_SENDING_TASKS FOR MESSAGE: {}", message.toString(), e);
```

Во всех остальных логах в проекте (включая соседний `log.info` строкой выше) сообщение — обычное предложение с маленькой буквы, капсом ничего не выделяется. Это единственное место, где стиль лога выбивается. Заодно `message.toString()` избыточен: `{}` в SLF4J сам вызывает `toString()` у переданного объекта.

**Рекомендация:**

Приведи к общему стилю: `log.error("Failed to handle email task for {}", message.recipient(), e);` — без капса и без явного `toString()`.

[scheduler] пакет /scheduler

19. `DailyReportScheduler` целиком держит логику отчёта, а не только сам триггер по расписанию

`DailyReportScheduler` одновременно и Spring-триггер (`@Scheduled`), и место, где идёт выборка из `UserRepository`/`TaskRepository`, группировка задач по пользователю, сборка текста письма (`buildTextReports`/`appendTitles`) и публикация в Kafka. 
Сервисного слоя между шедулером и репозиториями/Kafka нет — сравни с backend, где `TaskController` делегирует в `TaskService`, а публикация в Kafka вынесена в `KafkaService`. 
Здесь `@Scheduled`-метод и есть вся бизнес-логика.

**Рекомендация:**

Вынеси выборку, группировку, сборку текста и публикацию в отдельный сервис, внедрив в него `UserRepository`, `TaskRepository` и Kafka-публикацию. `DailyReportScheduler` тогда останется тонким: только `@Scheduled`-метод, вызывающий `dailyReportService.sendReports()`. Так же, как и `buildTextReports`/`appendTitles`, логику можно будет протестировать unit-тестом без поднятия Spring-контекста и переиспользовать за пределами шедулера, если отчёт понадобится запускать не только по расписанию (например, вручную через эндпоинт).

20. Cron и таймзона зашиты прямо в `@Scheduled(cron = "@daily", zone = "Europe/Moscow")`

Работает и для учебного проекта не создаёт серьёзных проблем. При этом в scheduler нет ни одного проперти-класса, и значения cron/timezone нельзя поменять между окружениями без пересборки.

**Рекомендация:**

Вынеси cron и timezone в `application.yaml` и типизированный `SchedulerProperties` (`config/properties`), а `@Scheduled` подставляй через `${scheduler.cron}`/`${scheduler.zone}`.

21. `DailyReportScheduler#sendDailyReports()` считает границу суток как скользящие 24 часа от момента запуска, а не календарный день

```java
Instant since = Instant.now().minus(1, ChronoUnit.DAYS);
```

Это не `[startOfDay, startOfNextDay)`, а «последние 24 часа от текущего момента исполнения». При небольшом дрейфе времени запуска (перезапуск планировщика, задержка джобы) окно перестаёт совпадать с календарными сутками — часть завершений может попасть в отчёт дважды или не попасть вовсе.

**Рекомендация:**

Считай границы явно от календарного дня в зоне из конфигурации:

```java
ZoneId zone = ZoneId.of(schedulerProperties.zone());
Instant startOfDay = LocalDate.now(zone).atStartOfDay(zone).toInstant();
```

и используй `startOfDay` вместо `Instant.now().minus(1, DAYS)`.

22. `DailyReportScheduler#sendDailyReports()` игнорирует результат `kafkaTemplate.send()`

В отличие от backend'ового `KafkaService`, здесь `send()` вызывается без обработки `CompletableFuture` — если брокер недоступен именно в момент отправки отчёта конкретному пользователю, ошибка нигде не будет видна, а лог покажет, что письмо "отправлено".

**Рекомендация:**

Обрабатывай результат `send()` через `whenComplete` и логируй ошибку с адресом пользователя, как это уже сделано в backend'овом `KafkaService`.

## РЕКОМЕНДАЦИИ

1. У `scheduler` и `mail-sender` есть только тривиальный `contextLoads()`-тест. Логика построения отчёта (`buildTextReports`/`appendTitles`) и границы суток — чистые методы, которые легко покрыть unit-тестами, и именно там сейчас живёт найденная ошибка с датами.

2. Стоит выровнять подход к обработке ошибок Kafka-отправки между сервисами: backend логирует результат `send()`, scheduler — нет. Единый паттерн (например, общий небольшой wrapper вокруг `KafkaTemplate` с логированием) сделает поведение предсказуемым при сбоях брокера.

3. В проекте уже есть один хорошо продуманный проперти-класс (`JWTProperties` с валидацией). Стоит довести этот же подход до CORS, SMTP и Kafka-топика — тогда все внешние конфигурации будут проверяться на старте и переопределяться по окружениям одинаково, а не только JWT.

4. По мере роста количества задач стоит присмотреться к индексам на `tasks.completed` и `tasks.completed_at` — именно по ним `scheduler` ежедневно фильтрует всю таблицу целиком, и это единственное место, где полное сканирование будет расти вместе с данными.

5. `findAllByCompletedFalse()` и `findAllByCompletedTrueAndCompletedAtAfter(since)` в `TaskRepository` — это два отдельных прохода по одной и той же таблице `tasks` с частично пересекающимися условиями. Их можно объединить в один запрос:

```java
@Query("SELECT t FROM TaskEntity t WHERE t.completed = false OR (t.completed = true AND t.completedAt > :since)")
List<TaskEntity> findAllRelevantForDailyReport(@Param("since") Instant since);
```

а в `DailyReportScheduler` сначала сгруппировать результат по `userId`, а внутри каждой группы разделить задачи на `incomplete`/`completed` по `task.isCompleted()`. Сокращает число запросов к БД за один прогон с двух до одного.

6. Стоит зафиксировать, что `TaskUpdateRequest` — это full-replace операция (PATCH ведёт себя как PUT), а не частичное обновление. Это не ошибка, но если в будущем понадобится частичное редактирование (например, только текста без указания `completed`), лучше сразу завести отдельный DTO, а не расширять текущий опциональными полями.

## ИТОГ

Проект сделан на хорошем уровне. Границы между backend, scheduler и mail-sender соблюдены: сущности и репозитории не смешиваются, миграциями управляет один сервис, а Kafka-контракт вынесен в общий модуль. Важные места вроде отправки письма до коммита, N+1, пустых писем и утечки entity в REST реализованы корректно.

Главная ошибка — в `TaskService#updateTask()`: при повторном сохранении выполненной задачи меняется `completedAt`, из-за чего теряется реальное время завершения. Это стоит исправить в первую очередь.

Ещё один важный момент — `DailyReportScheduler`: окно в последние 24 часа лучше заменить на календарные сутки, иначе при сдвиге расписания отчёты могут рассинхронизироваться.

Остальные замечания менее критичны: отсутствие интерфейсов у сервисов, нетипизированные настройки, слабая валидация email и лишняя логика в `JwtTokenProvider#isTokenValid()`.
