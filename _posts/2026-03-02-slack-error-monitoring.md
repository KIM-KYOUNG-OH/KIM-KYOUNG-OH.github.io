---
layout: post
published: true
title: Slack Webhook 기반 에러 모니터링 시스템 구축하기
categories: Infra
date: 2026-03-02
banner:
  image: https://github.com/user-attachments/assets/c4db0448-34f2-4a85-89dc-140d6204fea0
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-weight: bold;"
  subheading_style: "color: gold"
tags: Spring-Boot Logback Slack Webhook Monitoring MDC
sidebar: []
comments: true
---

서비스를 운영하다 보면 "장애가 발생했는데 아무도 몰랐다"는 상황이 가장 두렵습니다.
로그 파일을 뒤늦게 확인하고 나서야 문제를 인지하는 것은, 사용자가 이미 불편을 겪은 뒤라는 뜻이기 때문입니다.

이번 글에서는 **Logback + Slack Webhook**을 활용하여 운영 환경의 ERROR 로그를 실시간으로 Slack 채널에 전송하는 모니터링 시스템을 구축한 경험을 공유합니다.

## 도입 배경

에러 모니터링 시스템을 도입하기 전에는 다음과 같은 문제가 있었습니다:

- **사후 대응 방식의 한계**: 고객이 결함을 먼저 인지한 뒤 CS를 통해 전달받고 나서야 사후 조치하는 비효율적인 프로세스. 야간·주말에 발생한 에러는 다음 영업일까지 방치되기도 했습니다.
- **로그 추적의 어려움**: 에러 발생 시점 정도의 적은 정보만으로 서버 로그에서 타겟 에러 포인트를 찾아야 했습니다. 대량의 로그 속에서 원인이 되는 정확한 지점을 특정하기까지 시간이 오래 걸렸습니다.

이 문제들을 해결하기 위해 다음 목표를 세웠습니다:

1. ERROR 로그를 **실시간**으로 Slack 채널에 자동 전송하여 **선제적 장애 대응** 체계 구축
2. **MDC 기반 traceId**를 생성하여 요청 단위 로그 추적을 용이하게 개선
3. 비즈니스 예외는 **필터링**하여 실제 장애만 알림. 모든 에러를 무차별적으로 보내면 노이즈에 묻혀 정작 중요한 장애를 놓치게 되므로, 중요한 에러만 전송되도록 조절

---

## 아키텍처

### 전체 흐름

<div style="max-width: 700px; margin: 0 auto;">

```mermaid!
graph LR
    A[HTTP 요청] --> B[LoggingSetupFilter<br/>traceId 생성 & MDC 저장]
    B --> C[Controller / Service]
    C --> D{ERROR 발생?}
    D -->|Yes| E[SlackLogLayout<br/>예외 필터링 & 포맷팅]
    E --> F[Slack 채널 알림]
    D -->|No| G[정상 처리]
```

</div>

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| `LoggingSetupFilter` | HTTP 요청마다 traceId 생성, MDC에 저장 |
| `MdcTaskDecorator` | 비동기 스레드로 MDC 컨텍스트 전파 |
| `AsyncConfig` | 스레드 풀에 MdcTaskDecorator 적용 |
| `SlackLogLayout` | Slack 메시지 포맷팅 (이모지, traceId, 스택 트레이스) |
| `logback-spring.xml` | Slack Appender 설정, 레벨 필터링 |
| `application.yml` | Slack Webhook URL, 채널, 사용자명 설정 |

---

## 구현 가이드

### 전제 조건

- Spring Boot 2.x 이상
- Logback (Spring Boot 기본 로깅)
- Slack Workspace에 Incoming Webhook 설정 완료

### Step 1: 의존성 추가

**build.gradle**:

```gradle
dependencies {
    // Slack Appender
    implementation 'com.github.maricn:logback-slack-appender:1.6.1'
}
```

### Step 2: traceId 생성 필터

모든 HTTP 요청에 고유한 `traceId`를 부여하여 로그 추적을 가능하게 합니다.

**LoggingSetupFilter.java**:

