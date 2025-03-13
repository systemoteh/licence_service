# licence_service

Пример приложения на Spring Boot 3, которое реализует сервер лицензирования:

1. Сначала добавим зависимости в `pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <!-- ... -->
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
    <!-- ... -->
</project>
```

2. Класс модели запроса (`LicenseRequest.java`):

```java
public class LicenseRequest {
    private String cpuId;
    private String osName;
    private long totalMemory;
    private String macAddress;

    // Геттеры и сеттеры
}
```

3. Класс ответа (`LicenseResponse.java`):

```java
public class LicenseResponse {
    private boolean valid;
    private String message;

    // Конструкторы, геттеры и сеттеры
}
```

4. Конфигурационные свойства (`LicenseProperties.java`):

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import java.util.List;

@ConfigurationProperties(prefix = "license")
public class LicenseProperties {
    private List<String> allowedMacs;
    private long minimumMemory = 8589934592L; // 8GB по умолчанию
    private List<String> allowedCpuIds;

    // Геттеры и сеттеры
}
```

5. Сервис проверки лицензии (`LicenseService.java`):

```java
import org.springframework.stereotype.Service;

@Service
public class LicenseService {
    private final LicenseProperties properties;

    public LicenseService(LicenseProperties properties) {
        this.properties = properties;
    }

    public LicenseResponse validate(LicenseRequest request) {
        List<String> errors = new ArrayList<>();

        if (!properties.getAllowedMacs().contains(request.getMacAddress())) {
            errors.add("Неразрешенный MAC-адрес");
        }

        if (request.getTotalMemory() < properties.getMinimumMemory()) {
            errors.add("Недостаточно оперативной памяти (минимум %d GB)"
                .formatted(properties.getMinimumMemory() / 1024 / 1024 / 1024));
        }

        if (!properties.getAllowedCpuIds().contains(request.getCpuId())) {
            errors.add("Неразрешенный идентификатор CPU");
        }

        if (errors.isEmpty()) {
            return new LicenseResponse(true, "Лицензия действительна");
        }
        return new LicenseResponse(false, "Ошибки: " + String.join(", ", errors));
    }
}
```

6. REST контроллер (`LicenseController.java`):

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/license")
public class LicenseController {
    private final LicenseService licenseService;

    public LicenseController(LicenseService licenseService) {
        this.licenseService = licenseService;
    }

    @PostMapping("/validate")
    public LicenseResponse validateLicense(@RequestBody LicenseRequest request) {
        return licenseService.validate(request);
    }
}
```

7. Главный класс приложения:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.properties.EnableConfigurationProperties;

@SpringBootApplication
@EnableConfigurationProperties(LicenseProperties.class)
public class LicenseServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(LicenseServerApplication.class, args);
    }
}
```

8. Конфигурация в `application.yml`:

```yaml
license:
  allowed-macs:
    - "00:1A:2B:3C:4D:5E"
    - "AA:BB:CC:DD:EE:FF"
  minimum-memory: 17179869184 # 16GB
  allowed-cpu-ids:
    - "BFEBFBFF000306C3"
    - "AuthenticAMD123456"
```

Для тестирования можно отправить POST запрос на `http://localhost:8080/api/license/validate` с JSON телами:

Пример валидного запроса:
```json
{
    "cpuId": "BFEBFBFF000306C3",
    "osName": "Windows 10",
    "totalMemory": 17179869184,
    "macAddress": "00:1A:2B:3C:4D:5E"
}
```

Пример ответа:
```json
{
    "valid": true,
    "message": "Лицензия действительна"
}
```

Пример невалидного запроса:
```json
{
    "cpuId": "InvalidCPU",
    "osName": "Linux",
    "totalMemory": 4294967296,
    "macAddress": "11:22:33:44:55:66"
}
```

Пример ответа:
```json
{
    "valid": false,
    "message": "Ошибки: Неразрешенный MAC-адрес, Недостаточно оперативной памяти (минимум 16 GB), Неразрешенный идентификатор CPU"
}
```

Это базовая реализация, которую можно расширить следующими способами:
1. Добавить аутентификацию клиентов
2. Реализовать хранение лицензий в базе данных
3. Добавить механизм выдачи временных лицензий
4. Реализовать журналирование запросов
5. Добавить дополнительные проверки оборудования
6. Реализовать систему черного/белого списков
7. Добавить шифрование коммуникации
