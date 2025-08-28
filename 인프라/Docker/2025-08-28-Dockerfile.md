# Dockerfile

# Dockerfile이란?

- Docker 컨테이너 이미지를 생성하기 위한 텍스트 기반 설정 파일
- 컨테이너 내부 환경을 설정하는 일련의 명령어들이 포함되어 있음

## Dockerfile 기본 구조와 주요 명령어

1. FROM: 베이스 이미지를 지정(모든 Dockerfile은 FROM 명령으로 시작)
    
    ```dockerfile
    # 베이스 이미지로 JDK를 포함한 이미지 사용
    FROM eclipse-temurin:17-jdk
    ```
    
2. WORKDIR: 작업 디렉토리 설정
    
    ```dockerfile
    # 작업 디렉토리 설정
    WORKDIR /app
    ```
    
3. COPY / ADD: 파일이나 디렉토리를 호스트에서 컨테이너로 복사
    
    ```dockerfile
    # 그래들/메이븐 빌드 파일 복사
    COPY build.gradle settings.gradle gradlew ./
    COPY gradle ./gradle
    ```
    
4. RUN: 컨테이너 내에서 명령어를 실행(주로 패키지 설치나 설정에 사용)
    
    ```dockerfile
    # 의존성 다운로드를 위한 레이어 캐싱 활용
    RUN ./gradlew dependencies --no-daemon
    
    # 애플리케이션 빌드
    RUN ./gradlew build --no-daemon -x test
    ```
    
5. ENV: 환경변수를 설정
6. EXPOSE: 컨테이너가 특정 포트에서 수신 대기함을 나타냄
    
    ```dockerfile
    # 스프링 부트 애플리케이션이 사용할 포트 설정
    EXPOSE 8080
    ```
    
7. CMD / ENTRYPOINT: 컨테이너가 시작될 때 실행할 명령어를 지정
    
    ```dockerfile
    # 빌드된 JAR 파일 실행
    ENTRYPOINT ["java", "-jar", "/app/build/libs/myapp-0.0.1-SNAPSHOT.jar"]
    ```
    

## Dockerfile을 사용한 이미지 빌드 방법

```dockerfile
docker build -t my-app:1.0 .
```

## 일반적인 다단계 빌드 방식(스프링)

```dockerfile
# 빌드 스테이지: JDK를 포함한 이미지 사용
FROM eclipse-temurin:17-jdk AS build

# 작업 디렉토리 설정
WORKDIR /app

# Gradle 래퍼와 빌드 파일 복사
COPY gradlew .
COPY gradle gradle
COPY build.gradle .
COPY settings.gradle .

# Gradle 래퍼에 실행 권한 부여
RUN chmod +x ./gradlew

# 의존성 다운로드 (이 단계는 캐싱됨)
RUN ./gradlew dependencies --no-daemon

# 소스 코드 복사
COPY src src

# 애플리케이션 빌드
RUN ./gradlew build --no-daemon -x test

# 실행 스테이지: JRE만 포함한 이미지 사용 (이미지 크기 최소화)
FROM eclipse-temurin:17-jre

WORKDIR /app

# 빌드 스테이지에서 생성된 JAR 파일만 복사
COPY --from=build /app/build/libs/*.jar app.jar

# 애플리케이션 포트 설정
EXPOSE 8080

# 컨테이너 실행 시 JAR 파일 실행
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 간단한 버전

- 미리 빌드된 JAR 파일을 사용하는 방식
- CI/CD 파이프라인에서 빌드 단계와 이미지 생성 단계를 분리할 때 자주 사용

```dockerfile
# 빌드 스테이지: JDK를 포함한 이미지 사용
FROM openjdk:17-jdk-slim

# 작업 디렉토리 설정
WORKDIR /app

# 빌드 인자 정의 (기본값은 local)
ARG SPRING_PROFILES_ACTIVE=local

# 환경 변수로 설정
ENV SPRING_PROFILES_ACTIVE=${SPRING_PROFILES_ACTIVE}

# 빌드 스테이지에서 생성된 JAR 파일만 복사
COPY build/libs/*.jar app.jar

# Spring Boot 실행 명령어
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 헷갈렸던 점

`ENTRYPOINT`와 `CMD`의 차이점

- **ENTRYPOINT:** 컨테이너 실행 시 **항상 실행되는** 고정 명령어
- **CMD:** 컨테이너 실행 시 **덮어쓸 수 있는** 기본 명령어

**CMD**는 추천 사용법이나 기본 설정 예시를 보여주는 느낌이고, 사용자가 원하면 언제든 바꿀 수 있다.