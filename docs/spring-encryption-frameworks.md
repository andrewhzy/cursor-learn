# Spring Frameworks for Transparent Encryption

## Overview
This document covers Spring-based frameworks and libraries that provide transparent encryption/decryption capabilities, eliminating the need for manual encryption handling in application code.

## 1. Jasypt Spring Boot Starter (Most Popular)

### Description
Jasypt (Java Simplified Encryption) is the most widely used library for transparent encryption in Spring Boot applications.

### Maven Dependency
```xml
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

### Configuration
```properties
# application.properties
jasypt.encryptor.password=mySecretKey
jasypt.encryptor.algorithm=PBEWithMD5AndDES
jasypt.encryptor.key-obtention-iterations=1000
jasypt.encryptor.pool-size=1
jasypt.encryptor.provider-name=SunJCE
jasypt.encryptor.salt-generator-classname=org.jasypt.salt.RandomSaltGenerator
jasypt.encryptor.string-output-type=base64
```

### Entity Implementation
```java
@Entity
@Table(name = "tasks")
public class Task {
    @Id
    private UUID id;
    
    private String userId;
    
    // Transparent encryption using Jasypt
    @EncryptedJson
    @Column(name = "metadata")
    private Map<String, Object> metadata;
    
    @EncryptedString
    @Column(name = "original_filename")
    private String originalFilename;
    
    @EncryptedBytes
    @Column(name = "task_file_blob")
    private byte[] taskFileBlob;
    
    // Standard getters/setters - encryption is transparent
}
```

### Custom Annotations with Jasypt
```java
// Custom annotation for encrypted strings
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Convert(converter = EncryptedStringConverter.class)
public @interface EncryptedString {
}

// Custom annotation for encrypted JSON
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Convert(converter = EncryptedJsonConverter.class)
public @interface EncryptedJson {
}

// Custom annotation for encrypted bytes
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Convert(converter = EncryptedBytesConverter.class)
public @interface EncryptedBytes {
}
```

### Converters Implementation
```java
@Component
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    
    @Autowired
    private StringEncryptor stringEncryptor;
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        if (attribute == null) return null;
        return stringEncryptor.encrypt(attribute);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        return stringEncryptor.decrypt(dbData);
    }
}

@Component
public class EncryptedBytesConverter implements AttributeConverter<byte[], String> {
    
    @Autowired
    private ByteEncryptor byteEncryptor;
    
    @Override
    public String convertToDatabaseColumn(byte[] attribute) {
        if (attribute == null) return null;
        byte[] encrypted = byteEncryptor.encrypt(attribute);
        return Base64.getEncoder().encodeToString(encrypted);
    }
    
    @Override
    public byte[] convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        byte[] decoded = Base64.getDecoder().decode(dbData);
        return byteEncryptor.decrypt(decoded);
    }
}
```

## 2. Spring Boot Data JPA with Custom Encryption

### Configuration Class
```java
@Configuration
@EnableJpaRepositories
public class EncryptionConfig {
    
    @Bean
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();
        config.setPassword("encryption-password");
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
        config.setStringOutputType("base64");
        encryptor.setConfig(config);
        return encryptor;
    }
    
    @Bean
    public ByteEncryptor byteEncryptor() {
        PooledPBEByteEncryptor encryptor = new PooledPBEByteEncryptor();
        SimpleByteEncryptorPBEConfig config = new SimpleByteEncryptorPBEConfig();
        config.setPassword("encryption-password");
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setProviderName("SunJCE");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
        encryptor.setConfig(config);
        return encryptor;
    }
}
```

## 3. Spring Vault Integration

### Description
Spring Vault provides integration with HashiCorp Vault for external key management and encryption services.

### Maven Dependencies
```xml
<dependency>
    <groupId>org.springframework.vault</groupId>
    <artifactId>spring-vault-core</artifactId>
    <version>3.0.4</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-vault</artifactId>
