# Database Encryption Strategy

## Overview
This document outlines the encryption strategy for all five tables in the Task Management API, focusing on protecting sensitive data including file blobs, personal information, and business data.

## Tables Analysis and Encryption Requirements

### 1. `tasks` Table - **HIGH PRIORITY ENCRYPTION**

#### Sensitive Columns Requiring Encryption:
- **`task_file_blob`** (BYTEA) - Contains original Excel files with potentially sensitive business data
- **`results_file_blob`** (BYTEA) - Contains processed results that may include computed sensitive information
- **`original_filename`** (VARCHAR) - May contain sensitive information about business processes
- **`metadata`** (JSONB) - Could contain sensitive configuration or processing details
- **`error_message`** (TEXT) - May leak sensitive file content or processing details

#### Encryption Strategy:
```sql
-- Recommended approach: Application-level encryption with field-level keys
ALTER TABLE tasks 
ADD COLUMN task_file_blob_encrypted BYTEA,
ADD COLUMN results_file_blob_encrypted BYTEA,
ADD COLUMN original_filename_encrypted TEXT,
ADD COLUMN metadata_encrypted TEXT,
ADD COLUMN error_message_encrypted TEXT,
ADD COLUMN encryption_key_id VARCHAR(50);
```

### 2. `chat_evaluation_input` Table - **HIGH PRIORITY ENCRYPTION**

#### Sensitive Columns Requiring Encryption:
- **`question`** (TEXT) - May contain business-sensitive questions or confidential queries
- **`golden_answer`** (TEXT) - Contains expected answers that may reveal business logic
- **`golden_citations`** (JSONB) - URLs may point to internal/sensitive resources

#### Encryption Strategy:
```sql
ALTER TABLE chat_evaluation_input
ADD COLUMN question_encrypted TEXT,
ADD COLUMN golden_answer_encrypted TEXT,
ADD COLUMN golden_citations_encrypted TEXT,
ADD COLUMN encryption_key_id VARCHAR(50);
```

### 3. `chat_evaluation_output` Table - **HIGH PRIORITY ENCRYPTION**

#### Sensitive Columns Requiring Encryption:
- **`api_answer`** (TEXT) - Contains AI-generated responses that may include sensitive information
- **`api_citations`** (JSONB) - Generated citations may reveal internal data sources

#### Encryption Strategy:
```sql
ALTER TABLE chat_evaluation_output
ADD COLUMN api_answer_encrypted TEXT,
ADD COLUMN api_citations_encrypted TEXT,
ADD COLUMN encryption_key_id VARCHAR(50);
```

### 4. `url_cleaning_input` Table - **MEDIUM PRIORITY ENCRYPTION**

#### Sensitive Columns Requiring Encryption:
- **`url`** (TEXT) - URLs may contain sensitive internal endpoints or parameters
- **`parsed_title`** (VARCHAR) - Page titles may reveal sensitive content

#### Encryption Strategy:
```sql
ALTER TABLE url_cleaning_input
ADD COLUMN url_encrypted TEXT,
ADD COLUMN parsed_title_encrypted TEXT,
ADD COLUMN encryption_key_id VARCHAR(50);
```

### 5. `url_cleaning_output` Table - **MEDIUM PRIORITY ENCRYPTION**

#### Sensitive Columns Requiring Encryption:
- **`original_url`** (TEXT) - Same sensitivity as input URLs
- **`cleaned_url`** (TEXT) - Processed URLs may still contain sensitive information
- **`processing_notes`** (TEXT) - May contain details about internal processing logic

#### Encryption Strategy:
```sql
ALTER TABLE url_cleaning_output
ADD COLUMN original_url_encrypted TEXT,
ADD COLUMN cleaned_url_encrypted TEXT,
ADD COLUMN processing_notes_encrypted TEXT,
ADD COLUMN encryption_key_id VARCHAR(50);
```

## Encryption Implementation Strategy

### 1. Field-Level Encryption (Recommended)

#### Advantages:
- Granular control over what data is encrypted
- Can query non-encrypted fields normally
- Better performance than full-table encryption
- Supports partial data access without decryption

#### Implementation Approach:
```java
@Entity
@Table(name = "tasks")
public class Task {
    // Non-encrypted fields remain normal
    private UUID id;
    private String userId;
    private TaskStatus taskStatus;
    
    // Encrypted fields use special annotations
    @Encrypted
    @Column(name = "task_file_blob_encrypted")
    private byte[] taskFileBlob;
    
    @Encrypted
    @Column(name = "original_filename_encrypted")
    private String originalFilename;
    
    @Column(name = "encryption_key_id")
    private String encryptionKeyId;
}
```