```java
import com.github.f4b6a3.uuid.UuidCreator;
import org.slf4j.MDC;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;
import org.springframework.web.context.support.SpringBeanAutowiringSupport;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

/**
 * logging 공통 filter - MDC를 이용해 프로세스 이력 추적
 */
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class LoggingSetupFilter implements Filter {

    public static final String KEY_TRACE_ID = "traceId";

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        SpringBeanAutowiringSupport.processInjectionBasedOnServletContext(
            this, filterConfig.getServletContext()
        );
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain filterChain) throws IOException, ServletException {
        try {
            if (request instanceof HttpServletRequest) {
                String traceId = UuidCreator.getTimeOrderedWithHash().toString();
                MDC.put(KEY_TRACE_ID, traceId);
                request.setAttribute(KEY_TRACE_ID, MDC.get(KEY_TRACE_ID));
            }

            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(KEY_TRACE_ID);
            request.removeAttribute(KEY_TRACE_ID);
        }
    }

    @Override
    public void destroy() {}
}
```

핵심 포인트:
- `@Order(HIGHEST_PRECEDENCE)`: 모든 필터보다 먼저 실행되어 traceId가 전체 요청 흐름에서 사용 가능
- `traceId`: 요청 단위 고유 식별자 (UUID 기반)
- `finally` 블록: 요청 완료 후 MDC 정리 (스레드 풀 재사용 시 메모리 누수 방지)

### Step 3: MDC 비동기 컨텍스트 전파

`@Async` 메서드 실행 시 새로운 스레드가 할당되면 기존 MDC 컨텍스트가 유실됩니다.
`MdcTaskDecorator`를 사용하여 비동기 스레드에도 traceId를 전파합니다.

**MdcTaskDecorator.java**:

```java
import org.slf4j.MDC;
import org.springframework.core.task.TaskDecorator;

import java.util.Map;

public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        // 현재 스레드의 MDC 컨텍스트를 캡처
        Map<String, String> contextMap = MDC.getCopyOfContextMap();

        return () -> {
            // 비동기 스레드에 MDC 컨텍스트 설정
            if (contextMap != null) {
                MDC.setContextMap(contextMap);
            }
            try {
                runnable.run();
            } finally {
                // 스레드 풀 반납 전 MDC 정리
                MDC.clear();
            }
        };
    }
}
```

**동작 원리**:

```mermaid!
sequenceDiagram
    participant Main as 메인 스레드
    participant Dec as MdcTaskDecorator
    participant Async as 비동기 스레드

    Main->>Main: MDC에 traceId 저장
    Main->>Dec: @Async 메서드 호출
    Dec->>Dec: MDC.getCopyOfContextMap()
    Dec->>Async: 새 스레드에 컨텍스트 복사
    Async->>Async: MDC.setContextMap(contextMap)
    Async->>Async: 비즈니스 로직 실행 (traceId 유지)
    Async->>Async: MDC.clear() (finally)
```

### Step 4: 비동기 설정에 MdcTaskDecorator 적용

**AsyncConfig.java**:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "threadPoolTaskExecutor")
    public Executor threadPoolTaskExecutor() {
        ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
        taskExecutor.setCorePoolSize(3);
        taskExecutor.setMaxPoolSize(30);
        taskExecutor.setQueueCapacity(100);
        taskExecutor.setThreadNamePrefix("AsyncExecutor-");
        taskExecutor.setTaskDecorator(new MdcTaskDecorator());  // MDC 전파 핵심
        taskExecutor.initialize();
        return taskExecutor;
    }
}
```

> **주의**: `setTaskDecorator(new MdcTaskDecorator())`를 빠뜨리면 비동기 스레드에서 traceId가 `N/A`로 표시됩니다.

### Step 5: Slack 로그 레이아웃 (커스텀 포맷)

Slack 메시지를 사람이 읽기 좋은 형태로 포맷팅합니다.

**SlackLogLayout.java**:

```java
import ch.qos.logback.classic.Level;
import ch.qos.logback.classic.spi.ILoggingEvent;
import ch.qos.logback.classic.spi.IThrowableProxy;
import ch.qos.logback.classic.spi.StackTraceElementProxy;
import ch.qos.logback.core.LayoutBase;

import java.time.Instant;
import java.time.ZoneId;
import java.time.format.DateTimeFormatter;
import java.util.HashSet;
import java.util.Set;

/**
 * Slack 로그 전송을 위한 커스텀 레이아웃
 * - 로그 레벨별 이모지
 * - traceId 포함
 * - ERROR 시 전체 스택 트레이스
 * - 비즈니스 예외 필터링
 */
public class SlackLogLayout extends LayoutBase<ILoggingEvent> {

    private static final DateTimeFormatter DATE_FORMATTER =
        DateTimeFormatter.ofPattern("yy-MM-dd HH:mm:ss")
            .withZone(ZoneId.systemDefault());

