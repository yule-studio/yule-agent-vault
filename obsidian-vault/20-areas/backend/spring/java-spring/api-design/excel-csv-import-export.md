---
title: "Excel / CSV Import·Export (스트리밍 + 검증)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - excel
  - csv
---

# Excel / CSV Import·Export (스트리밍 + 검증)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | POI streaming / OpenCSV / async job |

**[[api-design|↑ api-design hub]]**

> 📐 공통: [[../common/response-envelope]] · [[file-upload-s3]] (Import 의 업로드).

---

## 1. 무엇을 만드는가

```
# Import — 대량 등록 (셀러 / 관리자)
POST /api/v1/imports/{kind}            # 파일 upload (S3) + job 생성
GET  /api/v1/imports/{jobId}            # job 진행 상태 + 오류 list

# Export — 다운로드
GET  /api/v1/exports/{kind}?...        # 작은 데이터: 동기 다운로드 (스트리밍)
POST /api/v1/exports/{kind}            # 큰 데이터: async job + 완료 시 S3 URL
GET  /api/v1/exports/{jobId}            # job 상태 + download URL
```

### 1.1 용량별 전략

| 행 수 | 방식 |
| --- | --- |
| < 1000 | 동기 응답 (`StreamingResponseBody`) |
| 1000 ~ 100,000 | 동기 + 스트리밍 |
| 100,000+ | **async + S3** (job 모델) |

---

## 2. 의존성

```kotlin
// build.gradle.kts
implementation("org.apache.poi:poi-ooxml:5.2.5")             // Excel (xlsx)
implementation("com.opencsv:opencsv:5.9")                    // CSV
```

---

## 3. CSV Export — 스트리밍 (작은~중간)

```java
@RestController
@RequestMapping("/api/v1/exports")
@PreAuthorize("hasRole('ADMIN')")
@RequiredArgsConstructor
public class CsvExportController {

    private final OrderQueryRepository orders;

    @GetMapping("/orders.csv")
    public ResponseEntity<StreamingResponseBody> orders(
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate from,
        @RequestParam @DateTimeFormat(iso = DateTimeFormat.ISO.DATE) LocalDate to
    ) {
        StreamingResponseBody body = output -> {
            try (var writer = new OutputStreamWriter(output, StandardCharsets.UTF_8);
                 var csv    = new CSVWriter(writer)) {

                // UTF-8 BOM (엑셀 한글 깨짐 방지)
                writer.write('﻿');
                csv.writeNext(new String[]{"주문번호", "구매자", "금액", "상태", "주문일시"});

                orders.streamBetween(from, to, row -> {                  // Cursor / Stream
                    csv.writeNext(new String[]{
                        row.orderNumber(),
                        row.buyerName(),
                        String.valueOf(row.amountKrw()),
                        row.status(),
                        row.createdAt().toString()
                    });
                });
            }
        };
        return ResponseEntity.ok()
            .contentType(MediaType.parseMediaType("text/csv; charset=UTF-8"))
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"orders.csv\"")
            .body(body);
    }
}
```

### 3.1 Repository — Cursor stream

```java
public interface OrderQueryRepository {
    void streamBetween(LocalDate from, LocalDate to, Consumer<OrderExportRow> consumer);
}

@Repository
@RequiredArgsConstructor
public class JpaOrderQueryRepository implements OrderQueryRepository {

    private final EntityManager em;

    @Override
    @Transactional(readOnly = true)
    public void streamBetween(LocalDate from, LocalDate to, Consumer<OrderExportRow> consumer) {
        try (Stream<OrderJpaEntity> stream = em.createQuery(
                "from OrderJpaEntity o where o.createdAt between :from and :to",
                OrderJpaEntity.class)
            .setParameter("from", from.atStartOfDay().toInstant(ZoneOffset.UTC))
            .setParameter("to", to.atStartOfDay().plusDays(1).toInstant(ZoneOffset.UTC))
            .setHint(QueryHints.HINT_FETCH_SIZE, 1000)
            .getResultStream()
        ) {
            stream.forEach(e -> consumer.accept(toRow(e)));
        }
    }
}
```

→ JPA `getResultStream` + `fetch_size` 로 메모리 안 폭발.

---

## 4. Excel Export — POI SXSSF (Streaming)

