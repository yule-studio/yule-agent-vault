---
title: "파일/이미지 업로드 (S3 presigned URL) — Java Spring Boot"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - file-upload
  - s3
---

# 파일/이미지 업로드 (S3 presigned URL) — Java Spring Boot

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | presigned PUT / 직접 업로드 / MIME 검증 / 후처리 |

**[[api-design|↑ api-design hub]]**

> 📐 **ORM**: §6 는 JPA Adapter sketch. 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 왜 presigned URL — 서버 경유 X

**나쁜 방법**: 클라 → 서버 (multipart) → S3. 서버가 다 받음 → 메모리/네트워크 부담 + heap OOM 위험.

**좋은 방법 (표준)**:
```
1. 클라 → POST /api/v1/files/presigned-url { contentType, sizeBytes, kind }
   → 서버: 검증 + S3 presigned PUT URL (5분 만료) + 우리 file 메타 row pre-create
   → 응답: { uploadUrl, fileId, formFields }
2. 클라 → PUT uploadUrl (binary). 서버 거치지 않음.
3. 클라 → POST /api/v1/files/{fileId}/complete
   → 서버: S3 HEAD 로 실제 업로드 확인 + 메타 status COMPLETED + 후처리 trigger
4. (S3 Event) → Lambda thumbnail 생성 / EXIF strip / virus scan
```

이점:
- 서버 비용 ↓ (binary 안 거침)
- 큰 파일 (동영상 GB) OK
- 직접 multi-part upload 가능

---

## 2. 무엇을 만드는가

```
POST /api/v1/files/presigned-url     # 업로드 URL 발급
POST /api/v1/files/{id}/complete     # 업로드 완료 확인
GET  /api/v1/files/{id}              # 메타 + download URL (presigned GET)
DELETE /api/v1/files/{id}            # soft delete + S3 expire 표시
```

---

## 3. 도메인

```java
// domain/file/FileMeta.java
public final class FileMeta {

    public enum Kind { PROFILE_IMAGE, PRODUCT_IMAGE, REVIEW_IMAGE, ATTACHMENT, VIDEO }
    public enum Status { PENDING, COMPLETED, FAILED, DELETED }

    private final FileId id;
    private final UserId ownerId;
    private final Kind kind;
    private final String s3Bucket;
    private final String s3Key;
    private final String contentType;
    private final long sizeBytes;
    private final String originalFilename;
    private Status status;
    private final Instant createdAt;
    private Instant completedAt;

    public static FileMeta initiate(FileId id, UserId ownerId, Kind kind,
                                    String bucket, String key, String contentType,
                                    long sizeBytes, String originalName, Instant now) {
        return new FileMeta(id, ownerId, kind, bucket, key, contentType,
                            sizeBytes, originalName, Status.PENDING, now, null);
    }

    public void markCompleted(long verifiedSize, Instant now) {
        if (status != Status.PENDING) {
            throw new BusinessException(ResponseCode.CONFLICT, "이미 완료된 파일");
        }
        if (verifiedSize != sizeBytes) {
            throw new BusinessException(ResponseCode.BAD_REQUEST,
                "선언 크기와 실제 크기가 다릅니다: " + sizeBytes + " vs " + verifiedSize);
        }
        this.status = Status.COMPLETED;
        this.completedAt = now;
    }

    public void markDeleted() { status = Status.DELETED; }

    // getters
}
```

DB:
```sql
CREATE TABLE files (
    id              CHAR(26) PRIMARY KEY,
    owner_id        CHAR(26) NOT NULL REFERENCES users(id),
    kind            VARCHAR(30) NOT NULL,
    s3_bucket       VARCHAR(100) NOT NULL,
    s3_key          VARCHAR(500) NOT NULL,
    content_type    VARCHAR(100) NOT NULL,
    size_bytes      BIGINT NOT NULL CHECK (size_bytes > 0),
    original_name   VARCHAR(255),
    status          VARCHAR(20) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
CREATE UNIQUE INDEX ux_files_s3 ON files (s3_bucket, s3_key);
CREATE INDEX ix_files_owner_kind ON files (owner_id, kind, status);
```

---

## 4. S3 Service

```kotlin
// build.gradle.kts
implementation("software.amazon.awssdk:s3:2.25.50")
implementation("software.amazon.awssdk:s3-transfer-manager:2.25.50")
```

