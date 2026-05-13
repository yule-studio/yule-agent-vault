---
title: "D-Bus — Desktop Bus"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:40:00+09:00
tags:
  - operating-system
  - ipc
  - dbus
---

# D-Bus

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | D-Bus 개념 + 사용 |

**[[ipc|↑ IPC hub]]**

---

## 1. 한 줄

Linux 데스크탑 / 시스템 서비스의 표준 IPC. Unix socket 위에 **메시지 + RPC + signal** 추상화.

---

## 2. 두 가지 bus

| Bus | 의미 |
| --- | --- |
| **System bus** | OS 전체 (hardware, network, systemd) |
| **Session bus** | 로그인 사용자 별 (데스크탑 앱) |

`dbus-daemon` 이 router.

---

## 3. 구조

```
Service (well-known name: org.freedesktop.NetworkManager)
  Object Path (/org/freedesktop/NetworkManager)
    Interface (org.freedesktop.NetworkManager)
      Method  (GetDevices, ...)
      Property (Version, ...)
      Signal  (DeviceAdded, ...)
```

→ 객체지향 RPC 비슷.

---

## 4. CLI — busctl / dbus-send

```bash
# 시스템 bus
busctl --system
busctl --system list

busctl --system call org.freedesktop.NetworkManager \
  /org/freedesktop/NetworkManager \
  org.freedesktop.NetworkManager GetDevices

# Session bus
busctl --user list
busctl --user introspect org.freedesktop.Notifications /org/freedesktop/Notifications

# dbus-send
dbus-send --system --dest=org.freedesktop.systemd1 \
  /org/freedesktop/systemd1 \
  org.freedesktop.systemd1.Manager.ListUnits
```

---

## 5. systemd 와의 통합

systemd 가 D-Bus 의 큰 사용자:

```bash
# systemctl 도 내부적으로 D-Bus
systemctl status nginx       # = busctl call ... systemd1.Manager.GetUnit
busctl call org.freedesktop.systemd1 /org/freedesktop/systemd1 \
  org.freedesktop.systemd1.Manager StartUnit ss "nginx.service" "replace"
```

자세히 → [[../linux/systemd]] (작성 예정)

---

## 6. PolicyKit / 권한

PolicyKit (polkit) = D-Bus 위에 사용자 권한 / 인증 정책.

```
사용자가 D-Bus method 호출
  ↓
PolicyKit 검사 (action ID, 사용자, 컨텍스트)
  ↓
허용 / 거부 / 인증 요구 (sudo 같은 GUI prompt)
```

---

## 7. 응용 예

| 서비스 | D-Bus name |
| --- | --- |
| systemd | org.freedesktop.systemd1 |
| NetworkManager | org.freedesktop.NetworkManager |
| logind | org.freedesktop.login1 |
| Notifications | org.freedesktop.Notifications |
| UPower | org.freedesktop.UPower |
| BlueZ (Bluetooth) | org.bluez |
| MPRIS (음악 플레이어) | org.mpris.MediaPlayer2.* |
| GNOME Settings | org.gnome.* |

---

## 8. 응용 코드 (Python)

```python
import dbus

bus = dbus.SystemBus()
obj = bus.get_object('org.freedesktop.systemd1', '/org/freedesktop/systemd1')
mgr = dbus.Interface(obj, 'org.freedesktop.systemd1.Manager')
print(mgr.ListUnits())
```

GLib (`Gio.DBusProxy`), Qt (`QDBusInterface`), Rust (`zbus`), Go 등 라이브러리.

---

## 9. Signal subscribe

```python
def on_added(path, ifaces):
    print("new device:", path)

bus.add_signal_receiver(on_added,
    signal_name='InterfacesAdded',
    dbus_interface='org.freedesktop.DBus.ObjectManager',
    bus_name='org.freedesktop.NetworkManager')
```

→ 시스템 이벤트 구독.

---

## 10. Wire Format

- binary 메시지
- header + body
- type signature (`s`, `i`, `as` 등)
- introspection XML

→ TCP / WS 와 다름. CLI 도구 (busctl) 가 추상.

---

## 11. 한계

- 같은 호스트만
- 거대 throughput 부적합 (~10 K msg/s)
- learning curve
- 분산 / 다른 호스트 → 외부 broker

→ **데스크탑 / OS 서비스 통합** 에 최적. 응용 ↔ 응용 일반 IPC 는 socket 권장.

---

## 12. kdbus / bus1 (실패)

D-Bus 를 kernel 안에 (성능). 결국 mainline 채택 X — 외부 dbus-broker 가 빠르게 발전.

---

## 13. 함정

### 13.1 D-Bus 가 모든 IPC 해결 X
무겁다. 일반 응용은 socket / shm.

### 13.2 시스템 vs 세션 혼동
`--system` / `--user` (busctl).

### 13.3 권한
PolicyKit 정책 검토. root method 의도치 않게 호출.

### 13.4 signal subscribe 누락
loop 시작 (GLib mainloop) 안 하면 안 받음.

### 13.5 async 모델
GLib / Qt event loop 와 결합. blocking call 함정.

### 13.6 introspection XML 잘못
잘못된 type signature → 호출 실패.

---

## 14. 학습 자료

- **D-Bus Specification** — freedesktop.org/wiki/Software/dbus
- **systemd** docs
- `busctl(1)`
- **GIO D-Bus** 튜토리얼

---

## 15. 관련

- [[unix-socket]] — 토대
- [[../linux/systemd]] (작성 예정)
- [[ipc]] — IPC hub
