[English](README.md)

# MStore Java SDK

| Sürüm | Tarih | Durum | İndirme |
|-------|-------|-------|---------|
| [v0.1.0](v0.1.0/) | 2026-03 | Son Sürüm | [v0.1.0/](v0.1.0/) |

## Kurulum (Maven)

```xml
<dependency>
    <groupId>com.moreum</groupId>
    <artifactId>mstore-client</artifactId>
    <version>0.1.0</version>
</dependency>
```

## Kullanım

```java
var client = MStoreClient.builder().endpoint("http://127.0.0.1:9011").build();
client.createBucket("my-bucket");
client.putObject("my-bucket", "hello.txt", data);
```

Gereksinimler: Java 17+
