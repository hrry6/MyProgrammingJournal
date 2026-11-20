# File Upload & Storage Management

[🏠 Home](../README.md) > [🍃 Spring Boot](./README.md)

---

## Configuration (`application.properties`)

Configure Spring's built-in multipart resolver limits and define a configurable storage directory:

```properties
# ==========================================
# MULTIPART FILE UPLOAD LIMITS
# ==========================================
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=5MB
spring.servlet.multipart.max-request-size=20MB

# ==========================================
# CUSTOM STORAGE DIRECTORY
# ==========================================
application.storage.upload-dir=${UPLOAD_DIR:./uploads}
```

### Key Properties

- **`max-file-size`**: Maximum allowed size for a single uploaded file (default is `1MB`).
- **`max-request-size`**: Maximum total payload size for an entire `multipart/form-data` HTTP request (important when uploading multiple files at once; default is `10MB`).
- **Git & Docker Tip**: Add `uploads/` to `.gitignore` and mount it as a persistent Docker volume (`- ./uploads:/app/uploads`) so user files survive container restarts.

---

## Architecture Best Practices for User Files

1. **Never Store Large Files as `BYTEA` in PostgreSQL:** Store binary files on disk or object storage (S3, MinIO, Cloudflare R2) and save only the **file metadata** (UUID filename, original name, MIME type, size, owner ID) in PostgreSQL.
2. **Rename Every Uploaded File with a UUID:** Never save files using `file.getOriginalFilename()` directly—it causes filename collisions and exposes the server to **Path Traversal** attacks (e.g., `../../etc/passwd`).
3. **Abstract Storage Behind an Interface:** Define a `StorageService` interface so switching from local filesystem storage to S3/MinIO later requires zero changes in controllers or business services.

---

## Storage Service Implementation

### A. Interface (`StorageService.java`)

```java
package com.hrry6.app.common.storage;

import org.springframework.core.io.Resource;
import org.springframework.web.multipart.MultipartFile;

public interface StorageService {
    StoredFileInfo store(MultipartFile file);
    Resource loadAsResource(String storedFilename);
    void delete(String storedFilename);
}
```

### B. Response Record (`StoredFileInfo.java`)

```java
package com.hrry6.app.common.storage;

public record StoredFileInfo(
        String storedFilename,
        String originalFilename,
        String contentType,
        long sizeBytes
) {}
```

### C. Custom Storage Exception (`FileStorageException.java`)

```java
package com.hrry6.app.common.exception;

public class FileStorageException extends RuntimeException {
    public FileStorageException(String message) {
        super(message);
    }

    public FileStorageException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### D. Local Filesystem Implementation (`LocalFileStorageService.java`)

Includes empty-file checks, MIME-type allowlisting, UUID renaming, and strict path normalization checks:

```java
package com.hrry6.app.common.storage;

import com.hrry6.app.common.exception.FileStorageException;
import com.hrry6.app.common.exception.ResourceNotFoundException;
import jakarta.annotation.PostConstruct;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.core.io.UrlResource;
import org.springframework.stereotype.Service;
import org.springframework.util.StringUtils;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.io.InputStream;
import java.net.MalformedURLException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.nio.file.StandardCopyOption;
import java.util.Set;
import java.util.UUID;

@Service
public class LocalFileStorageService implements StorageService {

    private static final Set<String> ALLOWED_CONTENT_TYPES = Set.of(
            "image/jpeg",
            "image/png",
            "image/webp",
            "application/pdf"
    );

    private final Path rootLocation;

    public LocalFileStorageService(@Value("${application.storage.upload-dir}") String uploadDir) {
        this.rootLocation = Paths.get(uploadDir).toAbsolutePath().normalize();
    }

    @PostConstruct
    public void init() {
        try {
            Files.createDirectories(rootLocation);
        } catch (IOException ex) {
            throw new FileStorageException("Could not initialize upload directory", ex);
        }
    }

    @Override
    public StoredFileInfo store(MultipartFile file) {
        if (file == null || file.isEmpty()) {
            throw new FileStorageException("Cannot store an empty file");
        }

        String contentType = file.getContentType();
        if (contentType == null || !ALLOWED_CONTENT_TYPES.contains(contentType)) {
            throw new FileStorageException("Unsupported file type: " + contentType);
        }

        String rawOriginalName = file.getOriginalFilename() != null ? file.getOriginalFilename() : "file";
        String cleanOriginalName = StringUtils.cleanPath(rawOriginalName);
        String extension = extractExtension(cleanOriginalName);
        String storedFilename = UUID.randomUUID() + extension;

        try {
            Path destinationFile = resolveSafePath(storedFilename);
            try (InputStream inputStream = file.getInputStream()) {
                Files.copy(inputStream, destinationFile, StandardCopyOption.REPLACE_EXISTING);
            }

            return new StoredFileInfo(
                    storedFilename,
                    cleanOriginalName,
                    contentType,
                    file.getSize()
            );
        } catch (IOException ex) {
            throw new FileStorageException("Failed to store file: " + cleanOriginalName, ex);
        }
    }

    @Override
    public Resource loadAsResource(String storedFilename) {
        try {
            Path filePath = resolveSafePath(storedFilename);
            Resource resource = new UrlResource(filePath.toUri());

            if (resource.exists() && resource.isReadable()) {
                return resource;
            }
            throw new ResourceNotFoundException("File not found: " + storedFilename);
        } catch (MalformedURLException ex) {
            throw new ResourceNotFoundException("File not found: " + storedFilename);
        }
    }

    @Override
    public void delete(String storedFilename) {
        try {
            Path filePath = resolveSafePath(storedFilename);
            Files.deleteIfExists(filePath);
        } catch (IOException ex) {
            throw new FileStorageException("Failed to delete file: " + storedFilename, ex);
        }
    }

