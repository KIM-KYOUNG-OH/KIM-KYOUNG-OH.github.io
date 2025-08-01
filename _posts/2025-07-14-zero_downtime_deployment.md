---
layout: post
published: true
title: 배포가 두렵다면? 무중단 배포 전략
categories: Infra
date: 2025-07-14
banner:
  image: https://github.com/user-attachments/assets/9f571387-6d43-4cee-8eb1-a612c079a2b4
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-weight: bold;"
  subheading_style: "color: gold"
tags: zero-downtime-deployment infra CI/CD
sidebar: []
comments: true
---

서비스 배포할 때마다 손이 떨리고 로그창만 바라보며 기도하게 되시나요?  
"이거 올렸다가 터지면 어떡하지..." 하는 불안, 누구나 겪어봤을 겁니다.

하지만 걱정 마세요.  
**다운타임 없이**, **문제 생겨도 빠르게 복구**할 수 있는  
3가지 무중단 배포 전략을 소개합니다.

1. 블루그린 배포 (Blue-Green)
2. 카나리 배포 (Canary)
3. 롤링 배포 (Rolling)

왜 무중단 배포를 도입해야하는가? 
---

무중단 배포(Zero Downtime Deployment)의 가장 큰 목적은 **서비스 중단 없이 안정적으로 새로운 버전을 배포하는 것**입니다.

- **고가용성(High Availability, HA)**: 사용자에게 끊김 없는 경험 제공
- **안전한 롤백**: 배포 실패 시 빠르게 이전 상태로 복구 가능
- **사용자 영향 최소화**: 일부 사용자에게만 점진 적용 가능
- **배포 자동화와 일관성 확보**: 실수 없이 반복 가능한 배포
- **빠른 피드백 루프**: 자주 배포하고 즉시 개선할 수 있는 구조 지원

서비스의 신뢰성과 배포 효율성을 동시에 잡기 위해 무중단 배포는 이제 선택이 아닌 **필수 전략**입니다.

블루그린 배포 (Blue-Green Deployment)
--- 

**기존 운영 환경(Blue)** 과 **새로운 버전(Green)** 을 **동시에 운영** 하며, **새 버전(Green)** 을 배포한 뒤, **트래픽을 Green으로 전환** 합니다.   
문제가 없다면 Green을 새로운 운영 환경으로 사용하고, 문제가 생기면 다시 Blue로 전환하여 **즉시 롤백** 이 가능합니다.

<img src="https://github.com/user-attachments/assets/f02c4474-b314-4230-80cc-e8757937ea79" alt="Image" width="3016" height="1428" />  

### 장점
- 테스트 완료된 환경으로만 전환 → 안정적
- 빠른 롤백

### 단점
- 블루, 그린 환경 2개 유지 필요 → **높은 운영 비용**
- 블루, 그린 모두 같은 데이터베이스를 바라보기 때문에 두 버전 모두 호환되도록 DB 스키마를 관리해야하고 점진적으로 마이그레이션 해야함 -> **DB 스키마 변경 시 관리 복잡** 

### 활용 예시
- 리스크가 큰 기능 출시
- 빠른 롤백이 중요한 상황

카나리 배포 (Canary Deployment)
---

카나리 배포(Canary Deployment)라는 이름은 과거 광산에서 ‘카나리아(canary)’ 새를 위험 신호 탐지에 활용한 것에서 유래했는데, 새가 먼저 유해 가스에 반응해 광부들이 신속히 대피할 수 있었기 때문입니다.    
카나리 배포는 새 버전을 먼저 일부 사용자에게만 적용해 문제를 확인하고, 모니터링 이후에 이상 없으면 점차 모든 사용자에게 확대하는 점진적 배포 방식입니다.  
일부 실사용자에게만 새 버전을 노출하여 리스크를 점검하는 목적으로 사용합니다.   

<img src="https://github.com/user-attachments/assets/1d4ddb72-a07e-4a58-a211-4cdf0c83edf7" alt="Image" width="1041" height="529" />  

### 장점
- 사용자 중 일부만 새 버전을 먼저 사용 -> **실제 사용자 기반 테스트 가능**  
- 문제가 생겨도 영향 범위 제한