    private String logPrefix = "unknown";
    private Set<String> excludedExceptions = new HashSet<>();

    @Override
    public String doLayout(ILoggingEvent event) {
        // 제외 대상 예외는 빈 문자열 반환 (Slack 전송 안 함)
        if (isExcludedException(event)) {
            return "";
        }

        StringBuilder sb = new StringBuilder();

        // 1. 레벨별 이모지
        String emoji = getEmojiForLevel(event.getLevel());
        sb.append(emoji).append(" *[").append(event.getLevel()).append("]*\n");

        // 2. 기본 정보
        String moduleName = getLogPrefix(event);
        String timestamp = DATE_FORMATTER.format(
            Instant.ofEpochMilli(event.getTimeStamp())
        );
        String traceId = event.getMDCPropertyMap().getOrDefault("traceId", "N/A");
        String loggerName = getShortLoggerName(event.getLoggerName());

        sb.append("[").append(moduleName)
          .append(" ").append(timestamp)
          .append(" TRACE_ID=").append(traceId)
          .append("] ").append(loggerName).append("\n");

        // 3. ERROR + 예외 → 스택 트레이스 포함
        if (event.getLevel().isGreaterOrEqual(Level.ERROR)
            && event.getThrowableProxy() != null) {
            appendErrorDetails(sb, event);
        } else {
            sb.append("> ").append(event.getFormattedMessage()).append("\n");
        }

        return sb.toString();
    }

    private void appendErrorDetails(StringBuilder sb, ILoggingEvent event) {
        IThrowableProxy throwableProxy = event.getThrowableProxy();

        String formattedMessage = event.getFormattedMessage();
        if (formattedMessage != null && !formattedMessage.isEmpty()) {
            sb.append(formattedMessage).append("\n");
        }

        sb.append("```\n");

        // 예외 클래스명 + 메시지
        sb.append(throwableProxy.getClassName());
        String message = throwableProxy.getMessage();
        if (message != null && !message.isEmpty()) {
            sb.append(": ").append(message);
        }
        sb.append("\n");

        // 스택 트레이스 전체 출력
        StackTraceElementProxy[] stackTrace =
            throwableProxy.getStackTraceElementProxyArray();
        if (stackTrace != null && stackTrace.length > 0) {
            for (StackTraceElementProxy element : stackTrace) {
                sb.append("  at ").append(element.toString()).append("\n");
            }
        }

        // Caused by 체인 출력
        IThrowableProxy cause = throwableProxy.getCause();
        while (cause != null) {
            sb.append("Caused by: ").append(cause.getClassName());
            if (cause.getMessage() != null) {
                sb.append(": ").append(cause.getMessage());
            }
            sb.append("\n");

            StackTraceElementProxy[] causeStackTrace =
                cause.getStackTraceElementProxyArray();
            if (causeStackTrace != null) {
                for (StackTraceElementProxy element : causeStackTrace) {
                    sb.append("  at ").append(element.toString()).append("\n");
                }
            }
            cause = cause.getCause();
        }

        sb.append("```\n");
    }

    private String getEmojiForLevel(Level level) {
        if (level.isGreaterOrEqual(Level.ERROR)) return ":rotating_light:";
        if (level.isGreaterOrEqual(Level.WARN))  return ":warning:";
        if (level.isGreaterOrEqual(Level.INFO))  return ":information_source:";
        return ":bug:";
    }

    private String getLogPrefix(ILoggingEvent event) {
        if (logPrefix != null && !logPrefix.equals("unknown")) {
            return logPrefix;
        }
        try {
            String contextLogPrefix =
                event.getLoggerContextVO().getPropertyMap().get("LOG_PREFIX");
            if (contextLogPrefix != null && !contextLogPrefix.isEmpty()) {
                this.logPrefix = contextLogPrefix;
                return contextLogPrefix;
            }
            return "unknown";
        } catch (Exception e) {
            return "unknown";
        }
    }

    public void setLogPrefix(String logPrefix) {
        this.logPrefix = logPrefix;
    }

    public void setExcludedExceptions(String excludedExceptions) {
        if (excludedExceptions != null && !excludedExceptions.trim().isEmpty()) {
            String[] exceptions = excludedExceptions.split(",");
            this.excludedExceptions = new HashSet<>();
            for (String exception : exceptions) {
                String trimmed = exception.trim();
                if (!trimmed.isEmpty()) {
                    this.excludedExceptions.add(trimmed);
                }
            }
        }
    }