```java
@GetMapping("/users.xlsx")
public void usersXlsx(HttpServletResponse response) throws IOException {
    response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    response.setHeader(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"users.xlsx\"");

    try (var workbook = new SXSSFWorkbook(100)) {       // 메모리 최근 100 행만
        var sheet = workbook.createSheet("Users");

        // 헤더
        var headerRow = sheet.createRow(0);
        var headers = new String[]{"ID", "이메일", "이름", "가입일"};
        for (int i = 0; i < headers.length; i++) {
            headerRow.createCell(i).setCellValue(headers[i]);
        }

        // 데이터 streaming
        var rowIdx = new int[]{1};
        users.streamAll(u -> {
            var r = sheet.createRow(rowIdx[0]++);
            r.createCell(0).setCellValue(u.id());
            r.createCell(1).setCellValue(u.email());
            r.createCell(2).setCellValue(u.name());
            r.createCell(3).setCellValue(u.createdAt().toString());
        });

        workbook.write(response.getOutputStream());
        workbook.dispose();                              // 임시 파일 정리
    }
}
```

> **`SXSSFWorkbook(100)`** — 최근 100 행만 메모리, 나머지는 디스크. **메모리 OOM 방지**. 100만 행 처리 가능.

> **함정**: `XSSFWorkbook` (non-streaming) 으로 100만 행 = OOM 확정. **무조건 SXSSF**.

---

## 5. Excel Export — Async Job (큰 데이터)

100,000+ 행은 응답 timeout 위험. **job 모델**.

```sql
CREATE TABLE export_jobs (
    id            CHAR(26) PRIMARY KEY,
    requested_by  CHAR(26) NOT NULL REFERENCES users(id),
    kind          VARCHAR(50) NOT NULL,       -- ORDERS / USERS / ...
    params        JSONB NOT NULL,
    status        VARCHAR(20) NOT NULL,       -- PENDING / RUNNING / SUCCEEDED / FAILED
    s3_key        VARCHAR(500),
    row_count     BIGINT,
    error         TEXT,
    requested_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    started_at    TIMESTAMPTZ,
    completed_at  TIMESTAMPTZ
);
```

```java
@Service
@RequiredArgsConstructor
public class ExportJobService {

    private final ExportJobRepository jobs;
    private final IdGenerator ids;
    private final Clock clock;
    private final TaskExecutor executor;          // @Async

    @Transactional
    public ExportJobId create(UserId requestedBy, String kind, Map<String, Object> params) {
        var job = new ExportJob(new ExportJobId(ids.next()), requestedBy,
            kind, params, "PENDING", null, null, null,
            Instant.now(clock), null, null);
        jobs.save(job);

        executor.execute(() -> runJob(job.id()));
        return job.id();
    }

    @Async
    public void runJob(ExportJobId jobId) {
        var job = jobs.findById(jobId).orElseThrow();
        job.markRunning(Instant.now());
        jobs.save(job);

        try {
            // 1. S3 multi-part upload — 직접 write
            var s3Key = "exports/" + jobId.value() + ".xlsx";
            var rowCount = writeExcelToS3(job.kind(), job.params(), s3Key);

            job.markSucceeded(s3Key, rowCount, Instant.now());
        } catch (Exception e) {
            job.markFailed(e.getMessage(), Instant.now());
            log.error("export job {} failed", jobId, e);
        }
        jobs.save(job);
    }
}
```

응답 (GET /exports/{jobId}):
```json
{
  "data": {
    "jobId": "...",
    "status": "SUCCEEDED",
    "rowCount": 320000,
    "downloadUrl": "https://cdn..../exports/....xlsx?X-Amz-..."
  }
}
```

→ presigned GET URL 로 다운로드 가능.

---

## 6. CSV Import — 한 줄씩 검증

```java
@RestController
@RequestMapping("/api/v1/imports")
@PreAuthorize("hasAnyRole('SELLER', 'ADMIN')")
@RequiredArgsConstructor
public class CsvImportController {

    private final ImportJobService service;

    /**
     * 클라가 [[file-upload-s3]] 로 먼저 S3 업로드 → fileId 전달.
     * 본 endpoint 는 job 만 생성.
     */
    @PostMapping("/products")
    public ResponseEntity<CommonResponse<Map<String, String>>> importProducts(
        @Valid @RequestBody ImportRequest req, Authentication auth
    ) {
        var jobId = service.create(new UserId(auth.getName()), "PRODUCT", req.fileId());
        return ResponseEntity.accepted().body(CommonResponse.success(ResponseCode.OK,
            Map.of("jobId", jobId.value()),
            "Import 시작. /imports/{jobId} 로 진행 확인."));
    }
}

public record ImportRequest(@NotBlank String fileId) {}
```

