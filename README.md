# 🐳 도커 이미지 빌드 및 최적화 프로젝트

## 📌 프로젝트 개요 (Objective)

본 프로젝트는 Spring Boot 애플리케이션의 Docker 이미지 빌드 전략에 따른 인프라 효율성을 비교 분석합니다. 단일 스테이지(Single-stage) 빌드부터 Multi-stage, 경량화 JRE, 그리고 Layered JAR 기술을 적용한 모델까지 총 4가지 케이스를 구축하고 정량적 지표(이미지 크기, 빌드 시간)를 측정하여 최적의 CI/CD 파이프라인 구성 방안을 도출하는 것을 목표로 합니다.

## 🧪 실험 설계 및 핵심 코드 (Experiment Strategies)

동일한 Spring Boot 소스 코드를 바탕으로 아래 4가지 전략을 적용하여 Dockerfile을 구성했습니다.

### 1. Version 1: Single-stage (Naive JDK)

빌드와 실행을 하나의 컨테이너에서 처리합니다. 불필요한 소스 코드와 빌드 도구(JDK, Gradle 캐시 등)가 런타임 이미지에 모두 포함된 가장 무거운 대조군입니다.

Dockerfile

```jsx
# 무거운 JDK 기반으로 빌드와 실행을 한 번에 처리
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY . .
RUN ./gradlew clean build -x test
CMD ["java", "-jar", "build/libs/app.jar"]
```

### 2. Version 2: Multi-stage (JDK)

빌드 단계(Builder)와 실행 단계를 분리하여 최종 이미지에는 빌드된 JAR 파일만 포함되도록 구성했습니다. (단, 실행 환경은 여전히 무거운 JDK를 사용합니다.)

Dockerfile

```jsx
# --- Stage 1: Builder ---
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew clean build -x test

# --- Stage 2: Runtime (여전히 무거운 JDK 사용) ---
FROM eclipse-temurin:17-jdk
WORKDIR /app
COPY --from=builder /app/build/libs/app.jar app.jar
CMD ["java", "-jar", "app.jar"]
```

### 3. Version 3: Multi-stage + JRE Alpine (실행 환경 경량화)

Version 2의 구조에서 런타임 베이스 이미지를 초경량 `jre-alpine`으로 변경하여, OS 및 자바 실행 환경 자체의 리소스를 최소화했습니다.

Dockerfile

```jsx
# --- Stage 1: Builder (생략, V2와 동일) ---

# --- Stage 2: Runtime (초경량 JRE로 변경) ---
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/app.jar app.jar
CMD ["java", "-jar", "app.jar"]
```

### 4. Version 4: Multi-stage + JRE + Layered JAR (완성형)

Spring Boot의 `layertools`를 활용하여 JAR 파일을 4개의 레이어(Dependencies, Loader, Snapshot, Application)로 분해 후 조립합니다. 캐시 히트율(Cache Hit Ratio)을 극대화한 최종 모델입니다.

Dockerfile

```jsx
# --- Stage 1: Builder (레이어 추출 과정 추가) ---
FROM eclipse-temurin:17-jdk AS builder
WORKDIR /app
COPY . .
RUN ./gradlew clean build -x test
WORKDIR /app/extracted
# JAR 파일을 4개의 레이어로 분해
RUN java -Djarmode=layertools -jar ../build/libs/app.jar extract

# --- Stage 2: Runtime (캐시 효율을 위한 분할 복사) ---
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# 변경 빈도가 낮은 순서대로 복사하여 Docker 캐시 극대화
COPY --from=builder /app/extracted/dependencies/ ./
COPY --from=builder /app/extracted/spring-boot-loader/ ./
COPY --from=builder /app/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/extracted/application/ ./

ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

## 📊 측정 결과 (Results)

> **측정 조건:** 코드 1줄 수정 후 재빌드 시 소요되는 시간(Warm Build)과 최종 이미지 크기를 비교.
> 

| **전략 (Version)** | **베이스 이미지** | **최종 이미지 크기** |
| --- | --- | --- |
| **V1 (Single-stage)** | `eclipse-temurin:17-jdk` | **775MB** |
| **V2 (Multi-stage)** | `eclipse-temurin:17-jdk` | **488MB** |
| **V3 (JRE Alpine)** | `eclipse-temurin:17-jre-alpine` | **250MB** |
| **V4 (Layered JAR)** | `eclipse-temurin:17-jre-alpine` | **250MB** |

## 💡 결과 분석 및 인사이트 (Analysis)

### 1. 이미지 크기 최적화 (Storage & Security)

Version 1에서 Version 3로 넘어가며 이미지 크기가 극적으로 감소했습니다. 이는 불필요한 빌드 도구와 무거운 OS 패키지가 제거되었기 때문입니다. 용량 감소는 단순히 디스크 공간 절약뿐만 아니라, **보안 공격 표면(Attack Surface)을 줄이는 효과**를 가져옵니다.

### 2. 빌드 및 배포 속도 최적화 (Cache & Network)

Version 3와 Version 4는 최종 이미지 크기는 동일하지만, **재빌드(Warm Build) 시간**에서 압도적인 차이를 보였습니다.

- **Version 3 (Fat JAR):** 코드 한 줄만 수정되어도 수십 MB에 달하는 전체 JAR 파일의 해시값이 변경되어, Docker 캐시가 무효화되고 통째로 다시 빌드됩니다.
- **Version 4 (Layered JAR):** 용량이 큰 라이브러리(Dependencies) 레이어는 Docker 캐시를 100% 재사용하고, 변경된 1MB 미만의 애플리케이션(Application) 레이어만 새로 굽기 때문에 빌드가 순식간에 완료됩니다.

### 3. 결론 (Conclusion)

실제 CI/CD 파이프라인과 대규모 인프라 환경에서는 **Version 4 (Multi-stage + JRE Alpine + Layered JAR)** 전략이 필수적입니다. 이 방식은 실행 환경의 리소스를 최소화할 뿐만 아니라, 빈번한 배포 시 이미지 Push/Pull에 소모되는 네트워크 대역폭과 시간을 획기적으로 절약해 줍니다.