    private boolean isExcludedException(ILoggingEvent event) {
        if (excludedExceptions.isEmpty() || event.getThrowableProxy() == null) {
            return false;
        }

        String exceptionClassName = event.getThrowableProxy().getClassName();

        for (String excluded : excludedExceptions) {
            if (exceptionClassName.equals(excluded)) return true;
            if (exceptionClassName.endsWith("." + excluded)) return true;
        }

        return false;
    }

    private String getShortLoggerName(String fullLoggerName) {
        if (fullLoggerName == null || fullLoggerName.isEmpty()) {
            return "Unknown";
        }
        int lastDot = fullLoggerName.lastIndexOf(".");
        if (lastDot >= 0 && lastDot < fullLoggerName.length() - 1) {
            return fullLoggerName.substring(lastDot + 1);
        }
        return fullLoggerName;
    }
}
```

`SlackLogLayout`의 주요 기능을 정리하면:

| 기능 | 설명 |
|------|------|
| 이모지 매핑 | ERROR → 🚨, WARN → ⚠️, INFO → ℹ️ |
| traceId 포함 | MDC에서 traceId를 추출하여 메시지에 포함 |
| 스택 트레이스 | ERROR 레벨 + 예외 발생 시 전체 스택 트레이스 출력 |
| Caused by 체인 | 예외 원인 체인을 순회하며 모든 스택 트레이스 포함 |
| 예외 필터링 | `excludedExceptions`에 등록된 비즈니스 예외는 빈 문자열 반환 |

### Step 6: Slack 모니터링 채널 및 Incoming Webhook 생성

Slack에서 에러 알림을 수신할 채널을 만들고 Webhook URL을 발급합니다.

#### 6-1. 모니터링 채널 생성

1. Slack 워크스페이스에서 **채널 추가** 클릭
2. 채널 정보 입력:
   - **이름**: `error-monitoring` (환경별로 `error-monitoring-dev`, `error-monitoring-prod` 등 분리 권장)
   - **가시성**: 비공개 채널 권장 (운영 노이즈 차단)
   - **설명**: `서버 ERROR 로그 자동 알림 채널`
3. 알림을 받아야 하는 팀원 초대

#### 6-2. Incoming Webhook URL 생성

1. 대상 모니터링 채널 설정 편집 → **통합** 탭 → **앱 추가** 클릭

<img src="https://github.com/user-attachments/assets/49110d57-e65c-4b33-94b0-bb96f94999f8" alt="Slack 채널 통합 탭 앱 추가" style="max-width: 600px;">

{:start="2"}
2. **'Incoming Webhooks'** 검색 후 설치

<img src="https://github.com/user-attachments/assets/433743dc-efea-417d-aeb6-883a2dd56516" alt="Incoming Webhooks 검색 및 구성" style="max-width: 600px;">

{:start="3"}
3. **'Slack에 추가'** 클릭 → 대상 모니터링 채널 선택 후 **'수신 웹후크 통합 앱 추가'** 클릭

<img src="https://github.com/user-attachments/assets/67d6e233-f192-481e-86e6-fe9280afca1a" alt="수신 웹후크 통합 앱 추가" style="max-width: 600px;">

{:start="4"}
4. 생성된 Webhook URL 복사

```
https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
```

> **주의**: Webhook URL은 외부에 노출되면 안 됩니다. 환경 변수 또는 시크릿 매니저를 통해 관리하세요.

### Step 7: application.yml 설정

```yaml
logging:
  config: classpath:logback-spring.xml
  slack:
    webhook-uri: https://hooks.slack.com/services/XXXXX/YYYYY/ZZZZZ
    channel: error-monitoring
    username: Server-Alert