</dependency>
```

### Configuration
```properties
# application.properties
spring.cloud.vault.host=localhost
spring.cloud.vault.port=8200
spring.cloud.vault.scheme=https
spring.cloud.vault.authentication=token
spring.cloud.vault.token=your-vault-token
spring.cloud.vault.kv.enabled=true
spring.cloud.vault.kv.backend=secret
```

### Vault Encryption Service
```java
@Service
public class VaultEncryptionService {
    
    @Autowired
    private VaultOperations vaultOperations;
    
    public String encrypt(String plaintext) {
        VaultTransitOperations transitOperations = vaultOperations.opsForTransit();
        return transitOperations.encrypt("my-key", plaintext);
    }
    
    public String decrypt(String ciphertext) {
        VaultTransitOperations transitOperations = vaultOperations.opsForTransit();
        return transitOperations.decrypt("my-key", ciphertext);
    }
}

@Component
public class VaultStringConverter implements AttributeConverter<String, String> {
    
    @Autowired
    private VaultEncryptionService vaultEncryptionService;
    
    @Override
    public String convertToDatabaseColumn(String attribute) {
        if (attribute == null) return null;
        return vaultEncryptionService.encrypt(attribute);
    }
    
    @Override
    public String convertToEntityAttribute(String dbData) {
        if (dbData == null) return null;
        return vaultEncryptionService.decrypt(dbData);
    }
}
```

## 4. Spring Data JPA Encrypt (Lightweight Option)

### Description
A lightweight library specifically designed for JPA entity encryption.

### Maven Dependency
```xml
<dependency>
    <groupId>io.github.manami-project</groupId>
    <artifactId>spring-data-jpa-encrypt</artifactId>
    <version>1.0.1</version>
</dependency>
```

### Configuration
```java
@Configuration
@EnableJpaRepositories
@EnableEncryptableRepositories
public class JpaEncryptConfig {
    
    @Bean
    public EncryptionService encryptionService() {
        return new AESEncryptionService("your-secret-key");
    }
}
```

### Entity Usage
```java
@Entity
@Table(name = "tasks")
public class Task {
    @Id
    private UUID id;
    
    @Encrypted
    @Column(name = "original_filename")
    private String originalFilename;
    
    @Encrypted
    @Column(name = "metadata")
    private String metadata;
    
    // No special handling needed - encryption is automatic
}
```

## 5. Custom Spring AOP Approach

### Aspect Implementation
```java
@Aspect
@Component
public class EncryptionAspect {
    
    @Autowired
    private EncryptionService encryptionService;
    
    @Around("@annotation(EncryptedField)")
    public Object encryptField(ProceedingJoinPoint joinPoint) throws Throwable {
        Object[] args = joinPoint.getArgs();
        
        // Encrypt arguments before method execution
        for (int i = 0; i < args.length; i++) {
            if (args[i] instanceof String) {
                args[i] = encryptionService.encrypt((String) args[i]);
            }
        }
        
        Object result = joinPoint.proceed(args);
        
        // Decrypt result if needed
        if (result instanceof String) {
            result = encryptionService.decrypt((String) result);
        }
        
        return result;
    }
}

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface EncryptedField {
}
```

## 6. Comparison Matrix

| Framework | Ease of Use | Performance | Key Management | Community Support | Enterprise Ready |
|-----------|-------------|-------------|----------------|-------------------|-------------------|
| **Jasypt** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Spring Vault** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **JPA Encrypt** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **Custom AOP** | ⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐ | ⭐⭐ |

## 7. Recommended Implementation for Your Project

### For Task Management API (Best Choice: Jasypt)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.github.ulisesbocchio</groupId>
    <artifactId>jasypt-spring-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
```

```java
// TaskEntity.java
@Entity
@Table(name = "tasks")
public class Task {
    @Id
    private UUID id;
    
    private String userId;
    private String taskStatus;
    
    // Transparent encryption - no manual encrypt/decrypt needed
    @EncryptedString
    @Column(name = "original_filename")
    private String originalFilename;
    
    @EncryptedJson
    @Column(name = "metadata")
    private Map<String, Object> metadata;
    
    @EncryptedBytes
    @Column(name = "task_file_blob")
    private byte[] taskFileBlob;
    
    @EncryptedBytes
    @Column(name = "results_file_blob")
    private byte[] resultsFileBlob;
    
    // Standard getters/setters - no encryption logic needed
}

// ChatEvaluationInput.java
@Entity
@Table(name = "chat_evaluation_input")
public class ChatEvaluationInput {
    @Id
    private Long id;
    
    private UUID taskId;
    private Integer rowNumber;
    
    @EncryptedString
    @Column(name = "question")
    private String question;
    
    @EncryptedString
    @Column(name = "golden_answer")
    private String goldenAnswer;
    
    @EncryptedJson
    @Column(name = "golden_citations")
    private List<String> goldenCitations;
    
    // Transparent encryption - no special handling needed
}
```