    // Prevents Path Traversal (e.g., "../../application.properties")
    private Path resolveSafePath(String filename) {
        Path resolved = rootLocation.resolve(filename).normalize();
        if (!resolved.startsWith(rootLocation)) {
            throw new FileStorageException("Invalid file path outside storage root");
        }
        return resolved;
    }

    private String extractExtension(String filename) {
        int dotIndex = filename.lastIndexOf('.');
        return (dotIndex >= 0) ? filename.substring(dotIndex).toLowerCase() : "";
    }
}
```

---

## Database Metadata Entity (`FileMetadata.java`)

Pair physical storage with a PostgreSQL table to track ownership, original filenames, and clean up orphaned files:

```java
package com.hrry6.app.modules.file;

import com.hrry6.app.common.entity.BaseEntity;
import com.hrry6.app.modules.auth.User;
import jakarta.persistence.*;
import lombok.*;

import java.util.UUID;

@Entity
@Table(name = "file_metadata")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class FileMetadata extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @Column(name = "stored_filename", nullable = false, unique = true)
    private String storedFilename;

    @Column(name = "original_filename", nullable = false)
    private String originalFilename;

    @Column(name = "content_type", nullable = false, length = 100)
    private String contentType;

    @Column(name = "size_bytes", nullable = false)
    private Long sizeBytes;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "uploaded_by", nullable = false)
    private User uploadedBy;
}
```

---

## REST Controller (`FileController.java`)

Supports single/multiple uploads, inline preview (`Content-Disposition: inline`), forced file download (`Content-Disposition: attachment`), and deletion:

```java
package com.hrry6.app.modules.file;

import com.hrry6.app.common.dto.ApiResponse;
import com.hrry6.app.common.storage.StorageService;
import com.hrry6.app.common.storage.StoredFileInfo;
import lombok.RequiredArgsConstructor;
import org.springframework.core.io.Resource;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Paths;
import java.util.Arrays;
import java.util.List;

@RestController
@RequestMapping("/api/v1/files")
@RequiredArgsConstructor
public class FileController {

    private final StorageService storageService;

    // 1. Single File Upload (multipart/form-data)
    @PostMapping(value = "/upload", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ApiResponse<StoredFileInfo>> uploadFile(
            @RequestParam("file") MultipartFile file
    ) {
        StoredFileInfo saved = storageService.store(file);
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.ok("File uploaded successfully", saved));
    }

    // 2. Multiple Files Upload
    @PostMapping(value = "/upload-multiple", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
    public ResponseEntity<ApiResponse<List<StoredFileInfo>>> uploadMultipleFiles(
            @RequestParam("files") MultipartFile[] files
    ) {
        List<StoredFileInfo> uploaded = Arrays.stream(files)
                .map(storageService::store)
                .toList();

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.ok("Files uploaded successfully", uploaded));
    }

    // 3. View / Download File
    // Use ?download=true to force browser Save-As dialog; defaults to inline preview
    @GetMapping("/{filename:.+}")
    public ResponseEntity<Resource> serveFile(
            @PathVariable String filename,
            @RequestParam(defaultValue = "false") boolean download
    ) throws IOException {
        Resource resource = storageService.loadAsResource(filename);

        String contentType = Files.probeContentType(Paths.get(resource.getFilename()));
        if (contentType == null) {
            contentType = MediaType.APPLICATION_OCTET_STREAM_VALUE;
        }

        String dispositionType = download ? "attachment" : "inline";

        return ResponseEntity.ok()
                .contentType(MediaType.parseMediaType(contentType))
                .header(HttpHeaders.CONTENT_DISPOSITION,
                        dispositionType + "; filename=\"" + resource.getFilename() + "\"")
                .body(resource);
    }

    // 4. Delete File
    @DeleteMapping("/{filename:.+}")
    public ResponseEntity<Void> deleteFile(@PathVariable String filename) {
        storageService.delete(filename);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Global Exception Handler Additions

Add handlers in `GlobalExceptionHandler.java` to catch oversized uploads and invalid file types cleanly:

```java
// 400 - Empty file, unsupported MIME type, or path traversal attempt
@ExceptionHandler(FileStorageException.class)
public ResponseEntity<ApiResponse<Void>> handleFileStorage(FileStorageException ex) {
    return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ApiResponse.error(ex.getMessage(), null));
}

// 413 - File exceeds spring.servlet.multipart.max-file-size
@ExceptionHandler(MaxUploadSizeExceededException.class)
public ResponseEntity<ApiResponse<Void>> handleMaxUploadSize(MaxUploadSizeExceededException ex) {
    return ResponseEntity
            .status(HttpStatus.PAYLOAD_TOO_LARGE)
            .body(ApiResponse.error("Uploaded file exceeds the maximum allowed size limit", null));
}
```

---

## Security & Gotchas Checklist

| Pitfall | Mitigation |
| :--- | :--- |
| **Path Traversal (`../`)** | Always resolve target paths against `rootLocation.toAbsolutePath().normalize()` and verify `resolved.startsWith(rootLocation)` |
| **Spoofed `Content-Type` Header** | `file.getContentType()` reads the client's HTTP header; for high-security apps, inspect file magic bytes using **Apache Tika** (`new Tika().detect(file.getInputStream())`) |
| **Executing Uploaded Scripts** | Never store user uploads inside `src/main/resources/static/` or `public/`, and serve files with `X-Content-Type-Options: nosniff` (enabled by default in Spring Security) |
| **Orphaned Files on DB Rollback** | If saving `FileMetadata` to PostgreSQL fails after `storageService.store(file)` succeeds, catch the exception or use `@TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)` to delete the orphaned file from disk |