### 단점
- 트래픽 분할, 모니터링, 자동화 등 복잡한 운영 필요
- 롤백이 단순 스위치가 아니라 점진적 롤백이므로 느림  

### 활용 예시
- 트래픽이 많은 서비스
- 사용자 반응을 기반으로 기능 적용 여부 판단할 때

롤링 배포 (Rolling Deployment)
---

기존 서버 인스턴스를 **하나씩 교체**하며 새 버전을 배포하는 방법입니다.  
이 방식은 별도의 추가 환경을 마련하지 않고도 배포할 수 있기 때문에, 가용한 인프라 자원이 제한적인 상황에서 매우 효율적입니다.  

<img width="1599" height="662" alt="Image" src="https://github.com/user-attachments/assets/ba1a850e-6e09-4ef8-861e-38ce791a65e8" />

### 장점
- 리소스 절약 (이중 환경 불필요)  

### 단점
- 배포 중 구버전/신버전 혼재 가능
- 문제 발생 시 점진적으로 반영되기 때문에 **롤백이 느림**

### 활용 예시
- 서버 자원이 제한적인 환경
- 단일 서비스 운영 구조

전략별 비교 요약
---

| 전략      | 다운타임 없음 | 롤백 용이성               | 리소스 비용       | 배포 속도 | 운영 복잡도 | 사용자 영향 제어     |
|-----------|---------------|----------------------|----------------|-----------|-------------|---------------|
| 블루그린   | ✅             | 높음 (즉시 롤백 가능)        | 높음 (이중 환경 유지) | 빠름      | 중간        | 전체 사용자 전환     |
| 카나리     | ✅             | 중간 (일부 점진적 롤백)     | 중간 (부분 인스턴스 운영) | 느림      | 높음        | 일부 사용자 대상     |
| 롤링       | ✅             | 낮음 (전체 점진적 롤백, 느림) | 낮음 (추가 환경 불필요) | 중간      | 낮음        | 전체 사용자 점진적 전환 |

어떤 전략을 선택해야 할까?  
---

| 상황                              | 추천 전략   |
|-----------------------------------|-------------|
| 빠르게 롤백할 수 있어야 한다        | 블루그린     |
| 점진적으로 사용자 반응을 보고 싶다 | 카나리       |
| 리소스가 부족하다                 | 롤링         |

정리
---

무중단 배포가 아직 도입되지 않았다면, 서비스 환경과 리소스 상황에 맞춰 무중단 배포 전략을 선택하는 것을 권장합니다.
- 롤링 배포: 구현 쉽고 인프라 부담 적음. 자원 제한된 환경에 적합  
- 카나리 배포: 일부 사용자에 점진 적용, 리스크 관리에 효과적. 사용자 반응에 민감한 서비스 추천  
- 블루그린 배포: 완전 분리 환경, 즉시 롤백 가능. 대규모 서비스나 빠른 복구 필요 시 적합  

추가적으로 다음 키워드들도 무중단 배포 시스템을 더욱 고도화하는 데 도움이 될 수 있습니다.
- Kubernetes 무중단 배포 전략
- Service Mesh
- Autoscaling
- Health Check
- Liveness/Readiness/Startup Probe

참고
--- 

[https://velog.io/@jingrow](https://velog.io/@jingrow/%EB%B8%94%EB%A3%A8%EA%B7%B8%EB%A6%B0-%EB%A1%A4%EB%A7%81-%EC%B9%B4%EB%82%98%EB%A6%AC-%EB%B0%B0%ED%8F%AC%EC%9D%98-%EC%B0%A8%EC%9D%B4%EC%A0%90%EA%B3%BC-%EC%96%B4%EB%96%A4-%EA%B2%BD%EC%9A%B0%EC%97%90-%EA%B0%81%EA%B0%81%EC%9D%98-%EB%B0%B0%ED%8F%AC%EB%B0%A9%EC%8B%9D%EC%9D%84-%EC%8B%9C%EB%8F%84%ED%95%98%EB%8A%94%EC%A7%80-%EC%A1%B0%EC%82%AC%ED%95%B4%EB%B3%B4%EC%84%B8%EC%9A%94)
