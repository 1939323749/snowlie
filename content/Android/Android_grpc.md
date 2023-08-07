---
title: "Android gRPC"
date: 2023-08-07T18:10:24+08:00
draft: false
---

# Android gRPC

Environment: Android Studio/Idea , Gradle

## Preparation

A proto file, a Android project

1. Put the proto file in the project, e.g. `app/src/main/proto/`
2. Add protobuf plugin in `build.gradle` of the project

```groovy
plugins {
    ...
    id 'com.google.protobuf' version '0.9.4'
}
protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.23.4"
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.56.0'
        }
    }
    generateProtoTasks {
        all().each { task ->
            task.builtins {
                java { option 'lite' }
            }
            task.plugins {
                grpc { option 'lite' }
            }
        }
    }
}
dependencies {
    ...
    implementation 'io.grpc:grpc-kotlin-stub:1.3.0'
    implementation 'io.grpc:grpc-protobuf-lite:1.56.1'
    implementation 'io.grpc:grpc-okhttp:1.56.1'
}
```

## Development

```bash
./gradlew generateDebugAndroidTestProto
```

## Reference

```kotlin
import proto.{SERVICE_NAME}Grpc
import proto.{SERVICE_NAME}OuterClass
```

## My Practice

[clipboard](https://github.com/1939323749/clipboard)