```java
// infrastructure/external/s3/S3FileService.java
@Component
@RequiredArgsConstructor
public class S3FileService {

    private final S3Client s3;
    private final S3Presigner presigner;
    @Value("${app.s3.bucket}") String bucket;
    @Value("${app.s3.upload-url-ttl-minutes:5}") int uploadTtlMinutes;
    @Value("${app.s3.download-url-ttl-minutes:15}") int downloadTtlMinutes;
    @Value("${app.s3.cdn-base-url}") String cdnBaseUrl;     // CloudFront / fronted

    public PresignedUpload buildUploadUrl(String key, String contentType, long sizeBytes) {
        var putRequest = PutObjectRequest.builder()
            .bucket(bucket)
            .key(key)
            .contentType(contentType)
            .contentLength(sizeBytes)
            // Content-MD5 / x-amz-meta 등 추가 가능
            .build();
        var presigned = presigner.presignPutObject(b -> b
            .signatureDuration(Duration.ofMinutes(uploadTtlMinutes))
            .putObjectRequest(putRequest)
        );
        return new PresignedUpload(presigned.url().toString(), bucket, key,
            Instant.now().plus(Duration.ofMinutes(uploadTtlMinutes)));
    }

    public String buildDownloadUrl(String key) {
        // CDN URL 이 있으면 우선 (캐시 + 비용 ↓)
        if (cdnBaseUrl != null && !cdnBaseUrl.isBlank()) {
            return cdnBaseUrl + "/" + key;
        }
        var getRequest = GetObjectRequest.builder().bucket(bucket).key(key).build();
        var presigned = presigner.presignGetObject(b -> b
            .signatureDuration(Duration.ofMinutes(downloadTtlMinutes))
            .getObjectRequest(getRequest)
        );
        return presigned.url().toString();
    }

    /** 업로드 완료 검증 — HEAD 로 실제 size / contentType 확인 */
    public HeadObjectResponse verify(String key) {
        try {
            return s3.headObject(HeadObjectRequest.builder().bucket(bucket).key(key).build());
        } catch (NoSuchKeyException e) {
            throw new BusinessException(ResponseCode.NOT_FOUND, "S3 에 파일이 없습니다.");
        }
    }

    public void deleteAsync(String key) {
        // 큰 파일은 lifecycle policy 로, 본 메서드는 즉시 삭제
        s3.deleteObject(DeleteObjectRequest.builder().bucket(bucket).key(key).build());
    }
}

public record PresignedUpload(String uploadUrl, String bucket, String key, Instant expiresAt) {}
```

```yaml
# application.yml
app:
  s3:
    bucket: ${S3_BUCKET}
    region: ap-northeast-2
    upload-url-ttl-minutes: 5
    download-url-ttl-minutes: 15
    cdn-base-url: ${CDN_BASE_URL:}
```

---

## 5. UseCase

### 5.1 IssueUploadUrlUseCase

```java
@Service
@RequiredArgsConstructor
public class IssueUploadUrlUseCase {

    private final FileMetaRepository files;
    private final S3FileService s3;
    private final IdGenerator ids;
    private final Clock clock;

    private static final long MAX_IMAGE_BYTES = 10L * 1024 * 1024;          // 10 MB
    private static final long MAX_VIDEO_BYTES = 500L * 1024 * 1024;         // 500 MB
    private static final Set<String> ALLOWED_IMAGE = Set.of(
        "image/jpeg", "image/png", "image/webp"
    );
    private static final Set<String> ALLOWED_VIDEO = Set.of("video/mp4", "video/quicktime");

    @Transactional
    public Result handle(UserId ownerId, FileMeta.Kind kind, String contentType, long sizeBytes,
                         String originalFilename) {
        validate(kind, contentType, sizeBytes);

        var fileId = new FileId(ids.next());
        var key = buildKey(kind, ownerId, fileId, originalFilename);

        var presigned = s3.buildUploadUrl(key, contentType, sizeBytes);
        var meta = FileMeta.initiate(fileId, ownerId, kind, presigned.bucket(), presigned.key(),
                                     contentType, sizeBytes, originalFilename, Instant.now(clock));
        files.save(meta);

        return new Result(fileId, presigned.uploadUrl(), presigned.expiresAt());
    }

    private void validate(FileMeta.Kind kind, String contentType, long sizeBytes) {
        if (sizeBytes <= 0) throw new BusinessException(ResponseCode.BAD_REQUEST, "유효하지 않은 파일 크기");

        boolean isImage = kind == FileMeta.Kind.PROFILE_IMAGE
            || kind == FileMeta.Kind.PRODUCT_IMAGE
            || kind == FileMeta.Kind.REVIEW_IMAGE;

        if (isImage) {
            if (!ALLOWED_IMAGE.contains(contentType))
                throw new BusinessException(ResponseCode.BAD_REQUEST, "허용되지 않는 이미지 형식");
            if (sizeBytes > MAX_IMAGE_BYTES)
                throw new BusinessException(ResponseCode.BAD_REQUEST, "이미지는 최대 10MB");
        } else if (kind == FileMeta.Kind.VIDEO) {
            if (!ALLOWED_VIDEO.contains(contentType))
                throw new BusinessException(ResponseCode.BAD_REQUEST, "허용되지 않는 동영상 형식");
            if (sizeBytes > MAX_VIDEO_BYTES)
                throw new BusinessException(ResponseCode.BAD_REQUEST, "동영상은 최대 500MB");
        }
        // ATTACHMENT 등은 별도 정책
    }

    private String buildKey(FileMeta.Kind kind, UserId ownerId, FileId fileId, String original) {
        // 키 패턴: kind/year/month/owner-fileId.ext
        var now = LocalDate.now();
        var ext = original != null && original.contains(".")
            ? original.substring(original.lastIndexOf('.'))     // ⚠️ 화이트리스트 검증 추가 권장
            : "";
        return String.format("%s/%d/%02d/%s/%s%s",
            kind.name().toLowerCase(),
            now.getYear(), now.getMonthValue(),
            ownerId.value(),
            fileId.value(),
            sanitizeExt(ext)
        );
    }

    private static String sanitizeExt(String ext) {
        if (ext == null || ext.isBlank()) return "";
        // 화이트리스트 확장자만
        return Set.of(".jpg", ".jpeg", ".png", ".webp", ".mp4", ".mov").contains(ext.toLowerCase())
            ? ext.toLowerCase()
            : "";
    }

    public record Result(FileId fileId, String uploadUrl, Instant expiresAt) {}
}
```