```

| 속성 | 설명 | 예시 |
|------|------|------|
| `config` | Logback 설정 파일 경로 (**반드시 `-spring` 접미사**) | `classpath:logback-spring.xml` |
| `webhook-uri` | Slack Incoming Webhook URL | `https://hooks.slack.com/services/...` |
| `channel` | 알림 수신 채널 (# 제외) | `error-monitoring` |
| `username` | 메시지 발신자명 | `API-Server-Alert` |

#### 왜 logback-spring.xml 이어야 하는가?

`logback.xml`과 `logback-spring.xml`은 **로딩 시점**이 다릅니다.

| 파일명 | 로딩 주체 | 시점 | Spring 기능 |
|--------|-----------|------|-------------|
| `logback.xml` | Logback 프레임워크 자체 | Spring 초기화 **이전** | 사용 불가 |
| `logback-spring.xml` | Spring Boot LoggingSystem | Spring 초기화 **이후** | 사용 가능 |

`logback.xml`은 Logback이 클래스패스에서 자동 감지하여 **Spring Context가 생성되기 전**에 로딩합니다. 이 시점에는 `application.yml`이 아직 파싱되지 않았기 때문에 Spring 확장 태그가 동작하지 않습니다.

이 가이드의 Slack Appender 설정에서 사용하는 `<springProperty>`가 대표적인 Spring 확장 태그입니다:

```xml
<!-- application.yml의 값을 logback 변수로 주입 -->
<springProperty name="SLACK_WEBHOOK_URI" source="logging.slack.webhook-uri"/>
<springProperty name="SLACK_CHANNEL"     source="logging.slack.channel"/>
<springProperty name="SLACK_USERNAME"    source="logging.slack.username"/>
```

`logback.xml`에서 위 태그를 사용하면 Spring Environment가 없으므로 **모든 값이 빈 문자열로 바인딩**되어 Slack 전송이 실패합니다.

> **정리**: `<springProperty>`, `<springProfile>` 등 Spring 확장 태그를 사용하려면 반드시 파일명이 `logback-spring.xml`이어야 합니다.

### Step 8: logback-spring.xml 설정

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false" scan="true" scanPeriod="300 seconds">

    <include resource="org/springframework/boot/logging/logback/defaults.xml" />

    <!-- 1. 공통 프로퍼티 -->
    <property name="LOG_LEVEL"       value="DEBUG"/>
    <property name="LOG_PREFIX"      value="my-api"/>
    <property name="LOG_FILE_PREFIX" value="my-api"/>
    <property name="LOG_PATH"        value="/home/ec2-user/logs/my-api"/>
    <property name="TOTAL_SIZE_CAP_SERVICE" value="50GB"/>

    <!-- 2. Slack 설정 (application.yml에서 주입) -->
    <springProperty name="SLACK_WEBHOOK_URI" source="logging.slack.webhook-uri"/>
    <springProperty name="SLACK_CHANNEL"     source="logging.slack.channel"/>
    <springProperty name="SLACK_USERNAME"    source="logging.slack.username"/>

    <!-- 3. 공통 Appender -->
    <include resource="logback-default.xml" />

    <!-- ============================================ -->
    <!-- 4. Slack Appender 설정                       -->
    <!-- ============================================ -->

    <!-- 4-1. Slack Appender (모든 레벨) - 명시적 호출용 -->
    <appender name="slack" class="com.github.maricn.logback.SlackAppender">
        <webhookUri>${SLACK_WEBHOOK_URI}</webhookUri>
        <channel>#${SLACK_CHANNEL}</channel>
        <username>${SLACK_USERNAME}</username>
        <colorCoding>true</colorCoding>
        <layout class="com.example.common.logging.SlackLogLayout">
            <logPrefix>${LOG_PREFIX}</logPrefix>
        </layout>
    </appender>

    <!-- 4-2. Slack Appender (ERROR 전용) - root logger용 -->
    <appender name="slackError" class="com.github.maricn.logback.SlackAppender">
        <webhookUri>${SLACK_WEBHOOK_URI}</webhookUri>
        <channel>#${SLACK_CHANNEL}</channel>
        <username>${SLACK_USERNAME}</username>
        <colorCoding>true</colorCoding>
        <layout class="com.example.common.logging.SlackLogLayout">
            <logPrefix>${LOG_PREFIX}</logPrefix>
            <!-- 비즈니스 예외는 Slack 전송 제외 (쉼표 구분자로 여러건 추가) -->
            <excludedExceptions>
                BizException,BizNotFoundException,ForbiddenException,
                BizAuthorizationException,BindException,ConflictException,
                ApiException
            </excludedExceptions>
        </layout>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>ERROR</level>
        </filter>
    </appender>

    <!-- ============================================ -->
    <!-- 5. 로거 설정                                 -->
    <!-- ============================================ -->

    <!-- Slack 전용 로거 (명시적 호출용 - 모든 레벨 허용) -->
    <logger name="slackLogger" level="DEBUG" additivity="false">
        <appender-ref ref="slack"/>
    </logger>

    <!-- Root Logger -->
    <root level="${LOG_LEVEL}">
        <appender-ref ref="asyncFile" />
        <!-- ERROR 이상 로그만 자동으로 Slack 전송 -->
        <appender-ref ref="slackError"/>
    </root>

</configuration>
```

#### Slack Appender를 왜 2개로 분리했는가?

| Appender | 용도 | 레벨 | 예외 필터링 | 연결 |
|----------|------|------|-------------|------|
| `slack` | 명시적 호출 | ALL | 없음 | `slackLogger` 전용 |
| `slackError` | 자동 ERROR 알림 | ERROR만 | 비즈니스 예외 제외 | root logger |

- **`slackError`**: 모든 ERROR 로그가 자동 전송됩니다. 단, `BizException` 같은 비즈니스 예외는 `excludedExceptions` 설정으로 필터링되어 실제 시스템 장애만 알림을 받습니다.
- **`slack`**: 개발자가 특정 시점에 명시적으로 Slack에 전송할 때 사용합니다. DEBUG/INFO 레벨도 가능하므로 배포 알림, 헬스체크 결과 등 다양한 용도로 활용할 수 있습니다.

---

## 사용법

### 1. 자동 ERROR 알림 (기본 동작)

별도 코드 변경 없이, 애플리케이션에서 발생하는 모든 ERROR 로그가 자동으로 Slack에 전송됩니다.

```java
@Slf4j
@Service
public class OrderService {

    public void processOrder(Long orderId) {
        try {
            // 비즈니스 로직
        } catch (Exception e) {
            // 이 ERROR 로그가 자동으로 Slack에 전송됨
            log.error("주문 처리 실패. orderId={}", orderId, e);
            throw e;
        }
    }
}
```

### 2. 명시적 Slack 전송 (선택적)

특정 이벤트를 DEBUG/INFO 레벨로 Slack에 전송하고 싶을 때 사용합니다.

```java
@RestController
public class MonitorController {

    // slackLogger 이름의 로거 사용
    private static final Logger slackLog = LoggerFactory.getLogger("slackLogger");

    @PostMapping("/api/deploy/notify")
    public String notifyDeploy() {
        slackLog.info("v2.5.0 배포 완료 - 정상 구동 확인");
        return "OK";
    }
}
```

### 3. 비동기 메서드에서의 로깅

`MdcTaskDecorator` 적용으로 `@Async` 메서드에서도 traceId가 유지됩니다.

```java
@Slf4j
@Service
public class NotificationService {

    @Async("threadPoolTaskExecutor")
    public void sendPushAsync(Long memberId, String message) {
        // traceId가 비동기 스레드에서도 유지됨
        log.info("푸시 전송 시작. memberId={}", memberId);

        try {
            // 푸시 전송 로직
        } catch (Exception e) {
            // ERROR → Slack 자동 전송 (traceId 포함)
            log.error("푸시 전송 실패. memberId={}", memberId, e);
        }
    }
}
```

---

## Slack 메시지 출력 예시

실제 Slack 채널에 전송되는 메시지는 다음과 같은 형태입니다:

<img src="https://github.com/user-attachments/assets/c4bf7751-04e9-436a-b88b-44842e717aa8" alt="Slack 에러 알림 메시지 예시" style="max-width: 600px;">

traceId가 포함되어 있으므로, 이 값으로 로그 파일을 검색하면 해당 요청의 전체 흐름을 추적할 수 있습니다.

---

## 결론

이 시스템을 도입한 후 다음과 같은 효과를 얻을 수 있었습니다:

- **장애 인지 시간 단축**: 에러 발생 즉시 Slack 알림으로 팀 전체가 인지
- **노이즈 제거**: 비즈니스 예외 필터링으로 실제 장애 알림만 수신
- **빠른 원인 파악**: traceId 기반으로 요청 단위 로그 추적 가능
- **비동기 추적 연속성**: MDC 전파로 `@Async` 메서드에서도 traceId 유지

별도의 모니터링 인프라 없이도, Logback + Slack Webhook 조합만으로 실시간 에러 모니터링 체계를 구축할 수 있습니다. 비용 부담 없이 빠르게 적용할 수 있으므로, 아직 에러 알림 시스템이 없는 프로젝트라면 도입을 검토해 보시기 바랍니다.

---

## 참고 자료

- [logback-slack-appender GitHub](https://github.com/maricn/logback-slack-appender)
- [SLF4J MDC 공식 문서](https://www.slf4j.org/manual.html#mdc)
- [Spring Boot Logging 공식 문서](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)
