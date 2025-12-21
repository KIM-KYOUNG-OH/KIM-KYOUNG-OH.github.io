---
layout: post
published: true
title: ModSecurity WAF로 웹 애플리케이션 보안 강화하기
categories: Infra
date: 2025-12-21
banner:
  image: https://github.com/user-attachments/assets/7b3bc968-fb7f-4f46-a178-c4abe1d97a42
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-weight: bold;"
  subheading_style: "color: gold"
tags: ModSecurity WAF OWASP Nginx Security
sidebar: []
comments: true
---

웹 애플리케이션이 복잡해질수록 보안 위협도 함께 증가합니다.  
SQL Injection, XSS, RCE 등 다양한 공격으로부터 서비스를 보호하기 위해 L7 WAF(Web Application Firewall)를 도입하게 되었고, 그 과정에서 ModSecurity를 선택한 이유와 구축 경험을 공유합니다.

## 도입 배경

웹 애플리케이션을 운영하면서 다음과 같은 보안 이슈들을 경험하게 되었습니다:

### 1. 증가하는 웹 공격 시도

서비스를 운영하면서 시간이 지날수록 보안 공격 시도가 점차 증가하는 것을 확인할 수 있었습니다. 서비스가 성장하고 사용자가 늘어나면서 공격자들의 타겟이 되었고, 실제 서비스 로그를 분석한 결과 다음과 같은 공격 패턴들이 지속적으로 감지되었습니다:

- **무작위 리소스 스캔 공격**: 존재하지 않는 경로에 대한 자동화된 스캔 시도 (/.env, /admin, /phpmyadmin, /wp-admin 등)
- **SQL Injection**: 데이터베이스 조작을 시도하는 악의적인 쿼리
- **XSS (Cross-Site Scripting)**: 스크립트 삽입을 통한 사용자 정보 탈취 시도
- **Path Traversal**: 파일 시스템 경로 조작을 통한 민감 정보 접근 시도
- **RCE (Remote Code Execution)**: 원격 코드 실행 공격

### 2. L3/L4 보안 솔루션과 코드 레벨 보안의 한계

**AWS ACL/Security Group 등 L3/L4 보안 솔루션의 한계:**
AWS Security Group, Network ACL과 같은 L3/L4 레벨 보안 솔루션은 IP 주소, 포트, TCP/UDP 프로토콜 기반으로 트래픽을 제어합니다.

예를 들어:
- Security Group: 특정 IP에서 443 포트 허용
- Network ACL: 특정 IP의 80 포트 접근 차단

하지만 이러한 솔루션들은 HTTP/HTTPS 요청의 **내용(페이로드)**을 검사할 수 없습니다. 따라서 정상적인 IP에서 80/443 포트로 들어오는 SQL Injection, XSS와 같은 애플리케이션 레벨 공격을 탐지하거나 차단할 수 없습니다.

**코드 레벨 보안의 한계:**
Spring Security와 같은 애플리케이션 프레임워크는 인증/인가, CSRF 방어 등 애플리케이션 로직 레벨의 보안을 담당합니다. 하지만 모든 입력값을 코드에서 검증하는 것은:
- 개발자의 실수나 누락 가능성이 존재
- 프레임워크나 라이브러리의 제로데이 취약점에 즉각 대응 어려움
- 비즈니스 로직에 도달하기 전 사전 차단이 불가능

**L7 WAF의 필요성:**
따라서 L3/L4 보안 솔루션과 애플리케이션 코드 사이에서, HTTP 요청의 **내용**을 분석하고 악의적인 패턴을 사전에 차단할 수 있는 **L7 계층 방화벽(WAF)**이 필요합니다.

### 3. 국제 표준 보안 Best Practice 적용

OWASP(Open Web Application Security Project)는 웹 애플리케이션 보안의 국제 표준으로, 전 세계 보안 전문가들이 검증한 Best Practice를 제공합니다.

단순히 규정을 준수하는 것을 넘어서, **OWASP Top 10**과 **OWASP CRS(Core Rule Set)**를 직접 경험하고 적용함으로써:
- 보안 위협에 대한 실질적인 대응 역량 강화
- 검증된 보안 패턴을 시스템에 통합
- 지속적인 룰셋 업데이트를 통한 점진적 보안 고도화

이를 통해 단발성 보안 조치가 아닌, **시스템의 점진적이고 지속가능한 보안 강화** 체계를 구축하고자 했습니다.

## 왜 ModSecurity인가?

여러 WAF 솔루션을 검토한 결과, ModSecurity를 선택하게 된 이유는 다음과 같습니다:

### 1. 오픈소스 기반의 유연성

