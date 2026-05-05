[AquaProdigy/tracker](https://github.com/AquaProdigy/tracker-docker-start)

Репозитории: [tracker-auth](https://github.com/AquaProdigy/tracker-auth) · [Tracker-task](https://github.com/AquaProdigy/Tracker-task) · [tracker-scheduler](https://github.com/AquaProdigy/tracker-scheduler) · [tracker-email-sender](https://github.com/AquaProdigy/tracker-email-sender) · [tracker-gateway](https://github.com/AquaProdigy/tracker-gateway)

---

## ХОРОШО

1. **Централизованная аутентификация в `JwtAuthFilter` на уровне gateway** — JWT-валидация и извлечение `userId` выполняются один раз в gateway. Downstream-сервисы получают чистый заголовок `X-User-Id` и не содержат никакой auth-логики. Это правильный паттерн аутентификации в микросервисах: изменение алгоритма проверки токена не требует изменений в `tracker-auth` или `tracker-task`.

2. **DIP в `tracker-email-sender`: `EmailSenderService` интерфейс + `GmailEmailSenderServiceImpl`** — единственный сервис в проекте, где абстракция корректно отделена от реализации. `KafkaConsumerService` зависит от интерфейса, а не от конкретного класса. Смена SMTP-провайдера потребует только новой реализации без изменения потребителя.

3. **`TaskMapper.updateEntity()` с `@BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)`** — частичное обновление реализовано правильно: только ненулевые поля из `TaskUpdateRequest` перезаписывают сущность. Поля, отсутствующие в запросе, остаются нетронутыми. Это корректная семантика для PUT-эндпоинта с опциональными полями.

4. **Разделение внутреннего и внешнего API через выделенный `InternalController`** — эндпоинты для межсервисного взаимодействия (`/internal/**`) отделены от публичных. Идея правильная, однако реализация повторяет один и тот же антипаттерн в обоих сервисах: проверка API-ключа в теле контроллера вместо интерсептора (см. замечания 12 и 21).

---

## ЗАМЕЧАНИЯ

### [tracker-gateway] пакет / (корневой)

**1. `JwtAuthFilter` расположен в корневом пакете**

`JwtAuthFilter` находится непосредственно в `org.example.trackergateway` рядом с `TrackerGatewayApplication`. Корневой пакет должен содержать только точку входа приложения. Размещение компонентов в корне нарушает организацию пакетов и затрудняет навигацию.

**Рекомендация:** переместить в подпакет:

```
org.example.trackergateway.filter.JwtAuthFilter
```

**2. Магическое число `TOKEN_SUBSTRING_LENGTH = 7` не связано с константой `BEARER_STARTWITH_TOKEN`**

`TOKEN_SUBSTRING_LENGTH = 7` — это `"Bearer ".length()`, но эта зависимость нигде не выражена в коде. Если `BEARER_STARTWITH_TOKEN` изменится, `TOKEN_SUBSTRING_LENGTH` останется `7` и токен будет обрезаться неверно без каких-либо ошибок компиляции.

**Рекомендация:**

```java
private static final String BEARER_PREFIX = "Bearer ";

// вместо: header.substring(TOKEN_SUBSTRING_LENGTH)
String token = header.substring(BEARER_PREFIX.length());
```

**3. `@Value` в `JwtAuthFilter` с инициализацией через `@PostConstruct`**

Оба поля изменяемы (не `final`), хотя семантически инициализируются один раз. `@PostConstruct` — это обходной приём для инициализации, которую лучше делать через конструктор. Та же проблема с `@Value` вместо `@ConfigurationProperties`, что описана в замечании 7.

**Рекомендация:** вычислять ключ в конструкторе через `@ConfigurationProperties`:

```java
@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {
    private final JwtProperties jwtProperties;
    private final SecretKey signingKey; // final — вычисляется один раз

    public JwtAuthFilter(JwtProperties jwtProperties) {
        this.jwtProperties = jwtProperties;
        this.signingKey = Keys.hmacShaKeyFor(
            jwtProperties.getSecret().getBytes(StandardCharsets.UTF_8)
        );
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String header = exchange.getRequest().getHeaders().getFirst(HttpHeaders.AUTHORIZATION);
        // ...
        Jwts.parser()
            .verifyWith(signingKey)  // ← уже готовое final поле
            .build()
            .parseSignedClaims(token);
        // ...
    }
}
```

**4. Хрупкий whitelist публичных путей в `JwtAuthFilter` — точное совпадение строк**

Проверка через `List.contains()` выполняет точное совпадение строк. Запрос на `/auth/login/` (trailing slash) или `/auth/login?redirect=...` не попадёт в whitelist и получит 401. При добавлении новых публичных эндпоинтов (например, `/auth/forgot-password`) нужно не забыть добавить их в список — это легко упустить.

**Рекомендация:** использовать `AntPathMatcher`:

```java
private final AntPathMatcher pathMatcher = new AntPathMatcher();

private static final List<String> OPEN_PATTERNS = List.of(
        "/auth/**"
);

private boolean isOpenPath(String path) {
    return OPEN_PATTERNS.stream()
            .anyMatch(pattern -> pathMatcher.match(pattern, path));
}
```

---

### [tracker-auth] пакет /advice

**5. `UsernameNotFoundException` в `@ExceptionHandler` — мёртвый код**

```java
@ExceptionHandler({UserNotFoundException.class, UsernameNotFoundException.class})
@ResponseStatus(HttpStatus.NOT_FOUND)
public ErrorMessage handleUserNotFoundException(UserNotFoundException ex) {
    return new ErrorMessage(ex.getMessage());
}
```

`UsernameNotFoundException` бросается внутри `UserDetailsService.loadUserByUsername()` в процессе аутентификации Spring Security. Однако до `@ExceptionHandler` оно никогда не доходит: `DaoAuthenticationProvider` перехватывает его раньше и конвертирует в `BadCredentialsException` — намеренно, чтобы по разному ответу нельзя было определить, существует пользователь или нет (защита от username enumeration).

Помимо этого, параметр метода имеет тип `UserNotFoundException`, тогда как в аннотации объявлены оба типа. Если бы `UsernameNotFoundException` всё же попало в handler, Spring не смог бы разрешить аргумент (`UsernameNotFoundException` не является `UserNotFoundException`) и бросил бы `IllegalStateException` → 500.

Итог: `UsernameNotFoundException` в аннотации — мёртвый код, который никогда не сработает.

**Рекомендация:** убрать `UsernameNotFoundException` из аннотации, оставив только `UserNotFoundException`:

```java
@ExceptionHandler(UserNotFoundException.class)
@ResponseStatus(HttpStatus.NOT_FOUND)
public ErrorMessage handleUserNotFoundException(UserNotFoundException ex) {
    return new ErrorMessage(ex.getMessage());
}
```

---

### [tracker-auth] пакет /config

**6. Пакеты названы во множественном числе — нарушение Java Naming Conventions**

По соглашению об именовании в Java (JLS §6.1 и Google Java Style Guide) пакеты должны быть в единственном числе: `service`, `controller`, `entity`, `repository`. Во всех сервисах проекта используется множественное число:

```
org.example.trackerauth.services
org.example.trackerauth.controllers
org.example.trackerauth.entities
org.example.trackertask.services
org.example.trackertask.controllers
org.example.trackertask.entities
```

**Рекомендация:** переименовать пакеты во всех сервисах:

```
services    → service
controllers → controller
entities    → entity
```

**7. Отсутствует `@ConfigurationProperties` — `@Value` разбросан по сервисам, контроллерам и фильтрам**

В `tracker-auth` конфигурационные значения (`jwt.secret`, `jwt.expiration`, `kafka.topic`, `internal.api-key`) инжектируются через `@Value` напрямую в те классы, где они используются: в `JwtTokenService`, `KafkaSenderService` и `InternalController`. 
Та же картина в `tracker-scheduler` (`TaskSchedulerService` содержит три `@Value`) и в `tracker-gateway` (`JwtAuthFilter`). 
Конфигурация — инфраструктурный слой. Сервисы и контроллеры не должны зависеть от строковых ключей properties.

Почему это плохо:
- При переименовании ключа нужно менять несколько классов в разных слоях.
- `@Value` не даёт валидации при старте: если переменная окружения не задана, приложение стартует с `null` и падает в рантайме.
- Дублирование строковых ключей (`${kafka.topic}` в двух разных сервисах) ведёт к рассинхронизации при рефакторинге.
- Нет единой точки просмотра всей конфигурации модуля.

**Рекомендация:** создать `@ConfigurationProperties`-классы в пакете `/config` для каждого сервиса:

```java
// tracker-auth: /config/JwtProperties.java
@Configuration
@ConfigurationProperties(prefix = "jwt")
@Validated
@Getter
@Setter
public class JwtProperties {
    @NotBlank
    private String secret;
    @Positive
    private long expiration;
}
```

После этого `JwtTokenService` становится:

```java
@Component
@RequiredArgsConstructor
public class JwtTokenService {
    private final JwtProperties jwtProperties;

    ......
}
```

**8. `JwtTokenService` размещён в пакете `/services`, хотя является инфраструктурным компонентом**

`JwtTokenService` не содержит бизнес-логики. Он инкапсулирует работу с JWT-библиотекой и криптографическими ключами — это инфраструктурный компонент, а не бизнес-сервис.
Его соседство с `AuthService` и `UserService` создаёт ложное впечатление о назначении класса и нарушает структуру слоёв.

**Рекомендация:** перенести в пакет `/security` или `/config`:

**9. Неконсистентный регистр ключей в `application.yml`**

Файл: `tracker-auth/src/main/resources/application.yml`, `tracker-scheduler/src/main/resources/application.yaml`.

В обоих файлах присутствует ключ с заглавной буквой:

```yaml
Kafka:          # <-- заглавная K
  topic: ${KAFKA_TOPIC_NAME}
```

Хотя Spring Boot поддерживает relaxed binding, `@Value("${kafka.topic}")` ссылается на ключ `kafka.topic` (строчная), а не `Kafka.topic`. Это работает благодаря case-insensitive matching в некоторых механизмах Spring, но создаёт неявность и потенциальный баг при использовании строгих профилей или `@ConfigurationProperties`.

**Рекомендация:** привести все ключи к kebab-case:

```yaml
kafka:
  bootstrap-servers: ${KAFKA_HOST}
  producer:
    key-serializer: org.apache.kafka.common.serialization.StringSerializer
    value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
  topic: ${KAFKA_TOPIC_NAME}
```

**10. `SecurityConfig` не настраивает `AuthenticationEntryPoint` — провал аутентификации возвращает 403 вместо 401**

Spring Security по умолчанию обрабатывает `AuthenticationException` через `ExceptionTranslationFilter`, который без явно заданного `AuthenticationEntryPoint` возвращает 403 Forbidden вместо 401 Unauthorized. 
В результате попытка войти с несуществующим email возвращает 403, а фронтенд отображает это как «неверный логин или пароль» — настоящая причина ошибки скрыта.

**Рекомендация:** задать `AuthenticationEntryPoint` в `SecurityConfig`:

```java
http.exceptionHandling(ex -> ex
    .authenticationEntryPoint((request, response, authException) -> {
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED); // 401
        response.setContentType("application/json;charset=UTF-8");
        response.getWriter().write("{\"error\": \"Unauthorized\"}");
    })
);
```

---

### [tracker-auth] пакет /controllers

**11. Отсутствуют интерфейсы у контроллеров и сервисов**

Ни один контроллер и ни один сервис в проекте не имеет интерфейса. Это затрудняет две вещи.

Для **контроллеров** интерфейс даёт версионирование API: контракт эндпоинтов (пути, методы, DTO) описывается в интерфейсе, а при выходе `v2` создаётся новая реализация без изменения старой:

```java
public interface UserApi {
    @GetMapping("/v1/user")
    ResponseEntity<UserDto> getUser(@RequestHeader("X-User-Id") Long userId);
}

@RestController
public class UserController implements UserApi {
    // реализация v1
}
```

Для **сервисов** интерфейс позволяет легко подменять реализацию — для тестов, для смены провайдера или для добавления декоратора — без изменения потребителей. 
В проекте это уже сделано правильно только в `tracker-email-sender` (`EmailSenderService` / `GmailEmailSenderServiceImpl`), но в остальных сервисах паттерн не применяется.

**Рекомендация:** выделить интерфейсы для всех сервисов и контроллеров

**12. Проверка API-ключа в теле контроллера — логика безопасности не на своём месте**

Ты смешал слои: контроллер выполняет проверку авторизации. Это нарушение SRP — контроллер должен только принять запрос и делегировать сервису. Проверка `X-Internal-Api-Key` — cross-cutting concern, которое должно жить в фильтре или интерсепторе. 
Тот же антипаттерн продублирован в `tracker-task/InternalController`. Если понадобится добавить rate limiting или изменить логику проверки, придётся менять два контроллера в двух разных сервисах.

**Рекомендация:** вынести в `HandlerInterceptor`:

```java
@Component
@RequiredArgsConstructor
public class InternalApiKeyInterceptor implements HandlerInterceptor {
    private final InternalProperties internalProperties;

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String key = request.getHeader("X-Internal-Api-Key");
        if (!internalProperties.getApiKey().equals(key)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            return false;
        }
        return true;
    }
}

@Configuration
@RequiredArgsConstructor
public class WebConfig implements WebMvcConfigurer {
    private final InternalApiKeyInterceptor interceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(interceptor)
                .addPathPatterns("/internal/**");
    }
}
```

**13. `UserController` принимает `X-User-Id` как `String` и вручную конвертирует в `Long`**

`Long.valueOf(userId)` бросит `NumberFormatException`, если заголовок содержит нечисловое значение. Это исключение не обрабатывается в `GlobalExceptionHandler` и приведёт к неструктурированному 500-ответу. Spring выполняет конвертацию типов заголовков автоматически.

**Рекомендация:** объявить параметр как `Long`

---

### [tracker-auth] пакет /dto

**14. Отсутствует разделение DTO на request/response — в названиях и пакетах**

В пакете `dto` все классы лежат вместе без разделения по назначению. Ответные DTO называются `UserDto` вместо `UserResponseDto`, хотя суффикс `Dto` не говорит о направлении передачи данных. При росте числа эндпоинтов это приводит к путанице: непонятно, какой класс входящий, а какой исходящий.

**Рекомендация:** разбить пакет на подпакеты и переименовать классы:

```
dto/
  request/
    AuthUserRequest.java
    RegisterUserRequest.java
  response/
    UserResponseDto.java     ← вместо UserDto
    ErrorResponseDto.java    ← вместо ErrorMessage
```

**15. `AuthUserRequest.password` не имеет аннотации `@NotBlank`**

По спецификации Bean Validation, `@Size` пропускает `null`-значения. Если клиент отправит запрос без поля `password` (или явно с `null`), валидация не сработает. `passwordEncoder.encode(null)` бросит `IllegalArgumentException` внутри `AuthService.register()`, которое не обрабатывается в `GlobalExceptionHandler` → 500-ответ.

**Рекомендация:** Добавить `@NotBlank` на поле password

**16. `@Data` на DTO `AuthUserRequest` вместо record**

`@Data` генерирует изменяемый класс с сеттерами. DTO для входящего запроса не должен быть изменяемым: он создаётся при десериализации и используется только для чтения. Открытые сеттеры создают риск случайной мутации объекта внутри сервиса. 
В `tracker-task` ты уже используешь records для DTO — здесь стоит поступить так же.

**Рекомендация:** использовать record вместо `@Data`

---

### [tracker-auth] пакет /entities

**17. `User implements UserDetails` — слияние persistence-слоя и security-слоя**

JPA-сущность напрямую реализует Spring Security интерфейс `UserDetails`. Это жёсткая связь между доменным объектом и инфраструктурой безопасности: сущность вынуждена реализовывать методы (`getAuthorities()`, `isAccountNonLocked()` и др.), не имеющие отношения к домену пользователя. 
При изменении структуры `User` нужно учитывать влияние на security-контракт. Тестирование security-логики требует JPA-контекста. Кроме того, в `SecurityConfig` стоит `anyRequest().permitAll()` — Spring Security в `tracker-auth` фактически не используется для аутентификации, что делает реализацию `UserDetails` бессмысленной: `JwtTokenService.generateToken(User)` мог бы вызывать `user.getEmail()` напрямую без интерфейса.

**Рекомендация:** создать отдельный адаптер

---

### [tracker-auth] пакет /services

**18. `AuthService.register()` использует `jakarta.transaction.Transactional` вместо Spring-аналога**

В проектах на Spring Boot следует использовать `org.springframework.transaction.annotation.Transactional`. Jakarta-версия не поддерживает Spring-специфичные атрибуты `propagation`, `rollbackFor`, `noRollbackFor` в совместимом с AOP виде. В частности, дефолтное поведение rollback при unchecked-исключениях одинаково, но при использовании `propagation = REQUIRES_NEW` или кастомных `rollbackFor` поведение может отличаться.

**Рекомендация:** Использовать Transactional из спринга

**19. Отправка Kafka-сообщения внутри транзакции БД**

`KafkaTemplate.send()` асинхронный по умолчанию — он возвращает `CompletableFuture` и не ждёт подтверждения брокера. Транзакция БД коммитится независимо от статуса Kafka. 
Если Kafka недоступна в момент регистрации, пользователь сохраняется в БД, но приветственное письмо не отправляется — без какой-либо ошибки для клиента. Если нужна гарантия «либо оба действия произошли, либо ни одно», нужна транзакционная координация.

**Рекомендация:** отправлять Kafka-сообщение после коммита транзакции через `@TransactionalEventListener` или использовать паттерн outbox:

```java
// AuthService:
@Transactional
public String register(AuthUserRequest authUserRequest) {
    User user = new User();
    user.setEmail(authUserRequest.getEmail());
    user.setPassword(passwordEncoder.encode(authUserRequest.getPassword()));
    try {
        User saved = userRepository.save(user);
        eventPublisher.publishEvent(new UserRegisteredEvent(saved.getEmail()));
        return jwtTokenService.generateToken(saved);
    } catch (DataIntegrityViolationException e) {
        throw new EmailAlreadyExistsException(ApiErrorMessages.EMAIL_ALREADY_EXISTS.getMessage());
    }
}

// Отдельный listener:
@Component
@RequiredArgsConstructor
public class UserRegistrationEventListener {
    private final KafkaSenderService kafkaSenderService;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onUserRegistered(UserRegisteredEvent event) {
        kafkaSenderService.sendMessageToKafka(new EmailLetterModel(
            event.getEmail(),
            NewUserEmailTemplate.TITLE,
            NewUserEmailTemplate.DESCRIPTION
        ));
    }
}
```

---

### [tracker-task] пакет /config (отсутствует)

**20. `tracker-task` не управляет своей схемой БД — миграции вынесены в `tracker-auth`**

Файлы: `tracker-auth/src/main/resources/db/migration/V1__init.sql`, `tracker-task/src/main/resources/application.yaml`.

`V1__init.sql` в `tracker-auth` создаёт не только таблицу `users`, но и таблицу `tasks` с FK `REFERENCES users(id)`. 
`tracker-task` не имеет ни Flyway, ни Liquibase. Это нарушение фундаментального принципа микросервисов — Database-per-Service.

Почему это критично:
- FK `REFERENCES users(id)` создаёт зависимость на уровне БД между двумя сервисами, которые должны быть физически независимы.
- `tracker-task` нельзя запустить до выполнения миграций в `tracker-auth` — скрытая зависимость деплоя.
- Масштабирование или разделение баз данных в будущем потребует переписывания схемы обоих сервисов.
- `tracker-auth` знает о внутренней структуре `tracker-task` — это прямое нарушение инкапсуляции на уровне сервисов.

**Рекомендация:** перенести управление схемой `tasks` в `tracker-task`. Добавить Flyway в зависимости `tracker-task` и создать собственный миграционный файл. 
Ссылочная целостность в микросервисах обеспечивается на уровне приложения (проверка userId через HTTP-вызов к auth-сервису при необходимости), а не через FK в БД.

---

### [tracker-task] пакет /controllers

**21. `InternalController` в `tracker-task` — дублирование антипаттерна из `tracker-auth`**

Та же проблема что в замечании 12: логика проверки API-ключа в контроллере, `@Value` в контроллере. Дублирование этого антипаттерна в двух сервисах показывает, что решение было скопировано вместо создания переиспользуемой абстракции.

**Рекомендация:** применить `HandlerInterceptor`-подход, описанный в замечании 12.

---

### [tracker-task] пакет /dto

**22. `TaskDto` не имеет суффикса `Response` — назначение класса неочевидно**

`TaskDto` является ответным объектом, возвращаемым клиенту. Суффикс `Dto` не указывает на направление: это входящий или исходящий объект? `TaskRequest` и `TaskUpdateRequest` уже используют суффикс `Request` — логично, чтобы ответный DTO тоже явно указывал своё назначение.

**Рекомендация:** переименовать и структурировать пакет аналогично tracker-auth:

```
dto/
  request/
    TaskRequest.java        ← без изменений
    TaskUpdateRequest.java  ← без изменений
  response/
    TaskResponse.java       ← вместо TaskDto
```

### [tracker-task] пакет /entities

**23. Несогласованные и нестандартные значения `TaskStatus`**

```java
public enum TaskStatus {
    ONGOING,    // прилагательное
    COMPLETED   // причастие прошедшего времени
}
```

Два нарушения. Во-первых, `ONGOING` — нестандартное слово для статуса задачи: оно не встречается в распространённых трекерах и API. Во-вторых, `ONGOING` и `COMPLETED` написаны в разном грамматическом стиле — первое является прилагательным, второе причастием. Имена значений enum должны быть единообразны.

**Рекомендация:** привести к одному стилю. Наиболее распространённый вариант для задач:

```java
public enum TaskStatus {
    IN_PROGRESS,
    COMPLETED
}
```

Или, если нужен более широкий набор статусов:

```java
public enum TaskStatus {
    OPEN,
    IN_PROGRESS,
    COMPLETED,
    CANCELLED
}
```

**24. `@Builder` на JPA-сущности `Task`**

`@Builder` на JPA-сущности — антипаттерн. Он провоцирует создание объектов через `Task.builder()...build()` напрямую в сервисном слое, смешивая ответственность создания объекта с бизнес-логикой. `@Builder` требует `@AllArgsConstructor`, что открывает возможность конструировать объект со всеми полями вручную — включая `id`, `createdAt`, `updatedAt`, которые должны задаваться только БД/Hibernate. 
В данном коде `TaskMapper` корректно игнорирует эти поля через `@Mapping(target = "...", ignore = true)`, но `@Builder` остаётся лишним — никто извне маппера не должен создавать `Task` напрямую.

**Рекомендация:** убрать `@Builder` и `@AllArgsConstructor`. Создание объектов — через MapStruct, как уже сделано.

**25. `@Column(unique = true)` на поле `title` противоречит реальному DB-ограничению**

Эта аннотация объявляет глобальный UNIQUE constraint на колонку `title` по всем пользователям. Реальная схема в `V1__init.sql` имеет составное ограничение `UNIQUE (user_id, title)` — уникальность в рамках одного пользователя. С `ddl-auto: none` реальная схема не трогается, но код вводит в заблуждение: читатель предполагает, что заголовок задачи уникален глобально.

**Рекомендация:** отразить реальное ограничение через `@Table(name = "tasks",
       uniqueConstraints = @UniqueConstraint(
           name = "uq_tasks_user_title",
           columnNames = {"user_id", "title"}
       ))`

---

### [tracker-task] пакет /services

**26. `TaskService.createTask()` — TOCTOU race condition и необработанный `DataIntegrityViolationException`**

Между проверкой наличия и сохранением нет атомарности. Два одновременных запроса могут оба пройти проверку и оба попытаться сохранить задачу с одинаковым title. 
БД выбросит `DataIntegrityViolationException` (нарушение UNIQUE), которое не перехватывается в `createTask()` — пользователь получит 500. Метод также не помечен `@Transactional`, хотя выполняет операцию записи.

**Рекомендация:** убрать SELECT, использовать constraint как единственный источник истины, перехватить `DataIntegrityViolationException`:

```java
@Transactional
public void createTask(TaskRequest taskRequest, Long userId) {
    Task task = taskMapper.toEntity(taskRequest);
    task.setUserId(userId);
    try {
        taskRepository.save(task);
        log.info("Creating task '{}' for user {}", taskRequest.title(), userId);
    } catch (DataIntegrityViolationException e) {
        throw new TaskAlreadyExistsException(ApiMessages.TASK_ALREADY_EXISTS.getMessage());
    }
}
```

Это одновременно убирает лишний SELECT перед INSERT и устраняет race condition.

**27. Дублирование stream-кода в `TaskService.getAllTasks()`**

`.stream().map(taskMapper::toDto).toList()` дублируется. Единственное различие — вызов репозитория.

**Рекомендация:**

```java
public List<TaskDto> getAllTasks(Long userId, TaskStatus status) {
    List<Task> tasks = (status != null)
            ? taskRepository.findAllByUserIdAndStatus(userId, status)
            : taskRepository.findAllByUserId(userId);

    log.info("Getting tasks for user {}, status filter: {}", userId, status);
    return tasks.stream().map(taskMapper::toDto).toList();
}
```

---

### [tracker-scheduler] пакет /config

**28. `RestClientConfig` создаёт `RestClient` без таймаутов и базового URL**

`RestClient.create()` создаёт клиент без каких-либо настроек таймаутов. HTTP-запросы в `TaskSchedulerService` к auth и task сервисам не имеют connect/read timeout. Если один из сервисов завис, scheduled-поток будет заблокирован indefinitely. Spring Scheduler по умолчанию использует один поток — зависший HTTP-вызов заблокирует все последующие scheduled задачи в приложении.

**Рекомендация:** Настроить таймауты

**29. Три `@Value` в `TaskSchedulerService` — конфигурация в бизнес-сервисе**

Та же проблема что в замечании 7. Сервис, содержащий бизнес-логику планировщика, напрямую читает строковые ключи из properties. При переименовании ключей или добавлении нового сервиса нужно лезть внутрь бизнес-класса.

**Рекомендация:** Использовать ConfigurationProperties

---

### [tracker-scheduler] пакет /model

**30. Пакет называется `model` вместо `dto`**

`model` — размытое название, которое не говорит о назначении классов. Классы в этом пакете (`Task`, `User`, `EmailLetterModel`) — это объекты передачи данных, используемые для HTTP-запросов и Kafka-сообщений, то есть DTO. Название `model` традиционно закреплено за доменными сущностями или JPA-сущностями, а не за DTO.

**Рекомендация:** переименовать пакет в `dto` и разделить на http/kafka 

---

### [tracker-scheduler] пакет /service

**31. Cron-выражение захардкожено в аннотации — изменение расписания требует перебилда**

Если понадобится изменить время запуска (например, перенести отчёт с полуночи на 7 утра или настроить разное расписание для разных окружений), придётся менять код, пересобирать и переразворачивать приложение. Расписание — это конфигурация, а не бизнес-логика.

**Рекомендация:** вынести cron в `application.yaml` и ссылаться на него через placeholder:

```java
@Scheduled(cron = "${scheduler.daily-report.cron}")
public void sendDailyReport() { ... }
```

```yaml
scheduler:
  daily-report:
    cron: "0 0 0 * * *"
```

Теперь для изменения расписания достаточно поменять переменную окружения или значение в `application.yaml` без перебилда.

**32. `sendDailyReport()` — N+1 HTTP-запросов**

При N пользователях выполняется N+1 последовательных синхронных HTTP-запросов в одном потоке: один запрос за списком всех пользователей, затем по одному запросу за задачами каждого. Это симптом архитектурной проблемы: планировщик берёт на себя агрегацию данных, которая должна происходить на стороне `tracker-task`.

**Рекомендация:** добавить в `tracker-task` один эндпоинт, который сам агрегирует нужные данные за сегодня:

```java
// GET /internal/tasks/daily-summary
// tracker-task возвращает список { userEmail, completedTasks } за текущий день
List<UserDailySummary> getDailySummary();
```

Тогда планировщик делает **один HTTP-запрос** и просто рассылает письма — никакого цикла, никакого N+1:

```java
@Scheduled(cron = "${scheduler.daily-report.cron}")
public void sendDailyReport() {
    taskServiceClient.getDailySummary()
        .stream()
        .filter(summary -> !summary.tasks().isEmpty())
        .forEach(summary -> kafkaSenderService.sendMessageToKafka(
            new EmailLetterModel(summary.userEmail(), TITLE_TEXT_LETTER, buildBody(summary.tasks()))
        ));
}
```

**33. Баг с логированием `Optional` в `sendDailyReport()`**

`Optional.toString()` возвращает строку вида `"Optional[actual content]"` или `"Optional.empty"` — в лог попадает обёртка, а не содержимое. Кроме того, передача строки напрямую в `log.info()` без плейсхолдера `{}` может привести к некорректному поведению, если строка содержит символы форматирования.

**Рекомендация:**

```java
log.info("Email body generated: {}", body.orElse("empty — skipping user"));
```

**34. `getStartOfDay()` использует системный часовой пояс JVM**

`LocalDateTime.now()` использует системный timezone JVM. В Docker-контейнере это обычно UTC. Если `updated_at` в БД хранится в UTC, а граница дня вычисляется в другом timezone, задачи, выполненные в конце или начале дня, попадут не в тот отчёт.

**Рекомендация:** явно указывать UTC

**35. Дублирование модельных классов между сервисами**

Файлы: `tracker-scheduler/model/{Task,User,TaskStatus,EmailLetterModel}.java`, `tracker-auth/dto/EmailLetterModel.java`, `tracker-email-sender/models/EmailLetterModel.java`.

`EmailLetterModel` определён в трёх сервисах (`tracker-auth`, `tracker-scheduler`, `tracker-email-sender`). `Task`, `User`, `TaskStatus` дублируются в `tracker-scheduler`. При добавлении нового поля в `EmailLetterModel` нужно синхронно менять три класса в трёх репозиториях — нарушение DRY с реальным риском рассинхронизации контрактов.

**Рекомендация:** вынести общие модели в Gradle subproject `tracker-common` и подключить его во все сервисы как зависимость

---

### [tracker-email-sender] пакет /config

**36. `EmailSenderConfig` создаёт неиспользуемый бин `MailtrapClient`**

`MailtrapClient` создаётся как бин, но нигде не инжектируется и не используется. `GmailEmailSenderServiceImpl` использует `JavaMailSender` — Spring Mail, работающий через `spring.mail.*` с Gmail SMTP. `MailProperties mailProperties` инжектируется в конфиг, но также не применяется. Это мёртвый код, создающий путаницу: непонятно, какой email-провайдер реально работает.

**Рекомендация:** удалить `EmailSenderConfig` полностью. `JavaMailSender` создаётся Spring Boot автоматически через `spring.mail.*`. Также удалить `email.sender.token` из `application.yaml` и соответствующую переменную окружения. Переименовать `GmailEmailSenderServiceImpl` в `SmtpEmailSenderServiceImpl`, поскольку `JavaMailSender` абстрагирует конкретный SMTP-провайдер.

**37. `.env` файл закоммичен в публичный репозиторий**

Файл: `tracker-email-sender/.env`.

Файл `.env` попал в git-репозиторий. Если он содержит реальные credentials (пароль Gmail, токен Mailtrap), они скомпрометированы.

**Рекомендация:** добавить `.env` в `.gitignore`, удалить из истории git (`git rm --cached .env && git commit`), сменить все credentials из файла. Для документирования необходимых переменных использовать `.env.example` с заглушками:

```bash
MAIL_PASSWORD=your_mail_password_here
MAIL_USERNAME=your_email@gmail.com
MAIL_PORT=587
KAFKA_HOST=kafka:9092
KAFKA_TOPIC_NAME=EMAIL_SENDING_TASKS
```

**38. Несогласованный базовый пакет `org.roadmap` в `tracker-email-sender`**

Файлы: все классы в `tracker-email-sender`.

Все остальные сервисы используют `org.example.*`. `tracker-email-sender` использует `org.roadmap.*`. Это нарушает единообразие именования в проекте.

**Рекомендация:** унифицировать

---

### [tracker-email-sender] пакет /services

**39. Неправильное логирование стектрейса в `GmailEmailSenderServiceImpl`**

`e.getStackTrace()` возвращает `StackTraceElement[]`. SLF4J вызовет `.toString()` на массиве, что даст нечитаемую строку вида `[Ljava.lang.StackTraceElement;@1a2b3c4`. Стектрейс в лог не попадёт. Стандартный способ логировать исключение в SLF4J — передать его последним аргументом без плейсхолдера.

**Рекомендация:**

```java
} catch (MailException e) {
    log.error("Failed to send email to {}: {}", emailLetterModel.email(), e.getMessage(), e);
}
```

**40. Конкурентность Kafka Consumer жёстко задана константой в коде**

Количество consumer-потоков задано строковой константой в коде. Изменить его без перекомпиляции и редеплоя невозможно. Spring поддерживает property placeholders в атрибутах `@KafkaListener`.

**Рекомендация:**

```java
@KafkaListener(topics = "${kafka.topic}", concurrency = "${kafka.consumer.concurrency:5}")
public void listen(EmailLetterModel emailLetterModel) { ... }
```

```yaml
kafka:
  consumer:
    concurrency: 5
```

---

## РЕКОМЕНДАЦИИ

1. **Ввести `@ConfigurationProperties` как единственный способ работы с конфигурацией во всех сервисах.** Заменить все `@Value`-поля на типизированные `@ConfigurationProperties`-классы в пакете `/config`. Это даёт валидацию при старте (`@Validated` + Bean Validation), единую точку просмотра конфигурации модуля, IDE-автодополнение и устраняет дублирование строковых ключей.

2. **Реализовать Database-per-Service.** Вынести схему таблицы `tasks` и её миграции из `tracker-auth` в `tracker-task`. Убрать FK `REFERENCES users(id)` — ссылочная целостность в микросервисах обеспечивается на уровне приложения, не БД. Добавить Flyway в `tracker-task`. Это устранит скрытую зависимость деплоя и позволит сервисам эволюционировать независимо.

3. **Вынести проверку `X-Internal-Api-Key` в `HandlerInterceptor`.** Дублирующаяся логика проверки API-ключа в контроллерах двух сервисов должна быть заменена единым интерсептором. Контроллеры не должны знать о деталях авторизации.

4. **Создать Gradle subproject `tracker-common` для общих DTO.** `EmailLetterModel`, дублирующийся в трёх сервисах, — минимальная единица, с которой нужно начать. Единый контракт, управляемый в одном месте, устранит риск рассинхронизации при изменении структуры сообщений Kafka.

5. **Отделить JPA-сущность `User` от `UserDetails`.** Создать отдельный security-адаптер `AuthUserDetails implements UserDetails`. Сущность должна описывать только домен пользователя; security-интерфейс — отдельная ответственность.

6. **Вынести агрегацию данных из `tracker-scheduler` в `tracker-task`.** Вместо N+1 HTTP-запросов за каждым пользователем добавить в `tracker-task` эндпоинт `/internal/tasks/daily-summary`, который сам агрегирует данные за день. Планировщик делает один запрос и рассылает письма. Дополнительно настроить connect/read timeout на `RestClient` и вынести cron-выражение в `application.yaml`.

7. **Написать интеграционные тесты с Testcontainers.** Во всех сервисах — только пустые test-классы. Минимально необходимо покрыть: регистрацию и логин в `tracker-auth` с реальным PostgreSQL, создание/обновление/удаление задач в `tracker-task`, Kafka-консьюмер в `tracker-email-sender` с embedded Kafka. Без тестов любой рефакторинг из этого ревью выполняется вслепую.

8. **Переименовать пакеты во всех сервисах в единственное число.** `services` → `service`, `controllers` → `controller`, `entities` → `entity` во всех модулях проекта. Это базовое требование Java Naming Conventions.

9. **Ввести интерфейсы для всех сервисов и контроллеров.** Сервисы без интерфейса жёстко привязывают потребителей к конкретной реализации. Контроллеры без интерфейса не поддерживают версионирование API. Исключение — `tracker-email-sender`, где паттерн уже применён правильно.

---

## ИТОГ

Проект демонстрирует понимание микросервисной топологии: сервисы разделены по ответственности, JWT-аутентификация централизована в gateway, Kafka применяется для асинхронной интеграции, MapStruct используется для маппинга с корректной обработкой частичных обновлений.

Главная архитектурная проблема — нарушение Database-per-Service: `tracker-auth` управляет схемой `tracker-task` через миграции и создаёт FK между таблицами разных доменов. Это отрицает независимость, которую должна обеспечивать микросервисная архитектура. Параллельно — конфигурация через `@Value` разбросана по всем слоям во всех сервисах вместо `@ConfigurationProperties`, что делает конфигурацию непрозрачной и ненадёжной. Вторая архитектурная проблема — `tracker-scheduler` берёт на себя агрегацию данных вместо того, чтобы делегировать её `tracker-task`: N+1 синхронных HTTP-запросов решаются одним специализированным эндпоинтом на стороне task-сервиса.

На уровне кода выделяются три критичных момента:
- мёртвый код `UsernameNotFoundException` в `@ExceptionHandler` (Spring Security перехватывает его раньше, до handler-а он никогда не доходит)
- необработанный `DataIntegrityViolationException` при конкурентном создании задачи
- и `.env` с чувствительными данными в публичном репозитории. 
Отдельно — Spring Security без настроенного `AuthenticationEntryPoint` возвращает 403 вместо 401, что фронтенд молча трактует как неверный логин/пароль.

В части конвенций и именования: пакеты во всех сервисах названы во множественном числе вместо единственного; DTO не разделены на `request`/`response` подпакеты; `TaskDto` и `UserDto` не отражают своё назначение в имени; `TaskStatus` содержит несогласованные по стилю значения (`ONGOING` vs `COMPLETED`); ни один сервис кроме `tracker-email-sender` не использует интерфейсы для сервисного и контроллерного слоёв.

Ключевые направления для роста: освоить `@ConfigurationProperties` как стандарт работы с конфигурацией в Spring; разобраться с паттерном Database-per-Service и ссылочной целостностью в микросервисах без FK на уровне БД; научиться покрывать сервисный слой интеграционными тестами с Testcontainers; изучить outbox паттерн для надёжной координации транзакции БД с Kafka; выработать привычку выносить любую конфигурацию (cron, таймауты, топики) в `application.yaml`.