### 2. Key Management Strategy

#### Hierarchical Key Structure:
```
Master Key (HSM/Cloud KMS)
├── Application-Level Key (rotated yearly)
│   ├── Table-Level Keys (rotated quarterly)
│   │   ├── User-Level Keys (rotated monthly)
│   │   └── Task-Level Keys (per upload batch)
```

#### Key Rotation Policy:
- **Master Keys**: Every 2 years or on compromise
- **Application Keys**: Every 12 months
- **Table Keys**: Every 3 months
- **User/Task Keys**: Every 1 month or per batch

### 3. Encryption Algorithms and Standards

#### Primary Encryption:
- **Algorithm**: AES-256-GCM (Galois/Counter Mode)
- **Key Size**: 256-bit keys
- **Nonce**: 96-bit random nonce per record
- **Authentication**: Built-in authentication tag

#### Blob File Encryption:
```java
// For large file blobs, use streaming encryption
public byte[] encryptFileBlob(byte[] fileData, String keyId) {
    SecretKey key = keyManager.getKey(keyId);
    Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
    
    byte[] nonce = generateRandomNonce(12); // 96-bit nonce
    GCMParameterSpec spec = new GCMParameterSpec(128, nonce);
    cipher.init(Cipher.ENCRYPT_MODE, key, spec);
    
    byte[] encrypted = cipher.doFinal(fileData);
    
    // Prepend nonce to encrypted data
    ByteBuffer buffer = ByteBuffer.allocate(nonce.length + encrypted.length);
    buffer.put(nonce);
    buffer.put(encrypted);
    return buffer.array();
}
```

### 4. Performance Considerations

#### Database Indexing Strategy:
```sql
-- Only index non-encrypted fields for queries
CREATE INDEX idx_tasks_user_status_encrypted ON tasks(user_id, task_status);
CREATE INDEX idx_tasks_encryption_key ON tasks(encryption_key_id);

-- For encrypted field searches, use hash-based lookups
CREATE INDEX idx_tasks_filename_hash ON tasks(SHA256(original_filename_encrypted));
```

#### Caching Strategy:
- **Encrypted Storage**: Always store encrypted in database
- **Memory Cache**: Cache decrypted data temporarily (max 5 minutes)
- **Application Cache**: Never cache encryption keys in application memory
- **Redis Cache**: Store encrypted data only, decrypt on retrieval

### 5. Migration Strategy

#### Phase 1: Add Encrypted Columns (Zero Downtime)
```sql
-- Add new encrypted columns without dropping existing ones
ALTER TABLE tasks ADD COLUMN task_file_blob_encrypted BYTEA;
ALTER TABLE tasks ADD COLUMN encryption_key_id VARCHAR(50);
-- Repeat for all tables and sensitive columns
```

#### Phase 2: Dual-Write Implementation
```java
@Transactional
public void saveTask(Task task) {
    // Write to both encrypted and plain columns during migration
    String keyId = keyManager.generateTaskKey(task.getUserId());
    
    task.setTaskFileBlob(plaintextBlob);
    task.setTaskFileBlobEncrypted(encrypt(plaintextBlob, keyId));
    task.setEncryptionKeyId(keyId);
    
    taskRepository.save(task);
}
```

#### Phase 3: Migration Background Job
```java
@Scheduled(fixedDelay = 60000) // Run every minute
public void migrateToEncryption() {
    List<Task> unencryptedTasks = taskRepository
        .findTop100ByEncryptionKeyIdIsNull();
    
    for (Task task : unencryptedTasks) {
        String keyId = keyManager.generateTaskKey(task.getUserId());
        task.setTaskFileBlobEncrypted(
            encrypt(task.getTaskFileBlob(), keyId)
        );
        task.setEncryptionKeyId(keyId);
        taskRepository.save(task);
    }
}
```

#### Phase 4: Switch to Encrypted Reads
```java
// Update application to read from encrypted columns
public Task getTask(UUID taskId) {
    Task task = taskRepository.findById(taskId);
    
    // Decrypt sensitive fields
    byte[] decryptedBlob = decrypt(
        task.getTaskFileBlobEncrypted(), 
        task.getEncryptionKeyId()
    );
    task.setTaskFileBlob(decryptedBlob);
    
    return task;
}
```