ModSecurity는 오픈소스 WAF 엔진으로, 상용 솔루션 대비 다음과 같은 장점이 있습니다:
- **비용 효율성**: 라이선스 비용 없이 무료로 사용 가능
- **커스터마이징**: 소스 코드 수준에서 수정 및 최적화 가능
- **커뮤니티 지원**: 활발한 오픈소스 커뮤니티와 풍부한 레퍼런스

### 2. OWASP CRS (Core Rule Set) 지원

ModSecurity는 OWASP에서 관리하는 Core Rule Set을 기본으로 제공합니다:
- **검증된 보안 룰**: 전 세계 보안 전문가들이 관리하는 룰셋
- **지속적인 업데이트**: 새로운 공격 패턴에 대한 신속한 대응
- **세밀한 튜닝**: 각 룰별로 활성화/비활성화 및 예외 처리 가능

### 3. Nginx와의 완벽한 통합

기존에 사용 중이던 Nginx 웹 서버와 모듈 형태로 통합할 수 있어:
- **성능 오버헤드 최소화**: 별도의 프록시 없이 직접 통합
- **간편한 관리**: Nginx 설정 파일 내에서 통합 관리
- **안정성**: 프로덕션 환경에서 검증된 안정적인 조합

### 4. 유연한 로깅과 모니터링

- **상세한 감사 로그**: 차단된 요청에 대한 상세 정보 기록
- **실시간 모니터링**: 공격 패턴 분석 및 대응 가능
- **False Positive 관리**: 오탐 케이스를 쉽게 식별하고 예외 처리 가능

---

## ModSecurity WAF 구축 과정

### 환경 정보

