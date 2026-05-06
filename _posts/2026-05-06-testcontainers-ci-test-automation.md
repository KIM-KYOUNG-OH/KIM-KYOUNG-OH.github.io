---
layout: post
published: true
title: "CI/CD 테스트 자동화 with Testcontainers"
categories: Test
date: 2026-05-06
banner:
  image: https://github.com/user-attachments/assets/3d904847-3436-4042-ae28-c0010dee881b
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-weight: bold;"
  subheading_style: "color: gold"
tags: Test Testcontainers Spring-Boot CI-CD Integration-Test Docker Gradle
sidebar: []
comments: true
---

프로젝트 초기에는 로컬 DB에 직접 연결하여 테스트를 수행해도 크게 문제가 없습니다.  
하지만 팀원이 늘어나고 CI 파이프라인이 구축되면, **"내 PC에서는 되는데 다른 PC에서는 안되는데요"** 라는 말이 등장하기 시작합니다.

이번 글에서는 Unit/Integration/E2E 역할별로 테스트 레이어를 효율적으로 분리하고, **Testcontainers**를 활용해 Docker만 있으면 누구나 동일한 DB 환경에서 테스트를 실행할 수 있도록 환경을 통일한 과정을 정리합니다.

## Testcontainers 란?

Testcontainers는 테스트 시 Docker 컨테이너 기반으로 테스트 환경 의존성을 제어할 수 있는 오픈소스 라이브러리입니다.  
서비스 의존성(DB, 메시지 큐, Redis 등)을 프로덕션과 동일한 실제 서비스로 Docker 컨테이너에 띄워 사용할 수 있습니다.

## 왜 Testcontainers 인가?

### 1. 작업자들 간 테스트 환경 통일

기존에는 각 개발자가 로컬에 설치한 DB에 의존하여 테스트를 수행했습니다.  
개발자마다 DB 버전, 설정, 초기 데이터가 달라서 한 사람이 작성한 테스트가 다른 사람의 환경에서 실패하는 일이 잦았습니다.  
CI 서버에서도 별도의 테스트용 DB를 수동으로 구축하고 관리해야 했습니다.

Testcontainers를 도입하면 `docker-compose-test.yml` 하나로 DB 버전, 설정, 스키마가 코드에 고정되므로, Docker만 설치되어 있으면 누구나 동일한 환경에서 테스트를 실행할 수 있습니다.

### 2. 테스트 환경의 외부 의존성 격리

테스트가 로컬에 설치된 DB에 직접 의존하면, DB 상태(다른 테스트가 남긴 데이터, 스키마 변경 등)에 따라 결과가 달라집니다.  
Testcontainers는 매 테스트 실행마다 일회용(throwaway) 컨테이너를 새로 생성하므로, 항상 알려진 초기 상태(known DB state)에서 테스트가 시작됩니다.  
외부 환경의 상태에 영향받지 않는 독립적인 테스트 실행을 보장합니다.

## 테스트 레이어(테스트 피라미드) 분리

```
        /  E2E  \        10% — 핵심 시나리오만
       /─────────\
      / Integration\     20% — 연동 지점의 정상/실패
     /───────────────\
    /      Unit       \  70% — 비즈니스 로직 전수 검증
```

|                    | Unit                                       | Integration                | E2E                                    |
| ------------------ | ------------------------------------------ | -------------------------- | -------------------------------------- |
| **목적**             | 단일 클래스, 최소 단위(Unit)의 비즈니스 로직 검증 (외부 의존성 없이) | 컴포넌트 간 연동(Service → DB) 검증 | 사용자 시나리오(HTTP → DB) 전체 흐름 검증 |
| **범위**             | 단일 클래스 (의존성은 Mock)                   | Service → Mapper → 실제 DB   | HTTP 요청 → Controller → Service → DB  |
| **외부 의존성**       | 전부 Mock                                   | DB만 실제, 나머지 `@MockBean`  | DB 실제, 외부 API는 Mock/Stub          |
| **Spring Context** | 없음 (`@ExtendWith(MockitoExtension.class)`) | 전체 로드 (`@SpringBootTest`)  | 서버 기동 (`@SpringBootTest(RANDOM_PORT)`) |

통합 테스트에서 모든 시나리오를 커버하려 하지 말고, **연동 지점의 정상/실패 케이스**에 집중하는 것이 중요합니다.  
모든 분기 검증은 Unit 테스트의 역할입니다.


### 상태 관리와 격리성

