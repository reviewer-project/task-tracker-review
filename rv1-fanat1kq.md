[fanat1kq](https://github.com/fanat1kq/task-tracker)

# НЕДОСТАТКИ РЕАЛИЗАЦИИ

## 1. Безопасность

### 1.1. Gateway не выполняет свою основную функцию (MEDIUM)
```java
// SecurityConfig.java (gateway)
.authorizeExchange(exchanges -> exchanges
    .pathMatchers("/api/auth/**").permitAll()
    .pathMatchers("/oauth2/**", "/login/oauth2/**").permitAll()
    .pathMatchers("/api/public/**").permitAll()
    .pathMatchers("/api/**").permitAll()
    .anyExchange().authenticated()
)
```
Gateway пропускает все запросы без проверки, а авторизация выполняется на уровне каждого микросервиса:
```java
// task-service SecurityConfig.java
.authorizeHttpRequests(authz -> authz.anyRequest().authenticated())
.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
```

**Проблема:** В текущей реализации Gateway работает как простой reverse proxy и не выполняет свою основную функцию — централизованную авторизацию. Возникает вопрос: зачем нужен Spring Cloud Gateway, если с этой задачей справится nginx?

**Преимущества централизованной авторизации в Gateway:**
- Единая точка проверки токенов (DRY principle)
- Микросервисы не нужно настраивать на работу с JWT
- Проще добавлять новые сервисы
- Меньше нагрузки на auth-service (JWK endpoint)
- Можно передавать userId в заголовке, убрав дублирование логики

**Рекомендация:** Перенести авторизацию в Gateway:
```java
// Gateway SecurityConfig.java
.authorizeExchange(exchanges -> exchanges
    .pathMatchers("/api/auth/login", "/api/auth/register").permitAll()
    .pathMatchers("/api/**").authenticated()
)
.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
```
И передавать userId в заголовке для микросервисов:
```java
// Gateway filter
.filter((exchange, chain) -> {
    String userId = exchange.getPrincipal()
        .map(p -> ((Jwt) p).getSubject()).block();
    return chain.filter(exchange.mutate()
        .request(r -> r.header("X-User-Id", userId))
        .build());
})
```
Тогда микросервисы могут доверять заголовку от Gateway и не проверять JWT повторно.

### 1.2. RSA ключи генерируются при каждом запуске (HIGH)
```java
// SecurityConfig.java (auth-service)
private static KeyPair generateRsaKey() throws NoSuchAlgorithmException {
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
    keyPairGenerator.initialize(2048);
    return keyPairGenerator.generateKeyPair();
}
```
При каждом перезапуске auth-service генерируются новые RSA ключи. Это означает:
- Все ранее выданные JWT токены становятся невалидными
- Пользователи будут разлогинены после каждого деплоя

**Рекомендация:** Хранить ключи в файле или использовать Key Management Service.

### 1.3. Hardcoded client secret (MEDIUM)
```java
// SecurityConfig.java (auth-service)
.clientSecret("{noop}gateway-secret")
```
Client secret для OAuth2 клиента захардкожен в коде с `{noop}` (без шифрования).

**Рекомендация:** Вынести в конфигурацию и использовать BCrypt для хранения.

### 1.4. Endpoint /api/auth/users возвращает всех пользователей (MEDIUM)
```java
// AuthController.java
@GetMapping("/users")
public List<UserDto> getAllUsers() {
    return userService.getAllUsers();
}
```
Endpoint доступен без авторизации (см. 1.2) и возвращает список всех пользователей. Это потенциальная утечка данных.

**Рекомендация:** Защитить endpoint и ограничить доступ только для администраторов или внутренних сервисов.

### 1.5. Ошибки валидации возвращают Internal Server Error (HIGH)
```java
// AuthControllerAdvice.java
import org.springframework.messaging.handler.annotation.support.MethodArgumentNotValidException;
// Неправильный импорт! Должен быть:
// import org.springframework.web.bind.MethodArgumentNotValidException;

@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationErrors(
    MethodArgumentNotValidException ex) {
    // Этот обработчик никогда не вызывается для REST валидации
}
```
Импортирован `MethodArgumentNotValidException` из пакета `org.springframework.messaging` вместо `org.springframework.web.bind`. Из-за этого ошибки валидации (например, короткий username) не перехватываются и падают в generic handler, возвращая "Internal Server Error" вместо понятного сообщения.

**Рекомендация:** Исправить импорт и добавить детализацию ошибок:
```java
import org.springframework.web.bind.MethodArgumentNotValidException;

@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidationErrors(
    MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .collect(Collectors.joining(", "));
    
    return ResponseEntity.badRequest().body(
        new ErrorResponse("VALIDATION_FAILED", message, 400));
}
```

### 1.6. RuntimeException вместо кастомных исключений (MEDIUM)
```java
// UserService.java
User user = userRepository.findUserByUsername(request.username())
    .orElseThrow(() -> new RuntimeException("User not found"));

// TaskService.java
Task task = taskRepository.findById(taskId).orElseThrow(
    () -> new RuntimeException("Task not found with id: " + taskId));
```
Использование `RuntimeException` раскрывает внутреннюю информацию и не позволяет корректно обрабатывать ошибки.

**Рекомендация:** Использовать кастомные исключения (`UserNotFoundException`, `TaskNotFoundException`) с глобальным обработчиком.

## 2. Логические ошибки

### 2.1. Отсутствует проверка владельца задачи при операциях (HIGH)
```java
// TaskController.java
@DeleteMapping("/{taskId}")
public void deleteTask(@PathVariable Long taskId) {
    taskService.deleteTask(taskId);
}

@PutMapping("/{taskId}")
public void updateStatusById(@PathVariable Long taskId,
                             @RequestBody TaskUpdateRequest updateRequest) {
    taskService.updateTask(taskId, updateRequest);
}
```
Нет проверки, что текущий пользователь является владельцем задачи. Любой авторизованный пользователь может удалить или изменить чужую задачу.

**Рекомендация:** Добавить проверку `task.getUserId().equals(currentUserId)` перед операциями.

### 2.2. getTasks() возвращает ВСЕ задачи (HIGH)
```java
// TaskService.java
@Transactional(readOnly = true)
public List<TaskDto> getTasks() {
    return taskMapper.toDtoList(taskRepository.findAll());
}
```
Метод возвращает все задачи из базы данных, а фильтрация происходит на фронтенде:
```javascript
// App.jsx
const userTasks = allTasks.filter(task => {
    const isUserTask = task.userId === currentUserId;
    return isUserTask;
});
```
Это неэффективно и небезопасно — пользователь получает доступ ко всем задачам.

**Рекомендация:** Фильтровать задачи по userId на бэкенде и добавить пагинацию:
```java
public Page<TaskDto> getTasks(Long userId, Pageable pageable) {
    return taskRepository.findByUserId(userId, pageable)
        .map(taskMapper::toDto);
}
```

### 2.3. Несоответствие типа userId в Entity (MEDIUM)
```java
// Task.java
@Column(nullable = false, columnDefinition = "UUID")
private Long userId;
```
Поле `userId` объявлено как `Long`, но в columnDefinition указан `UUID`. Это несоответствие типов.

**Рекомендация:** Использовать либо `Long` с `BIGINT`, либо `UUID` с соответствующим типом Java.

### 2.4. @PostMapping на методе сервиса (LOW)
```java
// TaskService.java
@PostMapping("/task/info")  // Аннотация контроллера на методе сервиса!
public void getAllTasksInfo(Long userId) {
    var dto = new UserInformationDto(userId);
    dataPublisher.sendUserInfo(dto);
}
```
Аннотация `@PostMapping` на методе сервиса не имеет эффекта и вводит в заблуждение.

**Рекомендация:** Удалить аннотацию.

### 2.5. Hardcoded IP в email шаблоне (MEDIUM)
```html
<!-- welcome.html -->
<a href="http://176.109.106.218:3001/" class="cta-button" target="_blank">
    Get Started Now
</a>
```
IP адрес захардкожен в шаблоне письма.

**Рекомендация:** Использовать переменную из конфигурации.

### 2.6. Hardcoded роль ADMIN для всех пользователей (MEDIUM)
```java
// UserMapper.java
@Mapping(target = "role", constant = "ADMIN")
AuthResponse toAuthResponse(String jwtToken, User user);
```
Всем пользователям присваивается роль ADMIN при логине.

**Рекомендация:** Хранить роли в БД и получать из сущности User.

## 3. Архитектурные проблемы

### 3.1. Отсутствует валидация входных данных в task-service (HIGH)
```java
// TaskRequestDto.java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class TaskRequestDto {
    private String title;
    private Long userId;
    private TaskStatus status;
}
```
Нет валидации (`@NotBlank`, `@NotNull`, `@Size`) для полей DTO. Можно создать задачу с пустым заголовком или без userId.

**Рекомендация:** Добавить валидацию:
```java
@NotBlank(message = "Title is required")
@Size(max = 255)
private String title;

@NotNull(message = "User ID is required")
private Long userId;
```

### 3.2. Нет @Validated на TaskController (MEDIUM)
```java
// TaskController.java
@RestController
@RequestMapping("/api/tasks")
@RequiredArgsConstructor
public class TaskController {
```
В отличие от `AuthController`, здесь нет `@Validated`, поэтому даже если добавить валидацию в DTO, она не будет работать.


## 4. Проблемы с Kafka

### 4.1. Генерация messageId через hashCode (MEDIUM)
```java
// DataPublisher.java
private Long generateNotificationId(Notification dto) {
    return (long) dto.hashCode();
}
```
`hashCode()` может давать коллизии, что приведет к потере сообщений при дедупликации.

**Рекомендация:** Использовать UUID или последовательный ID из базы данных.

### 4.2. Retry логика дублируется на producer и consumer (MEDIUM)

Retry реализован **дважды** — и на стороне отправителя, и на стороне получателя:

**1. Producer-side retry (task-service):**
```java
// DataPublisher.java
if (!retryService.shouldRetry(messageId, topic, dto)) {
    log.error("Max retries exceeded: {}, sending to DLQ immediately", messageId);
    errorHandlerService.handlePermanentFailure(topic, dto);
    return;
}

publisherService.send(topic, dto)
    .whenComplete((result, throwable) -> {
        if (throwable != null) {
            handleSendFailure(topic, dto, throwable, messageId);  // Retry с backoff
        }
    });

// RetryService.java — хранит счётчик retry в БД (FailedMessage)
public Duration calculateBackoffDelay(int retryCount) {
    double delaySeconds = retryProperties.initialDelay().getSeconds() *
        Math.pow(retryProperties.multiplier(), (double) retryCount - 1);
    return Duration.ofSeconds((long) Math.min(delaySeconds, maxDelaySeconds));
}
```

**2. Consumer-side retry (notification-service):**
```java
// TaskNotificationConsumer.java
@RetryableTopic(
    attempts = "6",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 30000)
)
@KafkaListener(topics = "${kafka.topics.email-sending}")
public void consume(ConsumerRecord<String, String> emailRecord, Acknowledgment ack) {
    // Spring Kafka автоматически создаёт retry topics:
    // email-sending-topic-retry-0, retry-1, ..., -dlt
}
```

**Проблемы такого подхода:**

| Аспект | Проблема |
|--------|----------|
| Количество попыток | Непонятно сколько: 3 (producer) × 6 (consumer) = до 18 попыток? |
| Хранение failed messages | Два места: таблица `failed_messages` (producer) + Kafka DLT topics (consumer) |
| Backoff настройки | Разные: producer (5s-1m), consumer (1s-30s) |
| Отладка | Сложно понять, где застряло сообщение |
| Мониторинг | Нужно следить за двумя системами |

**Когда какой подход использовать:**

| Producer-side retry | Consumer-side retry (`@RetryableTopic`) |
|---------------------|----------------------------------------|
| Kafka недоступна | Kafka работает, проблема в обработке |
| Сетевые проблемы | Временные ошибки (БД, внешний API) |
| Гарантия доставки в брокер | Гарантия успешной обработки |

**Рекомендация:** В данном случае достаточно **только consumer-side retry** через `@RetryableTopic`:
- Kafka в Docker Compose надёжна (локальная сеть)
- Проблемы скорее в обработке (отправка email, БД)
- `@RetryableTopic` — стандартный, хорошо документированный подход
- Меньше кода, проще поддержка

Producer-side retry имеет смысл оставить только для критичных сценариев, когда Kafka может быть недоступна (например, при деплое).

## 5. Проблемы с Docker

### 5.1. Frontend не использует nginx для production (MEDIUM)
```dockerfile
# Dockerfile (tracker-front)
RUN npm install -g serve
CMD ["serve", "-s", "dist", "-l", "3001"]
```
Для production лучше использовать nginx (файл `nginx.conf` есть, но не используется).

**Рекомендация:** Использовать multi-stage build с nginx:
```dockerfile
FROM node:18-alpine AS build
WORKDIR /app
COPY . .
RUN npm install && npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
```

### 5.2. Нет healthcheck для сервисов (MEDIUM)
В docker-compose только postgres имеет healthcheck. Остальные сервисы могут запускаться до готовности зависимостей.

**Рекомендация:** Добавить healthcheck и depends_on с condition.

### 5.3. Нет ограничений ресурсов (LOW)
```yaml
services:
  task-service:
    # Нет deploy.resources.limits
```
Для production рекомендуется ограничивать memory/cpu.

---

# ХОРОШО

## 1. Архитектура

### 1.1. Использование Outbox Pattern с Debezium
```java
// OutboxService.java
@Transactional
public void createEvent(String eventType, Object payload) {
    outboxEventRepository.save(OutboxEvent.builder()
        .eventType(eventType)
        .payload(payloadJson)
        .deduplicationKey(String.valueOf(UUID.randomUUID()))
        .build());
}
```
Отличный паттерн для гарантированной доставки событий. Транзакционная запись в outbox таблицу + CDC через Debezium обеспечивает at-least-once delivery.

### 1.2. Дедупликация сообщений на consumer side
```java
// DeduplicationService.java
@Transactional
public void executeWithDeduplication(String deduplicationKey, Runnable businessLogic) {
    if (!tryInsertRecord(deduplicationKey)) {
        throw new DuplicateMessageException(deduplicationKey);
    }
    try {
        businessLogic.run();
    } catch (Exception e) {
        processedMessageRepository.deleteByDeduplicationKey(deduplicationKey);
        throw e;
    }
}
```
Правильная реализация idempotent consumer с откатом при ошибке.

### 1.3. Plugin Registry для обработки уведомлений
```java
// NotificationManagerImpl.java
private final PluginRegistry<ReportPlugin, String> pluginRegistry;

@Override
public void handleEventByHandler(NotificationDto notificationDtoEvent) {
    pluginRegistry.getPluginsFor(notificationDtoEvent.recipientType())
        .forEach(handler -> handler.handle(notificationDtoEvent));
}
```
Хороший паттерн для расширяемости — легко добавить новые каналы уведомлений (SMS, Push).

### 1.4. Multi-stage Dockerfile
```dockerfile
FROM maven:3.9.9-eclipse-temurin-21 AS deps
# ...
FROM maven:3.9.9-eclipse-temurin-21 AS build
# ...
FROM eclipse-temurin:21-jre
```
Правильный подход с кэшированием зависимостей и минимальным runtime образом.

### 1.5. DLQ (Dead Letter Queue) для failed messages
```java
// PublisherService.java
public void sendToDlq(String mainTopic, Object message) {
    String dlqTopic = mainTopic + DLT_SUFFIX;
    send(dlqTopic, message);
}
```
Правильный подход к обработке сообщений, которые не удалось обработать.

### 1.6. Красивые HTML шаблоны для email
Шаблоны `welcome.html` и `task-summary.html` хорошо оформлены с responsive дизайном.

## 2. Frontend

### 2.1. Drag & Drop для задач
```javascript
const handleDrop = async (event, targetColumn) => {
    // ...
    if (targetColumn === 'new') {
        updatedTask.status = 'NEW';
    } else if (targetColumn === 'active') {
        updatedTask.status = 'IN_PROGRESS';
    }
    await updateTask(updatedTask);
};
```
Реализован Kanban-board интерфейс с drag & drop.

### 2.2. Оптимистичные обновления UI
```javascript
// Обновляем задачу в локальном состоянии сразу
setTasks(prev => prev.map(task =>
    task.id === updatedTask.id ? { ...updatedTask, status: status } : task
));
```

---

# ЗАМЕЧАНИЯ

## auth-service/

### 1. Использование `@Validated` вместо стандартного `@Valid`
```java
// AuthController.java
@Validated
@RestController
public class AuthController {
    public AuthResponse login(@Validated @RequestBody LoginRequest request)
```
`@Validated` — Spring-специфичная аннотация, которая нужна только для:
- Валидации `@PathVariable` / `@RequestParam` (когда аннотация на классе)
- Группы валидации (`@Validated(OnCreate.class)`)

В данном случае достаточно стандартного `@Valid` из Jakarta Bean Validation:
```java
@RestController
public class AuthController {
    public AuthResponse login(@Valid @RequestBody LoginRequest request)
```
В реальной практике чаще используется `@Valid` — он стандартный и переносимый между фреймворками.

### 2. Дублирование `@NotNull` и `@NotEmpty`
```java
// UserRequestDTO.java
@NotNull
@NotEmpty
@Size(min = 5, max = 20)
private String username;
```
`@NotEmpty` уже включает проверку на null.

**Рекомендация:** Использовать `@NotBlank` для строк — он также проверяет, что строка не состоит только из пробелов.

### 3. Разная минимальная длина пароля
```java
// LoginRequest.java
@Size(min = 6, message = "Password must be at least 6 characters")

// UserRequestDTO.java
@Size(min = 8)
```
Разные требования к паролю при логине (6) и регистрации (8).

**Рекомендация:** Использовать единое значение.

### 4. Email может быть null
```java
// User.java
@Column
private String email;  // Nullable

// UserMapper.java
default String getUserEmail(String email) {
    return email != null ? email : "user@mail.ru";
}
```
Email не обязателен, но используется для отправки уведомлений с fallback на захардкоженный адрес.

**Рекомендация:** Сделать email обязательным или не отправлять уведомления без email.

### 5. Возврат Entity вместо DTO
```java
// AuthController.java
@PostMapping("/register")
public User register(@Validated @RequestBody UserRequestDTO request) {
    return userService.signUp(request);
}
```
Возвращается Entity с паролем (хоть и зашифрованным).

**Рекомендация:** Возвращать DTO без чувствительных данных.

### 6. System.out.println в production коде
```java
// SecurityConfig.java
.successHandler((request, response, authentication) -> {
    System.out.println("=== LOGIN SUCCESS ===");
```

**Рекомендация:** Использовать логгер.

## task-service/

### 7. Нет пагинации для списка задач
```java
public List<TaskDto> getTasks() {
    return taskMapper.toDtoList(taskRepository.findAll());
}
```
При большом количестве задач это приведет к проблемам с производительностью.

**Рекомендация:** Добавить пагинацию через `Pageable`.

### 8. Неэффективный подсчет задач
```java
public String getTasksCountByStatusAndUserId(TaskStatus status, Long userId) {
    return String.valueOf(
        taskRepository.getTasksByStatusAndUserId(status, userId).size()
    );
}
```
Загружаются все задачи, чтобы посчитать их количество.

**Рекомендация:** Использовать метод с префиксом `count` — Spring Data сам сгенерирует COUNT запрос:
```java
long countByStatusAndUserId(TaskStatus status, Long userId);
```

### 9. Нет timestamp полей в Task entity
```java
// Task.java
@Entity
public class Task {
    private Long id;
    private String title;
    private String description;
    private Long userId;
    private TaskStatus status;
    // Нет createdAt, updatedAt, completedAt
}
```
По ТЗ требуется время пометки задачи как выполненной.

**Рекомендация:** Добавить audit поля с `@CreatedDate`, `@LastModifiedDate`.

## notification-service/

### 10. @SneakyThrows скрывает исключения
```java
@SneakyThrows
@KafkaListener(topics = "${kafka.topics.outbox}")
public void handleOutboxEvent(ConsumerRecord<String, String> outboxRecord, Acknowledgment ack) {
```
Lombok `@SneakyThrows` скрывает checked exceptions, что усложняет отладку.

**Рекомендация:** Явно обрабатывать исключения или использовать try-catch.

### 11. Парсинг Debezium payload вручную
```java
private String parseDebeziumPayload(String json) {
    return (json.startsWith("\"") && json.endsWith("\"")
        ? json.substring(1, json.length() - 1)
        : json).replace("\\\"", "\"");
}
```
Ручной парсинг JSON ненадежен.

**Рекомендация:** Использовать Jackson ObjectMapper или настроить Debezium transforms.

## frontend/

### 12. Авторизация слетает при перезагрузке страницы (HIGH)
```javascript
// App.jsx
const checkAuthentication = async () => {
    const token = localStorage.getItem('jwt');
    // ...
    const response = await fetch(`${AUTH_API_URL}/validate`, {
        method: 'POST',
        headers: {
            'Authorization': `Bearer ${token}`,
            'Content-Type': 'application/json'
        }
    });
```
При перезагрузке страницы фронтенд вызывает `/api/auth/validate` для проверки токена, но **такого endpoint в auth-service не существует**. Запрос возвращает ошибку, токен удаляется из localStorage, пользователь разлогинивается.

**Рекомендация:** Добавить endpoint в auth-service:
```java
@GetMapping("/validate")
public ResponseEntity<UserDto> validateToken(Authentication authentication) {
    String username = authentication.getName();
    User user = userRepository.findUserByUsername(username)
        .orElseThrow(() -> new UserNotFoundException(username));
    return ResponseEntity.ok(userMapper.toUserDto(user));
}
```
Или валидировать токен локально на фронтенде (проверять expiration из JWT payload).

### 13. console.log в production
```javascript
console.log('🔐 Initial JWT:', localStorage.getItem('jwt'));
console.log('🔍 User data from /validate:', userData);
```
Много отладочных логов, которые не должны быть в production.

**Рекомендация:** Удалить или использовать условную компиляцию.

### 14. Hardcoded URL для Google OAuth
```javascript
window.location.href = `http://localhost:9000/oauth2/authorization/google`;
```
Захардкоженный localhost URL.

**Рекомендация:** Использовать переменную окружения.

### 15. Отсутствует обработка ошибок API
```javascript
const response = await fetch(`${AUTH_API_URL}/validate`, {...});
if (response.ok) {
    // ...
} else {
    localStorage.removeItem('jwt');
}
```
При ошибке валидации просто удаляется токен без информирования пользователя.

---

# СООТВЕТСТВИЕ ТЗ

## ✅ Реализовано корректно:
- Регистрация пользователей
- Авторизация с JWT
- Logout
- Создание задач с заголовком
- Редактирование задач (заголовок, описание)
- Пометка задачи как сделанной
- Удаление задач
- Приветственное письмо при регистрации
- Ежедневная рассылка отчетов (cron)
- Одностраничное приложение с Ajax
- Docker Compose для всех сервисов
- Kafka как брокер сообщений
- CI/CD с GitHub Actions (упомянуто в README)

## ⚠️ Требует исправления:
| Пункт ТЗ | Проблема |
|----------|----------|
| Доступ к чужим задачам | Нет проверки владельца при операциях |
| Список задач | Возвращаются все задачи, фильтрация на фронте |
| Timestamp выполнения | Нет поля completedAt в Task |
| Безопасность API | ⚠️ Работает, но авторизация дублируется в каждом сервисе |
| Модальное окно без кнопки "сохранить" | По ТЗ изменения должны сохраняться автоматически — реализовано частично |

## ❓ Дополнительно реализовано (сверх ТЗ):
| Функционал | Статус |
|------------|--------|
| OAuth2 Authorization Server | ✅ Реализован |
| Google OAuth | ✅ Частично (есть кнопка) |
| Debezium CDC | ✅ Для outbox pattern |
| Drag & Drop Kanban | ✅ Отличная реализация |
| Профиль пользователя | ✅ UI есть, backend нет |
| Настройки уведомлений | ✅ UI есть, backend нет |

---

# ВЫВОД

Проект выполнен **хорошо** с интересными архитектурными решениями. Особенно выделяется:
- Outbox pattern с Debezium для надежной доставки событий
- Plugin-based архитектура для уведомлений
- Красивый и функциональный UI с Kanban-board

## Критичные проблемы (нужно исправить):
1. **Доступ к чужим задачам** → добавить проверку владельца
2. **RSA ключи генерируются при запуске** → хранить персистентно
3. **Задачи возвращаются без фильтрации** → фильтровать на бэкенде
4. **Gateway не выполняет авторизацию** → перенести проверку JWT в Gateway
5. **Ошибки валидации показывают Internal Server Error** → исправить импорт в AuthControllerAdvice
6. **Авторизация слетает при перезагрузке** → добавить endpoint `/api/auth/validate`

## Рекомендации по улучшению:
1. Добавить валидацию в task-service DTOs
2. Добавить audit поля (createdAt, updatedAt, completedAt) в Task
3. Реализовать пагинацию для списка задач
4. Вынести hardcoded значения в конфигурацию
5. Добавить интеграционные тесты (сейчас тестов нет)
6. Использовать nginx для frontend в production
7. Добавить healthcheck для всех сервисов в docker-compose

**Оценка:** 6/10 — проект с интересными архитектурными решениями (Outbox pattern, Plugin Registry), но с критичными проблемами безопасности: доступ к чужим задачам, отсутствие фильтрации на бэкенде, слетающая авторизация.

---

# ПРОЧЕЕ

### Опечатка в названии модуля
```
tracket-auth-service  // Должно быть tracker-auth-service
```
Опечатка в названии директории может вызвать путаницу.

### Смешение языков в логах
```java
// GmailReportPlugin.java
log.info("Cообщение отправлено на почту");
```
Смешение русского и английского языков в логах. Рекомендуется использовать единый язык (предпочтительно английский).

