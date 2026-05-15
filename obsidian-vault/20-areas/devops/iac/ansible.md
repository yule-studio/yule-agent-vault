---
title: "Ansible — agentless config 관리"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:14:00+09:00
tags: [devops, iac, ansible]
---

# Ansible — agentless config 관리

**[[iac|↑ iac]]**

---

## 1. 무엇

- agentless (SSH 기반).
- YAML playbook.
- idempotent (반복 실행 안전).
- 사용: VM 설정 / package install / config 변경 / app deploy.

---

## 2. inventory

```ini
# inventory.ini
[web]
web1.example.com
web2.example.com

[db]
db1.example.com

[prod:children]
web
db
```

---

## 3. playbook

```yaml
# install-nginx.yml
- hosts: web
  become: true
  vars:
    nginx_version: "1.27"
  tasks:
    - name: Update apt
      apt: {update_cache: true}

    - name: Install nginx
      apt: {name: "nginx={{ nginx_version }}*", state: present}

    - name: Configure nginx
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        mode: '0644'
      notify: Reload nginx

    - name: Ensure running
      service: {name: nginx, state: started, enabled: true}

  handlers:
    - name: Reload nginx
      service: {name: nginx, state: reloaded}
```

```bash
ansible-playbook -i inventory.ini install-nginx.yml
```

---

## 4. role (재사용)

```
roles/
└── nginx/
    ├── tasks/main.yml
    ├── handlers/main.yml
    ├── templates/nginx.conf.j2
    ├── vars/main.yml
    └── defaults/main.yml
```

```yaml
# playbook
- hosts: web
  roles:
    - nginx
    - { role: postgres, postgres_version: 16 }
```

---

## 5. Galaxy (community roles)

```bash
ansible-galaxy install geerlingguy.nginx
```

---

## 6. ansible-vault (secret)

```bash
ansible-vault encrypt secrets.yml
ansible-playbook --ask-vault-pass playbook.yml
```

---

## 7. Ansible vs Terraform

| | Ansible | Terraform |
| --- | --- | --- |
| state | X | O |
| 사용 | 서버 config | 인프라 자원 |
| 통신 | SSH | API |
| 둘 다 사용 | OK — TF 가 VM 생성, Ansible 이 config |

---

## 8. 함정

1. **secret git commit** → ansible-vault.
2. **idempotent X 한 task** (shell command) → 반복 실패.
3. **inventory 의 IP / hostname 변경** → DNS / template.
4. **--check (dry-run) 안 함** → 실수.
5. **SSH key 관리** — agent / vault.

---

## 9. 관련

- [[iac|↑ iac]]
- [[tools-comparison]]
- [[../linux/linux|↗ linux]]