### 5.2 CompleteUploadUseCase

```java
@Service
@RequiredArgsConstructor
public class CompleteUploadUseCase {

    private final FileMetaRepository files;
    private final S3FileService s3;
    private final ApplicationEventPublisher events;
    private final Clock clock;

    @Transactional
    public void handle(FileId fileId, UserId currentUser) {
        var meta = files.findById(fileId)
            .orElseThrow(() -> new BusinessException(ResponseCode.NOT_FOUND, "파일"));
        if (!meta.ownerId().equals(currentUser))
            throw new BusinessException(ResponseCode.FORBIDDEN);

        // S3 에 실제 업로드 확인
        var head = s3.verify(meta.s3Key());
        meta.markCompleted(head.contentLength(), Instant.now(clock));
        files.save(meta);

        // 후처리 트리거 (이벤트 — thumbnail / EXIF strip / virus scan)
        events.publishEvent(new FileUploaded(fileId, meta.kind(), meta.s3Bucket(),
            meta.s3Key(), Instant.now(clock)));
    }
}

public record FileUploaded(FileId fileId, FileMeta.Kind kind, String bucket, String key,
                           Instant occurredAt) implements DomainEvent {}
```

→ S3 Event Notifications + Lambda 가 대체 trigger. 그러나 직접 complete API 가 클라 흐름 명확.

---

## 6. Controller

```java
@Tag(name = "파일 업로드")
@RestController
@RequestMapping("/api/v1/files")
@PreAuthorize("isAuthenticated()")
@RequiredArgsConstructor
public class FileController {

    private final IssueUploadUrlUseCase issue;
    private final CompleteUploadUseCase complete;
    private final GetFileUseCase getFile;

    @PostMapping("/presigned-url")
    public ResponseEntity<CommonResponse<IssueUploadUrlUseCase.Result>> presigned(
        @Valid @RequestBody PresignedRequest req, Authentication auth
    ) {
        var result = issue.handle(
            new UserId(auth.getName()),
            req.kind(),
            req.contentType(),
            req.sizeBytes(),
            req.originalFilename()
        );
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, result, "URL 발급"));
    }

    @PostMapping("/{id}/complete")
    public ResponseEntity<CommonResponse<Void>> complete(@PathVariable String id, Authentication auth) {
        complete.handle(new FileId(id), new UserId(auth.getName()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, "업로드 완료"));
    }

    @GetMapping("/{id}")
    public ResponseEntity<CommonResponse<FileDetailResponse>> get(@PathVariable String id, Authentication auth) {
        var detail = getFile.handle(new FileId(id), new UserId(auth.getName()));
        return ResponseEntity.ok(CommonResponse.success(ResponseCode.OK, detail));
    }
}

public record PresignedRequest(
    @NotNull FileMeta.Kind kind,
    @NotBlank String contentType,
    @Positive long sizeBytes,
    String originalFilename
) {}
```

