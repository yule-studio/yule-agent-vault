---
title: "Jenkins — self-hosted enterprise"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:24:00+09:00
tags: [devops, cicd, jenkins]
---

# Jenkins — self-hosted enterprise

**[[cicd|↑ cicd]]**

---

## 1. 무엇

- self-hosted CI (Java).
- 플러그인 수천 개.
- enterprise 자체 인프라.
- declarative pipeline (Jenkinsfile).

---

## 2. Jenkinsfile (declarative)

```groovy
pipeline {
    agent any

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {
        IMAGE = "myapp:${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Test') {
            agent { docker 'eclipse-temurin:21-jdk' }
            steps {
                sh './gradlew test'
            }
            post {
                always { junit '**/build/test-results/test/*.xml' }
            }
        }

        stage('Build image') {
            steps {
                script {
                    docker.build(env.IMAGE)
                }
            }
        }

        stage('Push') {
            steps {
                script {
                    docker.withRegistry('https://ghcr.io', 'ghcr-creds') {
                        docker.image(env.IMAGE).push()
                    }
                }
            }
        }

        stage('Deploy prod') {
            when { branch 'main' }
            input message: 'Deploy?', ok: 'Yes'
            steps {
                sh "kubectl set image deploy/web app=${env.IMAGE}"
            }
        }
    }

    post {
        failure {
            slackSend channel: '#ci', message: "FAIL: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

---

## 3. agent

| agent | 사용 |
| --- | --- |
| `any` | 어떤 agent 든 |
| `none` | 각 stage 별 다른 |
| `docker '...'` | container 안 |
| `kubernetes` | k8s pod 동적 |
| `label 'gpu'` | 라벨 매칭 |

---

## 4. k8s plugin (동적 agent)

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                    - name: maven
                      image: maven:3.9
                      command: [cat]
                      tty: true
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }
    }
}
```

---

## 5. shared library

```
my-shared-lib/
└── vars/
    └── buildJavaApp.groovy
```

```groovy
// vars/buildJavaApp.groovy
def call(Map config) {
    sh "./gradlew bootJar"
    sh "docker build -t ${config.image} ."
}
```

```groovy
// Jenkinsfile
@Library('my-shared-lib') _
buildJavaApp(image: 'myapp:1.0')
```

---

## 6. 운영 권장

- master/agent 분리.
- agent 는 k8s 동적 (cost ↓).
- backup: $JENKINS_HOME 정기.
- LDAP / SAML SSO.
- Role-Based Access Control plugin.

---

## 7. 함정

1. **groovy script approval** — Jenkins admin 매번 승인 필요 (보안 vs 운영).
2. **plugin 호환성** — 업그레이드 시 깨짐.
3. **단일 master + 큰 워크로드** → 메모리 폭주.
4. **secret git commit** → Jenkins credentials store 사용.
5. **PR + script** — 외부 PR 의 Jenkinsfile 변경 → trusted PR만.

---

## 8. 관련

- [[cicd|↑ cicd]]
- [[tools-comparison]]
- [[pipeline-patterns]]