테스트의 **핵심 원칙은 멱등성**(어떤 순서로, 몇 번 실행해도 동일한 결과)입니다.  
각 레이어는 이를 다른 방식으로 보장합니다.

|                      | Unit         | Integration                 | E2E                  |
| -------------------- | ------------ | --------------------------- | -------------------- |
| **상태 발생**           | 없음 (순수 JVM)  | 있음 (DB INSERT/UPDATE)      | 있음 (DB COMMIT)      |
| **`@Transactional`** | 불필요          | **사용** (메서드 종료 시 자동 롤백)  | **사용 불가*** |
| **격리 주체**           | Mock (프레임워크) | `@Transactional` 롤백        | 개발자 직접 관리         |

> *E2E 테스트는 `RANDOM_PORT`로 실제 서버를 기동하므로, 테스트 코드와 서버가 서로 다른 스레드(다른 DB 커넥션)에서 실행됩니다. 테스트의 `@Transactional`은 서버가 이미 커밋한 데이터를 롤백할 수 없어, `@BeforeEach`에서 직접 정리해야 합니다.

## 구현 가이드

### Step 1: 디렉토리 구조

```
src/test/
├── java/com/example/
│   ├── unit/                          # Docker 불필요
│   │   └── {domain}/FooServiceTest.java
│   ├── integration/                   # Docker 필요
│   │   ├── support/IntegrationTestSupport.java
│   │   └── {domain}/FooServiceIntegrationTest.java
│   ├── e2e/                           # Docker 필요
│   │   ├── support/E2ETestSupport.java
│   │   └── {domain}/FooControllerE2ETest.java
│   └── support/
│       └── SharedTestContainers.java  # 싱글톤 컨테이너
└── resources/
    ├── application-test.yml
    ├── docker-compose-test.yml
    └── db/schema.sql
```

- `FooServiceTest.java` — Unit 테스트 코드
- `IntegrationTestSupport.java` — Integration 테스트에서 공통으로 사용하는 추상 클래스
- `FooServiceIntegrationTest.java` — Integration 테스트 코드
- `E2ETestSupport.java` — E2E 테스트에서 공통으로 사용하는 추상 클래스
- `FooControllerE2ETest.java` — E2E 테스트 코드
- `SharedTestContainers.java` — Testcontainers 싱글톤, 모든 DB 테스트가 공유
- `application-test.yml` — 테스트 프로필 설정
- `docker-compose-test.yml` — Testcontainers가 사용하는 컨테이너 정의
- `db/schema.sql` — 컨테이너 최초 기동 시 자동 실행되는 DDL

### Step 2: Gradle 테스트 태스크 분리

디렉토리 패턴으로 테스트를 분리합니다. 별도 sourceSet 없이 `include` 필터만으로 구현합니다.

```
./gradlew allTest 실행 시:

┌──────────┐     ┌────────────────┐     ┌──────────┐
│   test   │────►│ integrationTest│────►│  e2eTest │
│ (unit/)  │     │ (integration/) │     │  (e2e/)  │
│          │     │                │     │          │
│ Docker ✗ │     │   Docker ✓     │     │ Docker ✓ │
└──────────┘     └────────────────┘     └──────────┘
  빠름                보통                  느림
```

```groovy
// 공통 설정
tasks.withType(Test).configureEach {
    useJUnitPlatform()
    systemProperty 'spring.profiles.active', 'test'
    testLogging {
        events "failed"
        exceptionFormat "full"
    }
}

// Unit Test: unit/ 디렉토리
test {
    useJUnitPlatform()
    include '**/unit/**'
}

// Integration Test: integration/ 디렉토리
tasks.register('integrationTest', Test) {
    useJUnitPlatform()
    include '**/integration/**'
    shouldRunAfter test  // 순서 힌트 (단독 실행 가능)
}

// E2E Test: e2e/ 디렉토리
tasks.register('e2eTest', Test) {
    useJUnitPlatform()
    include '**/e2e/**'
    mustRunAfter tasks.named('integrationTest')  // 강제 순서
}

// 전체 순차 실행
tasks.register('allTest') {
    dependsOn test, tasks.named('integrationTest'), tasks.named('e2eTest')
    group = 'verification'
    description = 'unit → integration → e2e 순차 실행'
}

// build 시 allTest 자동 실행
build.dependsOn tasks.named('allTest')
```

### Step 3: Testcontainers 싱글톤 패턴

#### 의존성 추가