### Configuration
```properties
# application.properties
jasypt.encryptor.password=${ENCRYPTION_PASSWORD:default-dev-password}
jasypt.encryptor.algorithm=PBEWITHHMACSHA512ANDAES_256
jasypt.encryptor.key-obtention-iterations=1000
jasypt.encryptor.pool-size=4
jasypt.encryptor.string-output-type=base64
```

### Service Layer (No Encryption Logic Needed)
```java
@Service
@Transactional
public class TaskService {
    
    @Autowired
    private TaskRepository taskRepository;
    
    public Task createTask(String userId, String filename, byte[] fileData) {
        Task task = new Task();
        task.setUserId(userId);
        task.setOriginalFilename(filename); // Automatically encrypted
        task.setTaskFileBlob(fileData);     // Automatically encrypted
        
        return taskRepository.save(task);   // Encryption happens transparently
    }
    
    public Task getTask(UUID taskId) {
        Task task = taskRepository.findById(taskId).orElse(null);
        // Data is automatically decrypted when loaded
        return task;
    }
}
```

## 8. Migration Strategy with Jasypt

### Phase 1: Add Jasypt Dependencies
```bash
# Add to pom.xml and restart application
mvn clean install
```

### Phase 2: Add Annotations Gradually
```java
// Start with one table at a time
@Entity
public class Task {
    // Keep existing fields as-is initially
    private String originalFilename;
    
    // Add encrypted version alongside
    @EncryptedString
    @Column(name = "original_filename_encrypted")
    private String originalFilenameEncrypted;
}
```

### Phase 3: Background Migration
```java
@Component
public class EncryptionMigrationService {
    
    @Scheduled(fixedDelay = 60000)
    public void migrateToEncryption() {
        List<Task> unencryptedTasks = taskRepository
            .findTop100ByOriginalFilenameEncryptedIsNull();
            
        for (Task task : unencryptedTasks) {
            // Jasypt will automatically encrypt when setting
            task.setOriginalFilenameEncrypted(task.getOriginalFilename());
            taskRepository.save(task);
        }
    }
}
```

### Phase 4: Switch to Encrypted Fields
```java
// Update entity to use encrypted field as primary
@Entity
public class Task {
    @EncryptedString
    @Column(name = "original_filename")
    private String originalFilename; // Now encrypted transparently
}
```

## 9. Advantages of Transparent Encryption

### ✅ Benefits:
- **Zero Code Changes**: Service layer remains unchanged
- **Type Safety**: No string conversion issues
- **Automatic Handling**: Encryption/decryption is invisible
- **Spring Integration**: Works seamlessly with Spring Data JPA
- **Performance**: Optimized for database operations
- **Testing**: Easy to test with regular data

### ⚠️ Considerations:
- **Query Limitations**: Cannot query encrypted fields directly
- **Index Performance**: Encrypted fields cannot be indexed efficiently
- **Key Management**: Need proper key rotation strategy
- **Migration Complexity**: Requires careful planning for existing data

## Recommendation

**For your Task Management API, I recommend Jasypt Spring Boot Starter** because:

1. **Mature and Stable**: Widely used in production environments
2. **Spring Boot Integration**: Seamless integration with your existing stack
3. **Transparent Operations**: No changes needed in service layer
4. **Good Performance**: Optimized for JPA operations
5. **Strong Community**: Well-documented and supported
6. **Enterprise Ready**: Supports key rotation and management

The implementation would be completely transparent to your business logic, making it the ideal choice for your requirements. 