#### Phase 5: Drop Plain Columns
```sql
-- After migration is complete and verified
ALTER TABLE tasks DROP COLUMN task_file_blob;
ALTER TABLE tasks DROP COLUMN original_filename;
-- Rename encrypted columns
ALTER TABLE tasks RENAME COLUMN task_file_blob_encrypted TO task_file_blob;
```

## Spring Boot Implementation

### 1. Encryption Configuration
```java
@Configuration
@EnableConfigurationProperties(EncryptionProperties.class)
public class EncryptionConfig {
    
    @Bean
    public AESEncryptionService encryptionService(
            EncryptionProperties properties) {
        return new AESEncryptionService(properties);
    }
    
    @Bean
    public KeyManagementService keyManagementService() {
        // Integrate with AWS KMS, Azure Key Vault, or HashiCorp Vault
        return new CloudKMSKeyManagementService();
    }
}
```

### 2. JPA Attribute Converters
```java
@Converter
public class EncryptedStringConverter implements AttributeConverter<String, String> {
    
    @Autowired
    private AESEncryptionService encryptionService;
    
    @Override
    public String convertToDatabaseColumn(String plainText) {
        if (plainText == null) return null;
        return encryptionService.encrypt(plainText);
    }
    
    @Override
    public String convertToEntityAttribute(String encrypted) {
        if (encrypted == null) return null;
        return encryptionService.decrypt(encrypted);
    }
}
```

### 3. Custom Annotations
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Encrypted {
    String keyType() default "USER_LEVEL";
    boolean indexable() default false;
}
```

## Security and Compliance

### 1. Access Control
- **Database Level**: Restrict direct database access to encrypted columns
- **Application Level**: Role-based access to decryption functionality
- **API Level**: Only return decrypted data to authorized users

### 2. Audit Requirements
```java
@Entity
@Table(name = "encryption_audit_log")
public class EncryptionAuditLog {
    private UUID id;
    private String tableName;
    private String columnName;
    private String operation; // ENCRYPT, DECRYPT, KEY_ROTATION
    private String userId;
    private String keyId;
    private Instant timestamp;
    private String ipAddress;
    private String sessionId;
}
```

### 3. Compliance Considerations
- **GDPR**: Right to be forgotten - implement secure key deletion
- **SOX**: Financial data protection for business-related Excel files
- **HIPAA**: If any health-related data is processed
- **PCI DSS**: If any payment-related data is involved

## Monitoring and Alerting

### 1. Key Metrics to Monitor
- Encryption/decryption performance
- Key rotation completion rates
- Failed decryption attempts
- Unusual access patterns to encrypted data

### 2. Security Alerts
- Multiple failed decryption attempts
- Key rotation failures
- Unauthorized access to encryption keys
- Performance degradation in encryption operations

## Disaster Recovery

### 1. Key Backup Strategy
- **Primary**: Cloud KMS with multi-region replication
- **Secondary**: Encrypted key backups in separate cloud region
- **Tertiary**: Offline encrypted key escrow

### 2. Recovery Procedures
- Document step-by-step key recovery process
- Test recovery procedures quarterly
- Maintain recovery time objectives (RTO < 4 hours)
- Ensure recovery point objectives (RPO < 1 hour)

## Next Steps

1. **Phase 1** (Week 1-2): Implement encryption service and key management
2. **Phase 2** (Week 3-4): Add encrypted columns to all five tables
3. **Phase 3** (Week 5-6): Implement dual-write functionality
4. **Phase 4** (Week 7-8): Run background migration job
5. **Phase 5** (Week 9-10): Switch to encrypted reads and verify
6. **Phase 6** (Week 11-12): Drop plain columns and optimize performance

## Cost Analysis

### Storage Impact:
- **Base64 Encoding**: ~33% storage increase for encrypted text
- **Nonce + Auth Tag**: Additional 28 bytes per encrypted field
- **Key References**: 50 bytes per record for key management

### Performance Impact:
- **Encryption**: ~1-2ms per field for small data
- **Decryption**: ~1-2ms per field for small data
- **Large Blobs**: Streaming encryption for files >1MB

### Infrastructure Costs:
- **Key Management Service**: ~$1-3 per 10,000 operations
- **Additional Storage**: ~20-30% increase in database size
- **CPU Overhead**: ~5-10% increase in application CPU usage 