---

## 7. CORS — S3 bucket 설정

PUT 을 브라우저에서 직접 하려면 S3 CORS:

```json
[{
  "AllowedOrigins": ["https://shop.example.com"],
  "AllowedMethods": ["PUT", "GET", "HEAD"],
  "AllowedHeaders": ["*"],
  "ExposeHeaders": ["ETag"],
  "MaxAgeSeconds": 3600
}]
```

---

## 8. 후처리 — Thumbnail / EXIF strip / Virus scan

### 8.1 Lambda 식 (S3 Event)

```
S3 PUT (uploads/...) → S3 Event Notification → Lambda
   ├─ 이미지: thumbnail 생성 (256/512/1024) + EXIF strip → thumbnails/...
   ├─ 동영상: ffmpeg poster + transcoded mp4 → videos/...
   └─ virus scan (ClamAV) → quarantine/...
```

### 8.2 Spring 내부 식 (Async listener)

```java
@Component
@RequiredArgsConstructor
public class FileUploadedListener {

    private final ImageProcessor imageProcessor;
    private final S3FileService s3;

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(FileUploaded event) {
        if (event.kind() != FileMeta.Kind.PRODUCT_IMAGE) return;
        try {
            var input = s3.downloadStream(event.bucket(), event.key());
            for (int size : List.of(256, 512, 1024)) {
                var resized = imageProcessor.resize(input, size);
                var thumbKey = event.key().replaceFirst("^[^/]+/", "thumbnails/" + size + "/");
                s3.upload(event.bucket(), thumbKey, resized, "image/webp");
            }
        } catch (Exception e) {
            log.error("thumbnail failed for {}", event.key(), e);
        }
    }
}
```

→ Spring 내부 async 는 인스턴스 부하 ↑. **Lambda 가 권장** (분리된 컴퓨트).

---

## 9. 함정 모음

### 함정 1 — multipart 로 서버 거침
heap OOM. 큰 파일 = 메모리 폭발. **presigned URL** 표준.

### 함정 2 — contentType 신뢰
클라가 `image/png` 라고 하지만 실제는 binary executable. **S3 verify 의 contentType** 또는 magic bytes 검사.

### 함정 3 — sizeBytes 신뢰
무한 업로드. presigned 의 `contentLength` 강제 + complete 시 HEAD 검증.

### 함정 4 — 파일명 확장자 그대로
`evil.exe` 가 그대로 노출. **화이트리스트** + 서버가 새 이름 생성.

### 함정 5 — bucket public read
모든 파일 인터넷 노출. **private bucket + presigned GET URL**.

### 함정 6 — presigned URL 너무 김
1시간+ = brute force 가능. 5분 (PUT) / 15분 (GET) 권장.

### 함정 7 — 같은 key 재사용
같은 사용자가 같은 파일명 두 번 = 덮어쓰기 / 충돌. **fileId 를 key 에 포함**.

### 함정 8 — DELETE 가 S3 즉시
다른 곳에서 참조 (CDN 캐시, 임시 URL) 시 깨짐. **soft delete + lifecycle**.

### 함정 9 — Lambda 후처리 무한 루프
Lambda 가 thumbnail 만들어서 같은 bucket 의 다른 prefix 에 PUT → 또 trigger? **S3 event filter** 로 prefix 분리.

### 함정 10 — EXIF 메타데이터 노출
사진의 GPS 위치 등 PII. 이미지는 항상 EXIF strip.

### 함정 11 — 직접 PUT 시 클라가 잘못된 contentType 전송
서버의 presigned 와 PUT 요청의 contentType 불일치 = SignatureMismatch. 클라에 명시.

---

## 10. 운영 체크리스트

- [ ] bucket private + presigned 만
- [ ] CloudFront / CDN 으로 download
- [ ] S3 CORS 설정 (도메인 화이트리스트)
- [ ] lifecycle policy — uploads/ 의 1일 이내 PENDING 파일 자동 삭제
- [ ] virus scan (큰 첨부 시)
- [ ] thumbnail Lambda 또는 별도 워커
- [ ] file_size / file_count 모니터 (사용자별)
- [ ] presigned URL 시간 짧게
- [ ] DELETE 시 soft + S3 lifecycle 결합
- [ ] S3 access log 활성

---

## 11. 관련

- [[review]] (예정) — review image 의 file 참조
- [[product-crud]] — 상품 이미지
- [[../common/security-config]] — CORS
- [[api-design|↑ api-design hub]]
