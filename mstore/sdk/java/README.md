[Turkce](README.tr.md)

# MStore Java SDK

| Version | Date | Status | Download |
|---------|------|--------|----------|
| [v0.1.0](v0.1.0/) | 2026-03 | Latest | [v0.1.0/](v0.1.0/) |

## Install (Maven)

```xml
<dependency>
    <groupId>com.moreum</groupId>
    <artifactId>mstore-client</artifactId>
    <version>0.1.0</version>
</dependency>
```

## Usage

```java
var client = MStoreClient.builder().endpoint("http://127.0.0.1:9011").build();
client.createBucket("my-bucket");
client.putObject("my-bucket", "hello.txt", data);
```

Requirements: Java 17+
