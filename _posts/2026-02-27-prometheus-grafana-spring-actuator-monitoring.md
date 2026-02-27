---
layout: post
published: true
title: Prometheus + Grafana + Spring Actuator 서버 시스템 메트릭 모니터링하기
categories: Infra
date: 2026-02-27
banner:
  image: https://github.com/user-attachments/assets/1cf1d72e-31c9-4e5e-af16-1b66e13e2166
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-weight: bold;"
  subheading_style: "color: gold"
tags: Spring-Boot Actuator Prometheus Grafana Micrometer Docker Monitoring
sidebar: []
comments: true
---

서비스를 운영하다 보면 "서버가 느려졌는데 원인을 모르겠다"는 상황을 자주 마주칩니다.
CPU 사용률, 메모리, API 응답 시간 같은 시스템 메트릭을 실시간으로 확인할 수 있다면 장애 원인을 훨씬 빠르게 파악할 수 있습니다.

이번 글에서는 **Prometheus + Grafana + Spring Actuator** 조합으로 Spring Boot 애플리케이션의 시스템 메트릭을 수집하고 시각화하는 모니터링 환경을 구축한 경험을 공유합니다.

## 도입 배경

모니터링 시스템 도입을 통해 다음 네 가지 목표를 달성하고자 했습니다:

1. **메트릭 가시화를 통한 성능 모니터링 기반 마련**: CPU, 메모리, JVM 힙, GC 등 애플리케이션 핵심 메트릭을 실시간 대시보드로 시각화하여 시스템 상태를 즉시 파악할 수 있는 환경을 구성합니다.
2. **API 응답 속도(Latency) 분석 체계 구축**: API 엔드포인트별 응답 시간, 요청 빈도, 에러율을 한눈에 확인할 수 있도록 정리하여 성능 병목 구간을 빠르게 식별하고 대응할 수 있는 기반을 마련합니다.
3. **다중 서버 통합 모니터링**: 여러 서버 인스턴스의 메트릭을 하나의 모니터링 시스템에서 수집하고, 서버 간 부하 분포를 비교·분석할 수 있는 통합 관제 환경을 구성합니다.
4. **장애 조기 감지를 위한 확장 기반 마련**: Grafana의 알림(Alert) 기능과 연계하여, 메트릭 임계값 초과 시 장애 징후를 조기에 감지할 수 있는 확장 기반을 마련합니다.

---

## 아키텍처

### 전체 흐름

![모니터링 아키텍처](https://github.com/user-attachments/assets/1cf1d72e-31c9-4e5e-af16-1b66e13e2166)

### 각 컴포넌트의 역할

| 컴포넌트 | 역할 |
|----------|------|
| **Spring Actuator** | 애플리케이션 메트릭 엔드포인트 노출 (`/actuator/prometheus`) |
| **Micrometer** | 메트릭을 Prometheus 포맷으로 변환하는 어댑터 |
| **Prometheus** | Pull 방식으로 메트릭을 주기적으로 수집 & 시계열 DB에 저장 |
| **Grafana** | 수집된 메트릭을 대시보드로 시각화, 임계값 기반 알림 설정 |

**Pull 방식**이란 Prometheus가 설정된 간격(기본 15초)마다 Spring 애플리케이션의 `/actuator/prometheus` 엔드포인트를 직접 호출하여 메트릭을 가져오는 구조입니다. 애플리케이션이 메트릭을 보내는 Push 방식과 달리, 모니터링 대상이 증가해도 Prometheus 설정만 추가하면 되므로 확장성이 좋습니다.

---

## 구현 가이드

### Step 1: Spring Actuator + Micrometer 설정

먼저 Spring Boot 애플리케이션에 메트릭 수집 기능을 추가합니다.

**build.gradle**:

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator:2.7.0'
    implementation 'io.micrometer:micrometer-registry-prometheus:1.10.3'
}
```

**application.yml**:

```yaml
management:
  metrics:
    tags:
      application: my-api  # Grafana 대시보드에서 애플리케이션 식별용
    export:
      prometheus:
        enabled: true
    enable:
      http.server.requests: true
    distribution:
      percentiles-histogram:
        http.server.requests: false
  endpoints:
    web:
      exposure:
        include: prometheus  # prometheus 엔드포인트만 노출
```

설정 후 애플리케이션을 실행하면 `http://localhost:8080/actuator/prometheus`에서 메트릭을 확인할 수 있습니다.