- **버전**: Nginx 1.29.2 + ModSecurity v3 + OWASP CRS
- **대상 OS**: Amazon Linux 2 / CentOS 7 이상
- **구성 요소**:
  - **ModSecurity**: 오픈소스 WAF 엔진
  - **OWASP CRS**: 공격 패턴 탐지 룰셋 ([coreruleset.org](https://coreruleset.org/))

### 1. ModSecurity 설치

ModSecurity와 필요한 의존성 라이브러리를 설치합니다:

```bash
# 필수 패키지 설치
yum install yajl-devel git gcc-c++ flex bison curl-devel curl libxml2-devel \
    doxygen zlib-devel git automake libtool pcre-devel GeoIP-devel \
    lua-devel wget openssl-devel pcre2 pcre2-devel

# LMDB 설치
cd /opt/ && git clone https://github.com/LMDB/lmdb.git
cd lmdb/libraries/liblmdb
make && make install

# SSDEEP 설치
cd /opt/ && git clone https://github.com/ssdeep-project/ssdeep
cd ssdeep/
./bootstrap && ./configure && make && make install

# libmodsecurity 설치
cd /opt/ && git clone https://github.com/owasp-modsecurity/ModSecurity
cd ModSecurity
git submodule init && git submodule update
./build.sh && ./configure && make && make install
```

### 2. Nginx ModSecurity 모듈 빌드

기존 Nginx에 ModSecurity 모듈을 추가합니다:

```bash
# Nginx Connector 다운로드
cd /opt
git clone --depth 1 https://github.com/owasp-modsecurity/ModSecurity-nginx.git

# Nginx 소스 다운로드 및 빌드
nginx -V  # 현재 버전 확인
cd /usr/local/src
wget https://nginx.org/download/nginx-1.29.X.tar.gz
tar -xzvf nginx-1.29.X.tar.gz
cd nginx-1.29.X/

# 모듈 컴파일
./configure --with-http_ssl_module --with-http_v2_module \
    --with-compat --add-dynamic-module=/opt/ModSecurity-nginx/
make modules

# 모듈 복사
cp objs/ngx_http_modsecurity_module.so /etc/nginx/modules/
```

### 3. OWASP CRS 설정

OWASP Core Rule Set을 다운로드하고 설정합니다:

```bash
# 디렉터리 생성
sudo mkdir -p /etc/nginx/modsecurity
cd /etc/nginx/modsecurity

# CRS 다운로드
sudo git clone https://github.com/coreruleset/coreruleset.git owasp-crs
sudo cp owasp-crs/crs-setup.conf.example /etc/nginx/modsecurity/crs-setup.conf

# 로그 디렉터리 생성
sudo mkdir -p /var/log/modsecurity
sudo chown -R nginx:nginx /var/log/modsecurity
```

### 4. ModSecurity 핵심 설정

ModSecurity의 핵심 동작을 정의하는 설정 파일을 생성합니다:

```bash
# 캐시 디렉터리 생성
mkdir -p /var/cache/modsecurity/tmp
mkdir -p /var/cache/modsecurity/data

# 설정 파일 생성
vi /etc/nginx/modsecurity/modsecurity.conf
```

**modsecurity.conf 내용**:

```nginx
# ModSecurity 핵심 설정
SecRuleEngine On                        # 룰 적용: On=검사+차단, DetectionOnly=로그만
SecRequestBodyAccess On                 # 요청 본문 검사
SecResponseBodyAccess Off               # 응답 본문 검사 (성능 고려)

# 감사 로그 설정
SecAuditEngine RelevantOnly                 # 차단 이벤트만 로깅
SecAuditLogRelevantStatus "^(?:5|4(?!04))"  # 5xx, 4xx (404 제외) 로깅
SecAuditLogParts ABIJDEFHZ                  # 요약 정보만 기록
SecAuditLogType Serial                      # 순차적 로그 기록
SecAuditLog /var/log/modsecurity/modsec_audit.log  # 감사 로그 파일 경로

# 요청 크기 제한
SecRequestBodyLimit 52428800            # 최대 50MB
SecRequestBodyNoFilesLimit 2097152      # 파일 없는 요청 최대 2MB

# 요청 본문이 SecRequestBodyLimit를 초과할 경우 처리 방식
# ProcessPartial -> 본문 일부만 읽고 검사, 나머지는 통과
# Reject -> 기본값, 제한 초과 시 차단
SecRequestBodyLimitAction ProcessPartial

# 임시 디렉터리
SecTmpDir /var/cache/modsecurity/tmp
SecDataDir /var/cache/modsecurity/data
```

### 5. 로그 로테이션 설정

로그 파일이 과도하게 커지지 않도록 로테이션을 설정합니다:

```bash
sudo vi /etc/logrotate.d/modsecurity
```

**로그 로테이션 설정**:

```
/var/log/modsecurity/modsec_audit.log {
    daily
    missingok
    rotate 93
    compress
    delaycompress
    notifempty
    create 640 nginx adm
    sharedscripts
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 6. 커스텀 룰 설정

실제 운영 환경에서는 오탐(False Positive)으로 인해 정상 요청이 리젝될 수 있습니다. 이를 예외 처리하기 위한 커스텀 룰을 작성합니다:

```bash
vi /etc/nginx/modsecurity/custom_rules.conf
```

**주요 예외 처리 예시**:

```nginx
# ID 1000001
# ELB HealthCheck 예외
# 조건:
#   - User-Agent = "ELB-HealthChecker"
#   - Source IP = {{PrivateIP}}
# 제외 룰:
#   - 920350 (Host header is numeric IP)
SecRule REQUEST_HEADERS:User-Agent "ELB-HealthChecker" \
    "phase:1,id:1000001,nolog,pass,chain"
    SecRule REMOTE_ADDR "@ipMatch {{PrivateIP}}" \
        "ctl:ruleRemoveById=920350"
        
# ID 1000004
# PUT / DELETE / PATCH 허용
# 제외 룰:
#   - 911100 (Method enforcement)
SecRule REQUEST_METHOD "^(PUT|DELETE|PATCH)$" \
    "phase:1,id:1000004,nolog,pass,\
    ctl:ruleRemoveById=911100"
```

### 7. Nginx 통합 설정

```bash
vi /etc/nginx/modsecurity/main.conf
```

**main.conf 내용**:

```nginx
# 핵심 설정 포함
Include /etc/nginx/modsecurity/modsecurity.conf

# 커스텀 룰 (오탐 예외처리)
Include /etc/nginx/modsecurity/custom_rules.conf

# OWASP CRS
Include /etc/nginx/modsecurity/crs-setup.conf
Include /etc/nginx/modsecurity/owasp-crs/rules/*.conf
```

**nginx.conf에 ModSecurity 활성화**:

```bash
vi /etc/nginx/nginx.conf
```

```nginx
# 최상단에 모듈 로드
load_module /etc/nginx/modules/ngx_http_modsecurity_module.so;

# http {} 블록 안에 활성화
http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsecurity/main.conf;

    # ... 이하 생략 ...
}
```

### 8. 적용 및 테스트

```bash
# 설정 검증
nginx -t

# Nginx 재시작
nginx -s reload
```

**동작 테스트**:

```bash
# SQL Injection 테스트
curl -i --get --data-urlencode "id=1' OR '1'='1" "https://your-domain.com"
curl -i --get --data-urlencode "q=1 UNION SELECT username,password FROM users" "https://your-domain.com"
```

<img src="https://github.com/user-attachments/assets/b561a8dd-04fb-45ae-822f-c352e3604b0e" alt="ModSecurity SQL Injection 차단 로그" >

위 이미지(민감 정보는 블러 처리)와 같이 SQL Injection 공격을 요청하면 **owasp-crs/rules/REQUEST-932-APPLICATION-ATTACK-RCE** 룰셋에 의해 **403 Forbidden** 응답을 WAF 단에서 처리하는 것을 확인할 수 있습니다.

로그의 `--H--` 섹션을 분석하면 어떠한 룰셋에 의해 리젝되었는지 상세히 조회할 수 있습니다.

### 9. 로그 모니터링

```bash
# Nginx 에러 로그
tail -f /var/log/nginx/error.log

# ModSecurity 감사 로그
tail -f /var/log/modsecurity/modsec_audit.log
```

**로그 분석 방법:**
- `modsec_audit.log`에서 **`---H--`** 섹션이 있는 케이스가 실제로 WAF에 의해 차단된 요청입니다.
- `---H--` 섹션에는 어떤 OWASP CRS 룰셋(예: 942100-SQLi, 941100-XSS)에 의해 차단되었는지 상세 정보가 기록됩니다.
- 이 정보를 통해 오탐(False Positive) 여부를 판단하고 커스텀 룰로 예외 처리할 수 있습니다.

---

## 운영 시 주의사항

### 1. ⚠️중요: 오탐(False Positive) 관리

ModSecurity를 처음 적용할 때는 다음 단계를 권장합니다:

```nginx
SecRuleEngine DetectionOnly  # 로그만 기록, 차단하지 않음
```

- 최소 1~2주간 `DetectionOnly` 모드로 운영하며 로그 모니터링
- 정상 트래픽이 차단되는 패턴 식별
- `custom_rules.conf`에 예외 규칙 추가
- 안정화 후 `SecRuleEngine On`으로 전환

### 2. 성능 영향 최소화

- `SecResponseBodyAccess Off`: 응답 본문 검사 비활성화로 성능 향상
- `SecAuditEngine RelevantOnly`: 필요한 로그만 기록

### 3. 지속적인 룰 업데이트

```bash
cd /etc/nginx/modsecurity/owasp-crs
git pull origin main
nginx -s reload
```

OWASP CRS는 주기적으로 업데이트되므로, 새로운 공격 패턴에 대응하기 위해 정기적인 업데이트가 필요합니다.

---

## 결론

### 1. ModSecurity 도입 효과

ModSecurity WAF를 도입한 후 다음과 같은 긍정적인 효과를 얻을 수 있었습니다:

- **공격 차단율 향상**: SQL Injection, XSS 등 일반적인 웹 공격 시도를 사전에 차단
- **가시성 확보**: 감사 로그를 통해 어떤 공격이 얼마나 시도되는지 정량적으로 파악
- **빠른 대응**: 새로운 공격 패턴 발견 시 커스텀 룰로 신속하게 대응 가능
- **규정 준수**: 보안 감사 시 OWASP Top 10 대응 체계 입증

### 2. 오픈소스 WAF의 가치

상용 WAF 솔루션과 비교했을 때, ModSecurity는:
- **비용 효율성**: 라이선스 비용 없이 엔터프라이즈급 보안 제공
- **투명성**: 오픈소스이므로 내부 동작 원리를 정확히 파악 가능
- **커뮤니티**: 활발한 커뮤니티를 통한 빠른 문제 해결

물론 초기 설정과 튜닝에 시간이 필요하지만, 장기적으로 보면 충분히 투자할 가치가 있습니다.

### 3. 보안은 계층적 접근이 필요하다

ModSecurity WAF는 강력한 보안 도구이지만, 이것만으로 모든 보안 이슈를 해결할 수는 없습니다:

- **애플리케이션 레벨 검증**: 입력값 검증, 파라미터 바인딩 등 코드 레벨 보안 필수
- **인프라 레벨 보안**: 네트워크 방화벽, VPC 격리, IAM 정책 등 인프라 보안 병행
- **모니터링 및 대응**: 로그 분석, 이상 탐지, 침해 대응 프로세스 구축

ModSecurity는 이러한 다층 보안 전략의 중요한 한 축으로, **애플리케이션 레벨의 첫 번째 방어선** 역할을 충실히 수행합니다.

---

## 참고 자료

- [ModSecurity 공식 문서](https://github.com/owasp-modsecurity/ModSecurity)
- [ModSecurity-nginx Connector](https://github.com/owasp-modsecurity/ModSecurity-nginx)
- [OWASP Core Rule Set](https://github.com/coreruleset/coreruleset)
- [ModSecurity v3 컴파일 가이드](https://github.com/owasp-modsecurity/ModSecurity/wiki/Compilation-recipes-for-v3.x#amazon-linux-2)
- [Hoing.io - ModSecurity 설정 가이드](https://hoing.io/archives/9487#Nginx)
- [LiveYourIT - ModSecurity 구축기](https://liveyourit.tistory.com/11)
