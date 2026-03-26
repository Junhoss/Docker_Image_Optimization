# 🐳 도커 이미지 빌드 및 최적화 프로젝트

## 🧑‍💻 팀원 소개
| ![](https://avatars.githubusercontent.com/u/50224952?v=4) | ![](https://avatars.githubusercontent.com/u/204296918?v=3) |
|:---:|:---:|
| **이명진**<br>[@septeratz](https://github.com/septeratz) | **이준호**<br>[@Junhoss](https://github.com/Junhoss) |
<br/>


## 📌 프로젝트 개요 (Objective)

**Docker 이미지 빌드 전략 비교 분석 프로젝트**

본 프로젝트는 Spring Boot 애플리케이션의 Docker 이미지 빌드 전략에 따른 인프라 효율성을 비교 분석합니다.

**분석 대상 4가지 케이스**

- Single-stage 빌드 (기본)
- Multi-stage 빌드
- 경량화 JRE 적용
- Layered JAR 기술 적용

단일 스테이지(Single-stage) 빌드부터 Multi-stage, 경량화 JRE, 그리고 Layered JAR 기술을 적용한 모델까지 총 4가지 케이스를 구축하고 정량적 지표(이미지 크기, 빌드 시간)를 측정하여 최적의 CI/CD 파이프라인 구성 방안을 도출하는 것을 목표로 합니다.

## 🛠 실험 환경 (Environment)

본 실험은 다음과 같은 시스템 및 기술 스택 위에서 진행되었습니다.

- **Infrastructure (OS):** Ubuntu 20.04.1 LTS
- **Language:** Java 17
- **Framework:** Spring Boot 3.x
- **Build Tool:** Gradle (Wrapper)
- **Containerization:** Docker Engine
- **Base Images Used:**
    - `eclipse-temurin:17-jdk` (빌드 환경 및 대조군 런타임용)
    - `eclipse-temurin:17-jre-alpine` (최적화 런타임용 초경량 이미지)

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

| **전략 (Version)** | **베이스 이미지** | **최종 이미지 크기** | **Warm Build 시간** | **보안/경량화** |
| --- | --- | --- | --- | --- |
| **V1 (Single-stage)** | `eclipse-temurin:17-jdk` | 775MB | 약 80초 | 최하 (JDK, 소스코드 포함) |
| **V2 (Multi-stage)** | `eclipse-temurin:17-jdk` | 488MB | 약 80초 | 하 (JDK 포함) |
| **V3 (JRE Alpine)** | `eclipse-temurin:17-jre-alpine` | **250MB**  | 약 80초 | 상 (JRE Alpine) |
| **V4 (Layered JAR)** | `eclipse-temurin:17-jre-alpine` | **250MB** | **약 5초** | 최상 (JRE Alpine) |


## 💡 결과 분석 및 인사이트 (Analysis)

### 1. 빌드 및 배포 속도 최적화 (Cache & Network)

V3와 V4는 최종 이미지 크기가 250MB로 동일하지만, **재빌드(Warm Build) 시간**에서 압도적인 차이를 보였습니다. Layered JAR를 적용한 V4 전략이 비효율적인 V3 전략 대비 **약 93%의 빌드 시간 단축 효과(16배 속도 향상)**를 기록했습니다.

- **V3 (Fat JAR):** 코드가 단 1줄만 수정되어도 수십 MB에 달하는 전체 JAR 파일의 해시값이 변경됩니다. 이로 인해 Docker의 기존 캐시가 무효화되고 통째로 다시 빌드하면서 약 80초의 시간이 낭비되었습니다.
- **V4 (Layered JAR):** 애플리케이션을 4개의 레이어로 분리한 결과, 용량이 가장 큰 라이브러리(Dependencies) 영역은 Docker가 기존 캐시를 `0.0s` 만에 그대로 재사용했습니다. 오직 코드가 수정된 1MB 미만의 `application/` 레이어만 새로 구워내어 약 5초 만에 빌드가 완료되었습니다.

### 2. 이미지 크기 최적화 (Storage)

V1(775MB)에서 V3/V4(250MB)로 최적화하면서 최종 이미지 용량이 약 **3분의 1 수준**으로 감소했습니다. 불필요한 빌드 도구와 무거운 OS 패키지를 제거하여 디스크 공간을 절약했을 뿐만 아니라, 런타임 환경을 경량화하여 컨테이너 실행 효율을 극대화했습니다.

### 3. 결론 (Conclusion)

지속적 통합 및 배포(CI/CD) 파이프라인에서 잦은 코드 변경이 일어나는 환경이라면, **V4 (Multi-stage + JRE Alpine + Layered JAR)** 전략은 선택이 아닌 필수입니다. 이 방식은 컨테이너의 실행 리소스를 최소화할 뿐만 아니라, 극적인 빌드 시간 단축과 네트워크 트래픽 절감을 가능하게 합니다.