### Step 2: URI Cardinality Explosion 방지

Spring Actuator의 HTTP 메트릭은 기본적으로 요청 URI를 태그로 기록합니다. 그런데 `/api/users/1`, `/api/users/2`처럼 Path Variable이 포함된 URI는 각각 별도 메트릭으로 기록되어 **Cardinality Explosion**(메트릭 폭발)이 발생할 수 있습니다.

이를 방지하기 위해 정규화된 URI 템플릿(`/api/users/{id}`)을 사용하도록 `WebMvcTagsProvider`를 커스터마이징합니다.

**TimerConfig.java**:

```java
@Configuration
public class TimerConfig {

    @Bean
    public WebMvcTagsProvider webMvcTagsProvider() {
        return new DefaultWebMvcTagsProvider() {
            @Override
            public Iterable<Tag> getTags(HttpServletRequest request,
                                         HttpServletResponse response,
                                         Object handler, Throwable exception) {
                // URI Cardinality Explosion 방지를 위해 정규화된 URI 템플릿 사용
                String uriTemplate = (String) request.getAttribute(
                    HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
                if (uriTemplate == null) {
                    uriTemplate = request.getRequestURI(); // fallback
                }

                return List.of(
                    Tag.of("uri", uriTemplate),
                    Tag.of("method", request.getMethod()),
                    Tag.of("status", String.valueOf(response.getStatus())),
                    Tag.of("exception", exception == null
                        ? "None" : exception.getClass().getSimpleName()),
                    Tag.of("outcome", response.getStatus() >= 200
                        && response.getStatus() < 300 ? "SUCCESS" : "ERROR")
                );
            }
        };
    }

}
```

### Step 3: Actuator 엔드포인트 보안 설정

Actuator 엔드포인트는 애플리케이션의 민감한 정보를 노출하므로, 반드시 접근 제어가 필요합니다.

**SecurityConfig.java** (내부 네트워크에서만 접근 허용):

```java
@Configuration
@RequiredArgsConstructor
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    public void configure(WebSecurity web) throws Exception {
        web.ignoring().antMatchers("/actuator/**");
    }
}
```

> **보안 주의**: Spring Security에서 `/actuator/**`를 무시(ignoring)하더라도, 반드시 Nginx 등 웹 서버 레벨에서 외부 접근을 차단해야 합니다. Actuator 엔드포인트가 외부에 노출되면 중대한 보안 사고가 발생할 수 있습니다. `{hostUrl}/actuator`를 호출하면 현재 오픈된 Actuator 엔드포인트 목록을 확인할 수 있습니다.

