---
title: "Azure Files"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:40:00+09:00
tags:
  - azure
  - storage
  - files
---

# Azure Files

**[[storage|↑ Storage]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

매니지드 **SMB / NFS file share** — AWS EFS / GCP Filestore 동등. Windows / Linux / Mac mount.

---

## 2. 사용

```bash
# share 생성
az storage share-rm create \
  --resource-group myapp \
  --storage-account myappstorage \
  --name myshare \
  --quota 100
```

```hcl
resource "azurerm_storage_share" "share" {
  name                 = "myshare"
  storage_account_name = azurerm_storage_account.main.name
  quota                = 100               # GB
  enabled_protocol     = "SMB"             # 또는 NFS
}
```

---

## 3. Mount

Linux (SMB):
```bash
sudo mkdir /mnt/share
sudo mount -t cifs //myappstorage.file.core.windows.net/myshare /mnt/share \
  -o vers=3.0,credentials=/etc/cifs-creds,dir_mode=0755,file_mode=0644,serverino,nosharesock
```

Linux (NFS):
```bash
sudo mount -t nfs -o vers=4,minorversion=1,sec=sys myappstorage.file.core.windows.net:/myappstorage/myshare /mnt/share
```

Windows:
```powershell
net use Z: \\myappstorage.file.core.windows.net\myshare /user:myappstorage <key>
```

---

## 4. AKS 통합

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared
spec:
  accessModes: [ReadWriteMany]
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 100Gi
```

→ 여러 pod 가 같은 share mount.

---

## 5. Tier

| Tier | 의미 |
| --- | --- |
| Standard | HDD |
| Premium | SSD (low latency) |

---

## 6. 함정

- Linux SMB credential 권한 (0600)
- NFS = private endpoint 만 (public X)
- max share size = quota
- snapshot = 별도

---

## 7. 관련

- [[storage]]
- [[blob-storage]] — object 대안