```groovy
testImplementation 'org.testcontainers:testcontainers:2.0.3'
testImplementation 'org.testcontainers:testcontainers-mariadb:2.0.3'
testImplementation 'org.testcontainers:testcontainers-junit-jupiter:2.0.3'
```

#### docker-compose-test.yml

`src/test/resources/` 하위에 배치하여 classpath에서 접근합니다.

```yaml
services:
  mariadb:
    image: mariadb:10.11
    command: --sql-mode=ORACLE
    environment:
      MYSQL_DATABASE: myapp_test
      MYSQL_ROOT_PASSWORD: test
      MYSQL_USER: test
      MYSQL_PASSWORD: test
    volumes:
      - ./db/schema.sql:/docker-entrypoint-initdb.d/schema.sql
    ports:
      - "3306"
```

> **`docker-entrypoint-initdb.d/`**: MariaDB/MySQL 컨테이너는 최초 기동 시 이 경로의 `.sql` 파일을 자동 실행합니다. `volumes`로 `./db/schema.sql`을 이 경로에 연결하여 스키마를 자동 생성합니다.

> **`ports: - "3306"`**: 호스트 포트를 지정하지 않으면 Docker가 랜덤 포트를 할당합니다. Testcontainers가 `getServicePort()`로 매핑된 포트를 조회하므로 포트 충돌이 없습니다.

#### SharedTestContainers (싱글톤)

Integration & E2E 테스트 간 컨테이너를 새로 띄우지 않고 동일한 컨테이너를 재사용합니다.  
`static` 블록에서 컨테이너를 시작하면, 해당 클래스를 참조하는 모든 테스트가 동일한 컨테이너 인스턴스를 공유하므로 전체 실행 시간이 크게 단축됩니다.

```java
public final class SharedTestContainers {

    private static final String SERVICE_NAME = "mariadb";
    private static final int SERVICE_PORT = 3306;

    public static final ComposeContainer COMPOSE =
            new ComposeContainer(classpathFile("docker-compose-test.yml"))
                    .withExposedService(SERVICE_NAME, SERVICE_PORT,
                            Wait.forListeningPort()
                                .withStartupTimeout(Duration.ofSeconds(60)));

    // static 블록에서 1회만 시작 → JVM 종료 시 자동 정리
    static { COMPOSE.start(); }

    private SharedTestContainers() {}

    public static void registerDatasourceProperties(
            DynamicPropertyRegistry registry) {
        String host = COMPOSE.getServiceHost(SERVICE_NAME, SERVICE_PORT);
        int port = COMPOSE.getServicePort(SERVICE_NAME, SERVICE_PORT);
        registry.add("spring.datasource.url",
                () -> "jdbc:mariadb://" + host + ":" + port + "/myapp_test");
        registry.add("spring.datasource.username", () -> "test");
        registry.add("spring.datasource.password", () -> "test");
        registry.add("spring.datasource.driver-class-name",
                () -> "org.mariadb.jdbc.Driver");
    }

    private static File classpathFile(String name) {
        try {
            return new File(SharedTestContainers.class
                    .getClassLoader().getResource(name).toURI());
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### Step 4: 테스트 추상 클래스

#### Integration Test 추상 클래스

```java
@SpringBootTest
@Transactional
public abstract class IntegrationTestSupport {

    @DynamicPropertySource
    static void setDatasourceProperties(DynamicPropertyRegistry registry) {
        SharedTestContainers.registerDatasourceProperties(registry);
    }
}
```

- `@SpringBootTest`: 전체 ApplicationContext를 로드
- `@Transactional`: 각 테스트 메서드 종료 시 자동 롤백하여 테스트 간 데이터 격리
- `@DynamicPropertySource`: Spring Context 로드 시 호출되어, `SharedTestContainers`가 띄운 컨테이너의 DB 연결 정보(호스트, 랜덤 포트, 계정)를 `spring.datasource.*` 속성에 동적으로 주입합니다. Docker가 할당하는 포트가 매번 달라지므로 `application-test.yml`에 미리 고정할 수 없어 이 방식이 필요합니다.

#### E2E Test 추상 클래스

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public abstract class E2ETestSupport {

    @Autowired
    protected TestRestTemplate restTemplate;

    @DynamicPropertySource
    static void setDatasourceProperties(DynamicPropertyRegistry registry) {
        SharedTestContainers.registerDatasourceProperties(registry);
    }
}
```

