---
title: "Factory Method — Java / Spring"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, factory-method, java, spring, jdbc]
---

# Factory Method — Java / Spring

**[[factory-method|↑ Factory Method hub]]**

## 1. 기본 Java

```java
interface Document {
    void open();
}
class PdfDoc implements Document {
    public void open() { /* ... */ }
}
class WordDoc implements Document {
    public void open() { /* ... */ }
}

abstract class DocumentCreator {
    abstract Document createDoc();        // factory method
    void openDoc() { createDoc().open(); }
}

class PdfCreator extends DocumentCreator {
    Document createDoc() { return new PdfDoc(); }
}
class WordCreator extends DocumentCreator {
    Document createDoc() { return new WordDoc(); }
}
```

## 2. Static factory method (Joshua Bloch Effective Java)

```java
public class User {
    private final String name;
    private User(String n) { this.name = n; }

    // factory methods — 의미를 이름으로
    public static User fromEmail(String email) { ... }
    public static User fromUsername(String name) { ... }
    public static User guest() { ... }
}
```

- constructor 보다 **이름이 있는** factory method.
- caching / pooling 가능 (`Integer.valueOf` 가 이렇게 -128~127 cache).

## 3. Spring `@Bean` 메서드

```java
@Configuration
class DataSourceConfig {
    @Bean
    public DataSource dataSource(@Value("${db.url}") String url) {
        return DataSourceBuilder.create()
            .url(url)
            .build();
    }
}
```

- `@Bean` 메서드 = Spring 의 factory method.
- 환경 / 프로필 별 다른 구현 주입 (예: `@Profile("dev")`).

## 4. JDBC `DriverManager.getConnection` — 표준 예

```java
String url = "jdbc:postgresql://...";
Connection conn = DriverManager.getConnection(url);
// driver = url scheme 으로 결정 (postgresql / mysql / oracle)
```

- URL 의 prefix 가 어떤 driver 의 객체를 만들지 결정.
- 클래식 factory method 의 industry 표준.

## 5. Spring `FactoryBean` interface

```java
public class MyServiceFactoryBean implements FactoryBean<MyService> {
    @Override
    public MyService getObject() {
        // 복잡한 초기화 로직
        return new MyService(...);
    }
    @Override
    public Class<?> getObjectType() { return MyService.class; }
}
```

- bean 의 생성 로직을 별도 class 로 분리.
- Spring 안 ProxyFactoryBean, EncryptablePropertyResolver 등.

## 6. 함정

1. **factory 가 if-else / switch 폭주** — 객체 종류 추가마다 factory 수정. 등록 기반 (registry) 으로.
2. **Reflection factory** — `Class.forName + newInstance` 가 type safety 잃음.
3. **`@Bean` 메서드의 self-invocation** — 같은 class 안 `@Bean` 메서드끼리 호출 시 proxy 우회. constructor injection 권장.

## 7. 관련

- [[factory-method]]
- [[factory-method-python]]
- [[factory-method-typescript]]
- [[factory-method-go]]
- [[../abstract-factory/abstract-factory]]
