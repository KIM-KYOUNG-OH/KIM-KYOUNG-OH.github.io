---
layout: post
title: 항해 Plus 1주차 회고(WIL)
categories: 회고
date: 2024-12-22
banner:
  image: https://github.com/user-attachments/assets/0d8d4116-3471-4654-82e8-2b5b558ccc85
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: clean-code test concurrency-control
sidebar: []
comments: true
---

항해 Plus 1주차 핵심 주제는 ‘클린 코드 & 테스트’ 에 대한 내용이었습니다.

과제는 간단한 포인트 조회/적립 기능을 구현하는 API를 만들어보는 내용이었고 거기에 동시성 이슈까지 해결해보는 내용이었습니다.

## 클린 코드

최대한 **책임의 분리 원칙**을 초점으로 코드를 작성하기 위해 노력했습니다.

포인트 충전 함수의 로직은 크게 동시성 제어 로직과 포인트를 충전하는 DB IO 작업으로 나뉘어집니다. 동시성 제어 로직은 핵심 비즈니스와 관심사는 아니라고 판단하여 별도의 클래스로 책임을 분리하여 작성했습니다.

도메인 객체의 내부 성분은 외부로부터 캡슐화하고 객체 자신을 수정하는 로직은 도메인 클래스의 멤버 함수로 제공하는 방식을 사용했습니다.

## 테스트

단위 테스트와 통합 테스트의 사용 목적과 사용법에 대해서 익힐 수 있었습니다.

단위 테스트는 외부 의존성과 격리된 환경에서 순수 자기 자신만 테스트하는 것을 목적으로 하는 테스트입니다.

통합 테스트는 외부 의존성과 연동되어 전체적으로 시스템이 전반적으로 의도대로 동작하는지를 목적으로 하는 테스트입니다.

TDD라는 Test 먼저 작성하면서 비즈니스를 설계하는 방법론도 있지만 현업에선 불가능하다는 의견이 많습니다. 테스트도 결국 요구사항 분석과 도메인에 대한 이해가 바탕이 되어야 하기 때문입니다.

line coverage와 branch coverage 에 대해서도 학습할 수 있었는데 line coverage 100%는 불필요한 테스트까지 작성해야하기 때문에 현업에서는 채우기 어렵고 brach coverage 100% 까지는 현업에서 도입해볼만 하다고 생각됩니다.

branch coverage는 if와 같은 조건절 분기에 대한 참/거짓이 모두 실행되었는지를 측정하는 지표입니다.

## 동시성 제어

해당 과제에서는 애플리케이션 레벨에서 동시성 문제를 해결하기 위한 방법에 초점을 맞췄습니다.

java Concurrent 패키지에서 지원하는 ConcurrentHashMap과 ReentrantLock을 사용해서 동시성 이슈를 해결했습니다.

ConcurrentHashMap은 세그먼트 단위로 자원에 대한 스레드 접근을 제어하기 때문에 Thread-safe를 보장하고 동시성 성능까지 높일 수 있는 자료구조 입니다.

ReentrantLock은 락 획득과 반납을 빌트인 함수로 간편하게 제공하고 이전 스레드가 락을 반납할 때까지 이후 스레드가 blocking되는 방식으로 구현됩니다.

단, 락이 반환되는 속도보다 요청이 쌓이는 속도가 더 빠른 환경에선 Thread Pool의 스레드가 마르거나 blocking 되는 자원이 커지면서 OOME가 발생할 우려가 있기 때문에 대규모 시스템에선 Zookeeper나 Redis 같은 독립된 인메모리 인스턴스를 사용해서 동시성 문제를 해결하는 방법도 고려해볼 수 있습니다.

## 과제 소스코드 링크

👉 [과제 메인 깃헙 레포지토리 Link](https://github.com/KIM-KYOUNG-OH/hhplus-1st-week)  

👉 [기본 과제 PR Link](https://github.com/KIM-KYOUNG-OH/hhplus-1st-week/pull/1)

👉 [동시성 이슈 테스트 보고서 작성 PR Link](https://github.com/KIM-KYOUNG-OH/hhplus-1st-week/pull/2)

## KPT 회고

### Keep

- 단위 테스트를 작성하는 습관
- 동시성 이슈를 항상 고민해보는 습관

### Problem

- ReentrantLock 같은 경우는 fairness 문제가 있는데 기술 도입 전에 기술에 대한 깊이 있는 이해와 도입 이유가 분명해야한다
- 기술을 나만의 언어로 설명할 수 있는 수준의 이해가 부족함

### Try

- 다음 동시성 이슈 문제 해결시 Compare And Swap 개념 도입
- ModelMapper를 이용한 레이어간 DTO 변환 로직 작성

## 과제 제출 결과

![스크린샷 2024-12-23 000009](https://github.com/user-attachments/assets/25987210-aa29-4ccd-a21d-979dd8903cec)  

![스크린샷 2024-12-23 000910](https://github.com/user-attachments/assets/2508e65b-81d6-4dd0-b060-eb735332f91b)  

![스크린샷 2024-12-23 000924](https://github.com/user-attachments/assets/8e6f73db-1d55-4165-b3ae-91bdc284de9a)  