```java
@Service
@RequiredArgsConstructor
public class ProductImportJobRunner {

    private final S3FileService s3;
    private final ProductImportService productImport;
    private final ImportJobRepository jobs;
    private final ImportJobErrorRepository errors;

    @Async
    public void run(ImportJobId jobId) {
        var job = jobs.findById(jobId).orElseThrow();
        try (var input = s3.downloadStream(job.bucket(), job.s3Key());
             var reader = new InputStreamReader(input, StandardCharsets.UTF_8);
             var csvReader = new CSVReader(reader)) {

            csvReader.readNext();                  // header skip
            String[] row;
            int processed = 0, succeeded = 0, failed = 0;
            while ((row = csvReader.readNext()) != null) {
                processed++;
                try {
                    productImport.importRow(job.requestedBy(), row);
                    succeeded++;
                } catch (Exception e) {
                    failed++;
                    errors.save(new ImportError(ids.next(), jobId, processed,
                        e.getMessage(), Arrays.toString(row), Instant.now()));
                }
                // 진행률 — 100건마다 update
                if (processed % 100 == 0) {
                    job.updateProgress(processed, succeeded, failed);
                    jobs.save(job);
                }
            }
            job.markCompleted(processed, succeeded, failed, Instant.now());
        } catch (Exception e) {
            job.markFailed(e.getMessage(), Instant.now());
        }
        jobs.save(job);
    }
}
```

→ 행 단위 검증 + 오류 row 별도 테이블 저장. UI 에서 "Import 결과: 1000건 처리, 8건 실패 — 상세보기" 가능.

---

## 7. Import 결과 응답

```http
GET /api/v1/imports/01HZ-JOB...

200 OK
{
  "data": {
    "jobId": "01HZ...",
    "status": "SUCCEEDED",
    "progress": { "processed": 1000, "succeeded": 992, "failed": 8 },
    "errors": [
      { "row": 47, "message": "유효하지 않은 가격" },
      { "row": 123, "message": "존재하지 않는 카테고리: 'xyz'" },
      ...
    ]
  }
}
```

---

## 8. 보안 / 검증

- 권한 — SELLER 는 자기 상품, ADMIN 은 전체
- file size limit ([[file-upload-s3]] 의 50MB)
- CSV injection 방어 — cell 의 `=`, `+`, `-`, `@` 으로 시작하는 값은 quote
- 한글 encoding — UTF-8 + BOM (Excel 호환)
- formula 무력화 (Excel 의 `=SUM(...)` 같은 셀)

```java
private String safeCellValue(String v) {
    if (v != null && !v.isEmpty()) {
        var first = v.charAt(0);
        if (first == '=' || first == '+' || first == '-' || first == '@') {
            return "'" + v;                    // formula 무력화
        }
    }
    return v;
}
```

---

## 9. 함정 모음

### 함정 1 — `XSSFWorkbook` (non-streaming)
100k 행 = OOM. **SXSSFWorkbook(100)** 강제.

### 함정 2 — 동기 endpoint 로 큰 export
응답 timeout (30s) + LB drop. **async + S3 job**.

### 함정 3 — UTF-8 BOM 누락
Excel 에서 한글 깨짐. `﻿` prepend.

### 함정 4 — CSV injection
악의적 cell (`=cmd|'/c calc'!A1`) 이 Excel 에서 실행. quote prefix.

### 함정 5 — Stream 종료 X
`getResultStream` 의 connection 누수. try-with-resources.

### 함정 6 — Import 시 트랜잭션 하나
1000 행 = 한 트랜잭션 = lock 길게 + 한 실패 시 모두 rollback. **행 단위 트랜잭션** (실패 행만 skip + 기록).

### 함정 7 — 진행률 매 행 update
DB 부하. 100건마다.

### 함정 8 — file 검증 부족
악의적 사용자가 거대 파일 / formula bomb. size + row count 한계.

### 함정 9 — Excel 시간대
Excel 의 date = serial (1900-01-01 = 1). Java 의 `Instant` 와 변환 명시.

### 함정 10 — async job 결과 fetch 없음
사용자가 결과 확인 못 함. `GET /imports/{jobId}` 의 폴링 또는 push 알림.

---

## 10. 운영 체크리스트

- [ ] SXSSF 100 window
- [ ] UTF-8 BOM
- [ ] CSV injection 방어
- [ ] Stream try-with-resources
- [ ] Import 행 단위 트랜잭션
- [ ] async job 모델 + S3 + presigned download
- [ ] file size + row count 한계
- [ ] job 진행률 폴링 + 결과 push
- [ ] error 결과 별도 테이블 + UI

---

## 11. 관련

- [[file-upload-s3]] — Import 의 업로드
- [[../common/response-envelope]]
- 스케줄 job (예정)
- [[api-design|↑ api-design hub]]