- `RANDOM_PORT`: 실제 내장 서버를 랜덤 포트로 기동. CI에서 여러 빌드가 동시에 돌 때 포트 충돌을 방지
- `@Transactional` 없음: 테스트 스레드와 서버 스레드가 분리되어 롤백이 동작하지 않으므로, `@BeforeEach`에서 데이터를 직접 정리

### Step 5: application-test.yml

실제 DB 연결 정보는 `@DynamicPropertySource`가 런타임에 덮어쓰므로, 여기 값은 사용되지 않습니다.  
다만 메인 `application.yml`이 `${system.datasource.url}` 같은 키를 참조하고 있으면, test 프로필에서도 해당 키가 존재해야 파싱 에러가 발생하지 않으므로 임의의 값을 채워둡니다.

```yaml
system:
  datasource:
    driver-class-name: org.mariadb.jdbc.Driver
    url: jdbc:mariadb://localhost:3306/dummy
    username: dummy
    password: dummy
```

### Step 6: 테스트 코드 작성

#### 6-1. Unit Test

Spring Context 없이 Mockito만으로 단일 클래스의 비즈니스 로직을 검증합니다.

```java
@ExtendWith(MockitoExtension.class)
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderServiceTest {

    @InjectMocks
    private OrderServiceImpl orderService;

    @Mock
    private OrderMapper orderMapper;

    @Test
    @Order(1)
    @DisplayName("신규 주문 등록 성공")
    void saveOrder_success() {
        // given
        OrderInfo orderInfo = new OrderInfo();
        orderInfo.setOrderDt("20260305");
        given(orderMapper.getOrderId()).willReturn("ORD0001");

        // when
        orderService.saveOrder(orderInfo);

        // then
        verify(orderMapper, times(1)).insertOrder(any());
    }

    @Test
    @Order(2)
    @DisplayName("중복 주문 시 ConflictException 발생")
    void saveOrder_duplicate_throwsConflict() {
        // given
        OrderInfo orderInfo = new OrderInfo();
        orderInfo.setOrderDt("20260305");
        given(orderMapper.findOrder(any())).willReturn(new OrderInfo());

        // when & then
        assertThatThrownBy(() -> orderService.saveOrder(orderInfo))
                .isInstanceOf(ConflictException.class);
        verify(orderMapper, times(0)).insertOrder(any());
    }
}
```

#### 6-2. Integration Test

Service → Mapper → 실제 DB(Testcontainers) 연동을 검증합니다.  
외부 의존성(파일 업로드 등)은 `@MockBean`으로 차단합니다.

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderServiceIntegrationTest extends IntegrationTestSupport {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderMapper orderMapper;

    @MockBean
    private FileUploadService fileUploadService;

    @Test
    @Order(1)
    @DisplayName("주문 등록 후 DB에서 조회된다")
    void saveOrder_persistsToDb() {
        // given
        OrderInfo orderInfo = new OrderInfo();
        orderInfo.setOrderDt("20260305");

        // when
        orderService.saveOrder(orderInfo);

        // then
        OrderInfo saved = orderMapper.findByDate("20260305");
        assertThat(saved).isNotNull();
    }

    @Test
    @Order(2)
    @DisplayName("존재하지 않는 주문 조회 시 NotFoundException 발생")
    void getOrder_notFound_throwsException() {
        // given
        String nonExistentId = "NONEXISTENT_ID";

        // when & then
        assertThatThrownBy(() -> orderService.getOrderDetail(nonExistentId))
                .isInstanceOf(NotFoundException.class);
    }
}
```

#### 6-3. E2E Test

HTTP 요청부터 DB 저장까지 전체 흐름을 검증합니다.  
`TestRestTemplate`으로 실제 서버에 요청을 보냅니다.

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderControllerE2ETest extends E2ETestSupport {

    @Autowired
    private OrderMapper orderMapper;

    @Test
    @Order(1)
    @DisplayName("주문 등록 API - HTTP 200 + DB 저장")
    void saveOrder_httpRoundTrip_success() {
        // given
        HttpHeaders headers = createAuthHeaders();
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("orderDt", "20260305");

        // when
        ResponseEntity<Response> response = restTemplate.postForEntity(
                "/api/order", new HttpEntity<>(params, headers), Response.class);

        // then
        assertThat(response.getStatusCodeValue()).isEqualTo(200);
        assertThat(orderMapper.findByDate("20260305")).isNotNull();
    }

    @Test
    @Order(2)
    @DisplayName("존재하지 않는 주문 상세 조회")
    void getOrderDetail_notFound() {
        // given
        HttpHeaders headers = createAuthHeaders();
        MultiValueMap<String, String> params = new LinkedMultiValueMap<>();
        params.add("orderId", "NONEXISTENT_ID");

        // when
        ResponseEntity<Response> response = restTemplate.postForEntity(
                "/api/order/detail", new HttpEntity<>(params, headers), Response.class);

        // then
        assertThat(response.getStatusCodeValue()).isEqualTo(404);
    }
}
```

