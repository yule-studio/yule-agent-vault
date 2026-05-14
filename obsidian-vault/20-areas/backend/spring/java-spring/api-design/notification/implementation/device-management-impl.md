---
title: "Device 관리 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:58:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, device]
---

# Device 관리 구현 ★

**[[implementation|↑ hub]]**

---

## 1. Controller

```java
@RestController
@RequestMapping("/api/v1/me/devices")
@RequiredArgsConstructor
public class DeviceController {

    private final DeviceManagementService service;

    @PostMapping
    public ResponseEntity<DeviceResponse> register(
            @Valid @RequestBody RegisterDeviceRequest req,
            @AuthenticationPrincipal AuthUser auth) {
        var device = service.register(auth.userId(), req);
        return ResponseEntity.ok(DeviceResponse.from(device));
    }

    @PatchMapping("/{deviceId}")
    public ResponseEntity<Void> update(
            @PathVariable String deviceId,
            @Valid @RequestBody UpdateDeviceRequest req,
            @AuthenticationPrincipal AuthUser auth) {
        service.update(auth.userId(), DeviceId.of(deviceId), req);
        return ResponseEntity.ok().build();
    }

    @DeleteMapping("/{deviceId}")
    public ResponseEntity<Void> delete(
            @PathVariable String deviceId,
            @AuthenticationPrincipal AuthUser auth) {
        service.deactivate(auth.userId(), DeviceId.of(deviceId));
        return ResponseEntity.ok().build();
    }
}
```

---

## 2. Service (UPSERT)

```java
@Service
@RequiredArgsConstructor
public class DeviceManagementService {

    private final UserDeviceRepository repo;
    private final IdGenerator ids;
    private final Clock clock;

    @Transactional
    public UserDevice register(UserId user, RegisterDeviceRequest req) {
        // UPSERT — 같은 token 이면 user_id 갱신
        return repo.findByToken(req.token())
            .map(existing -> {
                existing.rotateToken(new DeviceToken(req.token(), req.channelType()),
                                      clock.now());
                if (!existing.userId().equals(user)) {
                    // device 양도 — 이전 user 의 token 무효
                    log.info("device transferred from {} to {}", existing.userId(), user);
                    existing.reassign(user);
                }
                return repo.save(existing);
            })
            .orElseGet(() -> repo.save(UserDevice.register(
                DeviceId.next(), user,
                req.platform(), req.channelType(),
                new DeviceToken(req.token(), req.channelType()),
                req.deviceId(), req.locale(), req.timezone(),
                clock.now())));
    }

    @Transactional
    public void deactivate(UserId user, DeviceId deviceId) {
        var device = repo.findById(deviceId).orElseThrow();
        if (!device.userId().equals(user)) throw new ForbiddenException();
        device.deactivate();
        repo.save(device);
    }
}
```

---

## 3. Cleanup cron

```java
@Scheduled(cron = "0 0 4 * * *")    // 매일 04:00
public void cleanupInactive() {
    var threshold = clock.now().minus(Duration.ofDays(90));
    int n = repo.deactivateInactiveBefore(threshold);
    log.info("deactivated {} inactive devices", n);
}
```

---

## 4. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/device-management]]
- [[../database/user-devices-table]]
- [[../security/device-token-rotation]]