![Actuator 엔드포인트 목록](https://github.com/user-attachments/assets/095bd410-606a-4fe6-b90d-786bdf0effad)

**Nginx - Actuator 엔드포인트 외부 차단**:

```nginx
location ~ ^/actuator(/.*)?$ {
    deny all;
}
```

Prometheus는 내부 네트워크(Private IP)를 통해 직접 메트릭을 수집하므로, Nginx를 경유하지 않아 이 차단 규칙에 영향을 받지 않습니다.

### Step 4: Prometheus + Grafana 설치 (Docker Compose)

모니터링 서버에 Prometheus와 Grafana를 Docker Compose로 설치합니다.

**디렉토리 구조**:

```
/opt/monitoring
 ├─ docker-compose.yml          # Prometheus + Grafana 컨테이너 정의
 ├─ prometheus/
 │   ├─ prometheus.yml           # 수집 대상(targets), 주기 등 설정
 │   └─ data/                    # 시계열 메트릭 저장소 (볼륨 마운트)
 └─ grafana/
     ├─ data/                    # 대시보드, 사용자 설정 등 영구 저장소
     └─ provisioning/
         ├─ dashboards/          # 대시보드 자동 프로비저닝 JSON
         └─ datasources/         # 데이터 소스 자동 프로비저닝 YAML
```

**Docker 설치** (Amazon Linux 2 기준):

```bash
sudo yum update -y

# Docker 설치
sudo amazon-linux-extras install docker -y
sudo service docker start
sudo usermod -aG docker ec2-user

# Docker Compose 설치
sudo curl -L \
 https://github.com/docker/compose/releases/download/v2.29.2/docker-compose-linux-x86_64 \
 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

**docker-compose.yml**:

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.54.0
    container_name: prometheus
    user: "1000:1000"                # 호스트 사용자 권한으로 실행 (볼륨 권한 이슈 방지)
    restart: always                  # 컨테이너 비정상 종료 시 자동 재시작
    networks:
      - monitoring-net
    ports:
      - "9094:9090"                  # 호스트 9094 → 컨테이너 9090 포트 매핑
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro  # 설정 파일 (읽기 전용)
      - ./prometheus/data:/prometheus                                  # 메트릭 저장소
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"   # 메트릭 보관 기간
      - "--storage.tsdb.retention.size=4GB"   # 최대 저장 용량
      - "--web.enable-admin-api"              # 관리 API 활성화 (스냅샷, 삭제 등)
      - "--web.console.libraries=/usr/share/prometheus/console_libraries"
      - "--web.console.templates=/usr/share/prometheus/consoles"

  grafana:
    image: grafana/grafana:10.4.3
    container_name: grafana
    depends_on:
      - prometheus                   # Prometheus 컨테이너 먼저 실행
    user: "1000:1000"
    restart: always
    networks:
      - monitoring-net
    ports:
      - "3004:3000"                  # 호스트 3004 → 컨테이너 3000 포트 매핑
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: <your-password>
      GF_USERS_ALLOW_SIGN_UP: "false"          # 외부 회원가입 차단
      GF_SERVER_DOMAIN: "grafana.prod.local"   # 리버스 프록시 도메인
    volumes:
      - ./grafana/data:/var/lib/grafana                    # 대시보드, 설정 영구 저장
      - ./grafana/provisioning:/etc/grafana/provisioning   # 자동 프로비저닝

networks:
  monitoring-net:
    driver: bridge                   # 컨테이너 간 내부 통신용 브릿지 네트워크
```


### Step 5: Prometheus 수집 대상 설정

**prometheus.yml**:

```yaml
global:
  scrape_interval: 15s      # 메트릭 수집 주기
  scrape_timeout: 10s       # 수집 타임아웃
  evaluation_interval: 15s  # 알림 규칙 평가 주기

scrape_configs:
  - job_name: 'spring-actuator'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['10.0.1.10:8080']  # Server1 Private IP
        labels:
          instance: 'server1'

      - targets: ['10.0.1.11:8080']  # Server2 Private IP
        labels:
          instance: 'server2'
```

`targets`에는 Spring Boot 애플리케이션의 **Private IP**를 지정합니다. Prometheus가 내부 네트워크를 통해 직접 `/actuator/prometheus`를 호출합니다.

### Step 6: Docker Compose 실행

```bash
cd /opt/monitoring
docker-compose up -d
docker ps  # 컨테이너 상태 확인
```

정상 실행 후 접속 확인:
- Prometheus: `http://<monitoring-server>:9094`

![Prometheus 접속 화면](https://github.com/user-attachments/assets/8a86edad-01f8-4a2d-af01-2fea8e77b8df)

- Grafana: `http://<monitoring-server>:3004`

![Grafana 접속 화면](https://github.com/user-attachments/assets/5d02b979-c771-4b64-af6e-d41c3397a665)

### Step 7: Nginx 리버스 프록시 설정

외부에서 Prometheus와 Grafana에 접근할 때는 Nginx 리버스 프록시를 통해 IP 제한을 적용합니다.

**grafana.conf**:

```nginx
server {
    listen       3104;
    server_name  monitoring.example.com;

    access_log  /var/log/nginx/grafana/access.log  main;
    error_log   /var/log/nginx/grafana/error.log;

    # IP 제한 - 허용된 IP만 접근 가능
    allow 10.0.0.0/8;      # 사내 네트워크
    allow 203.0.113.50;     # VPN IP
    deny all;

    location / {
        proxy_pass http://localhost:3004;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**prometheus.conf**:

```nginx
server {
    listen 9194;
    server_name monitoring.example.com;

    access_log  /var/log/nginx/prometheus/access.log  main;
    error_log   /var/log/nginx/prometheus/error.log;

    # IP 제한
    allow 10.0.0.0/8;
    allow 203.0.113.50;
    deny all;

    location / {
        proxy_pass http://localhost:9094;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

### Step 8: Grafana 대시보드 설정

#### 8-1. Data Source 연결

1. Grafana 로그인 후 **Configuration → Data Sources → Add data source** 클릭
2. **Prometheus** 선택
3. URL에 `http://prometheus:9094` 입력 (Docker 네트워크 내부 통신)
4. **Save & Test** 클릭하여 연결 확인

![Grafana Data Source 설정](https://github.com/user-attachments/assets/bb801070-4493-4b05-a783-ce86507e6a0d)

#### 8-2. Dashboard Import

Grafana는 커뮤니티에서 만든 대시보드를 Import하여 바로 사용할 수 있습니다.

1. **Dashboards → Import** 클릭
2. Dashboard ID 입력 후 **Load** 클릭
3. Data Source에서 앞서 추가한 Prometheus 선택
4. **Import** 클릭  

![Grafana Dashboard Import](https://github.com/user-attachments/assets/72b45287-1477-4e90-87b6-45769b1e8479)  

**추천 대시보드**:

| Dashboard | ID | 특징 |
|-----------|-----|------|
| [JVM Micrometer](https://grafana.com/grafana/dashboards/4701-jvm-micrometer/) | 4701 | JVM 메모리, GC, 스레드 모니터링 |
| [JustAI System Monitor](https://grafana.com/grafana/dashboards/11378-justai-system-monitor/) | 11378 | 시스템 리소스 종합 모니터링 |
| [Spring Boot Statistics](https://grafana.com/grafana/dashboards/12464-spring-boot-statistics/) | 12464 | HTTP 요청 통계, 응답 시간 분석 |

### Step 9: 대시보드 모니터링

대시보드 Import가 완료되면 Prometheus에서 수집한 메트릭이 실시간으로 시각화됩니다. CPU 사용률, JVM 힙 메모리, GC 빈도, 스레드 수 등 주요 시스템 지표를 한 화면에서 확인할 수 있습니다.

![Grafana 대시보드 모니터링 화면](https://github.com/user-attachments/assets/4259c69f-66a3-4b37-9560-c5519835a4c2)

또한 API 엔드포인트별 응답 속도(Latency) 리더보드를 통해, 어떤 API가 느린지 한눈에 파악할 수 있습니다. 응답 시간이 긴 API를 기준으로 정렬되므로 성능 병목 구간을 빠르게 식별하고 우선순위를 정하여 대응할 수 있습니다.

![API 응답 속도 리더보드](https://github.com/user-attachments/assets/3a0ae0aa-1505-4023-b642-7db6c18a74a2)

---

## 운영 팁

### 수집 메트릭 용량 관리

Prometheus는 시계열 데이터를 로컬 디스크에 저장하므로, 주기적으로 용량을 확인해야 합니다.

```bash
# 모니터링 디렉토리 용량 확인
sudo du -h --max-depth=1 ~/monitoring/
```

docker-compose.yml에서 설정한 `retention.time=15d`와 `retention.size=4GB`에 의해 오래된 데이터는 자동으로 정리됩니다.

### 컨테이너 관리

```bash
# 컨테이너 중지 및 제거
docker-compose down

# Grafana 데이터 초기화 (대시보드, 설정 포함)
rm -rf ./grafana/data/*

# 재시작
docker-compose up -d
```

---

## 정리

Prometheus + Grafana + Spring Actuator 조합은 Spring Boot 기반 서비스에서 가장 널리 사용되는 모니터링 구성입니다.

이 구성을 도입한 후 다음과 같은 효과를 얻을 수 있었습니다:

- **실시간 상태 파악**: CPU, 메모리, JVM 힙, GC, HTTP 응답 시간을 대시보드에서 즉시 확인
- **다중 서버 비교**: 여러 서버 인스턴스의 메트릭을 한 화면에서 비교하여 부하 불균형 감지
- **장애 선제 대응**: Grafana 알림 기능으로 임계값 초과 시 사전 알림 가능
- **낮은 도입 비용**: Actuator와 Micrometer는 별도 코드 추가 없이 주요 메트릭을 제공하며, Prometheus + Grafana는 오픈소스로 무료

---

## 참고 자료

- [Spring Boot Actuator + Prometheus 설정](https://sjh9708.tistory.com/275)
- [Prometheus + Grafana 모니터링](https://devlemon.tistory.com/30)
- [Spring Boot Monitoring with Grafana](https://woojjam.tistory.com/6)
- [우아한형제들 기술블로그 - 모니터링](https://techblog.woowahan.com/9232/)
- [Micrometer 메트릭 설정](https://notavoid.tistory.com/70)
- [Grafana Dashboard 구성](https://nyyang.tistory.com/175)