---

### 테스트 실행

#### 로컬 환경 (Mac + OrbStack)

Docker를 실행하기 위해서는 Linux 커널이 필요합니다.  
macOS에는 Linux 커널이 없으므로 가상 Linux VM을 띄워서 그 위에서 Docker를 실행해야 합니다.  
[OrbStack](https://orbstack.dev/download)은 경량 Linux VM 기반으로 Docker 런타임 환경을 제공해주는 데스크탑 애플리케이션입니다.

OrbStack 설치 후 Testcontainers가 Docker 소켓 경로를 인식하도록 설정합니다.

```shell
echo "docker.host=unix:///var/run/docker.sock" > ~/.testcontainers.properties
```

```shell
# 개별 실행
./gradlew :module-name:test              # Unit만
./gradlew :module-name:integrationTest   # Integration만
./gradlew :module-name:e2eTest           # E2E만

# 전체 순차 실행
./gradlew :module-name:allTest           # Unit → Integration → E2E
```

`allTest`를 실행하면 `test` → `integrationTest` → `e2eTest` 순서로 각 레이어가 순차 실행되고, 전체 결과가 출력됩니다.

![allTest 콘솔 실행 결과](https://github.com/user-attachments/assets/b530bb7b-04a6-4d79-b0d4-cbf6cf001434)

![로컬 테스트 실행 결과](https://github.com/user-attachments/assets/6aa00b0f-086f-4e1e-99c9-17ccfd6fbae8)

테스트를 실행하면 Testcontainers가 자동으로 3개의 컨테이너를 띄웁니다.

| 컨테이너 | 이미지 | 역할 |
|-----------|--------|------|
| mariadb | `mariadb:10.11` | `docker-compose-test.yml`에 정의한 테스트용 DB |
| testcontainers-ryuk | `testcontainers/ryuk` | 테스트 종료 시 컨테이너·네트워크·볼륨을 자동 정리 |
| testcontainers-socat | `alpine/socat` | 호스트와 Docker Compose 내부 네트워크 간 포트 포워딩 중계 |

mariadb만 직접 정의한 컨테이너이고, ryuk과 socat은 Testcontainers가 내부적으로 관리하는 인프라 컨테이너입니다.

테스트가 종료되면 아래 이미지처럼 ryuk이 모든 컨테이너를 자동으로 삭제합니다. 수동 정리가 필요 없습니다.

![테스트 종료 후 컨테이너 자동 삭제](https://github.com/user-attachments/assets/6a9b3e5c-6b18-4214-9497-7720d5c9901f)

---

#### Jenkins CI/CD 환경

Docker 설치 (CentOS/Rocky Linux):

```shell
# Docker 저장소 추가 + 설치
sudo yum-config-manager --add-repo \
  https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Jenkins 실행 사용자에게 Docker 권한 부여
sudo usermod -aG docker <jenkins-user>

# Docker 서비스 시작
sudo systemctl enable docker && sudo systemctl start docker
```

> `usermod -aG` 후 Jenkins 서비스를 재시작해야 새 그룹 권한이 적용됩니다.

Jenkins Pipeline 실행: 빌드 명령 실행 시 Unit → Integration → E2E 테스트가 자동으로 수행됩니다.

![Jenkins 빌드 결과](https://github.com/user-attachments/assets/c1f8289d-9f3d-4731-bc92-85a5e834579e)


---

## 정리

이번 포스팅에서는 Testcontainers를 활용해 로컬과 CI 환경 모두에서 동일한 DB 환경으로 테스트를 실행하는 방법을 알아봤습니다.
Unit / Integration / E2E를 디렉토리 구조로 분리하여 각 테스트 레이어의 목적과 범위를 명확히 하고, 매 실행마다 일회용 컨테이너가 생성되고 자동 삭제되는 격리된 환경에서 안전하게 테스트할 수 있는 구조를 구축했습니다.

