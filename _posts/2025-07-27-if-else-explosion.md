---
layout: post
published: true
title: if-else는 이제 그만, 전략 패턴으로 코드 개선하기
author: 김경오
categories: Design-Pattern
date: 2025-07-27
banner:
  image: https://github.com/user-attachments/assets/d56b47b5-7556-4ca3-82ec-e61122362fff
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: design-pattern spring java strategy-pattern 
sidebar: []
---

개발자라면 한 번쯤은 `if-else` 문으로 가득한 클래스를 만들어본 적이 있을 겁니다.  
하지만 시간이 지나고 코드가 커질수록, 우리는 더 깔끔하고 유연한 구조를 원하게 됩니다.  
이번 포스팅에서 `if-else` 코드를 줄이면서 코드를 단순하고 유지보수에 유리한 구조로 변경하는 방법을 알아 보겠습니다.

if-else 지옥
---
간단한 예시부터 시작해보겠습니다.  
당신은 배송 서비스를 개발 중이고, 현재는 도보, 트럭, 기차 세 가지 배송 수단을 지원하고 있습니다.
```java
public class DeliveryService { 
    public void delivery (String deliveryType) { 
        if ("WALK".equals(deliveryType)) { 
            // 도보
         } else if ("TRUCK".equals(deliveryType)) { 
            // 트럭
         } else if ("TRAIN".equals(deliveryType)) { 
            // 기차
         } 
    } 
}
```
그런데 어느날 기획팀에서 이렇게 말합니다.  
"배송수단 선박, 항공을 추가할 수 있을까요?"

이제 여러분의 클래스는 `else if` 블록이 계속 추가되며 점점 복잡해집니다.  
처음에는 단순했던 로직이 어느새 거대한 조건 분기문으로 뒤덮이게 되죠.

### if-else 방식의 문제점
- 조건이 늘어날수록 로직이 복잡해지고 유지보수가 어려워짐  
- 새로운 타입이 생길 때마다 기존 코드를 수정해야 함 → OCP 위반  
- 테스트, 가독성, 확장성 모두 악화

이제는 if-else 지옥에서 벗어나 더 유연하고 확장 가능한 구조로 나아갈 때입니다.  
바로, **전략 패턴(Strategy Pattern)** 을 통해 말이죠.

전략 패턴으로 구조 개선하기
---

전략 패턴(Strategy Pattern)은 알고리즘(또는 행위)을 캡슐화하여 여러 개의 알고리즘을 상호 교환 가능하게 만들고, 실행 시점에 적절한 알고리즘을 선택할 수 있도록 하는 디자인 패턴입니다.

즉, **"어떤 작업을 수행하는 방법(전략)을 클래스로 분리하고, 필요에 따라 해당 전략을 동적으로 바꿔가며 사용할 수 있게 하는 패턴"** 입니다.

이를 통해 조건문 분기를 줄이고, 확장성과 유지보수성을 높일 수 있습니다.

1.전략 인터페이스 정의
---
```java
public interface DeliveryStrategy {
    void deliver();
}
```

2.각 타입별 전략 클래스 구현
---

```java
@Component("WALK")
public class WalkDeliveryStrategy implements DeliveryStrategy {
  @Override
  public void deliver() {
    System.out.println("도보로 배송합니다.");
  }
}

@Component("TRUCK")
public class TruckDeliveryStrategy implements DeliveryStrategy {
  @Override
  public void deliver() {
    System.out.println("트럭으로 배송합니다.");
  }
}

@Component("TRAIN")
public class TrainDeliveryStrategy implements DeliveryStrategy {
  @Override
  public void deliver() {
    System.out.println("기차로 배송합니다.");
  }
}

@Component("SHIP")
public class ShipDeliveryStrategy implements DeliveryStrategy {
    @Override
    public void deliver() {
        System.out.println("선박으로 배송합니다.");
    }
}

@Component("AIRPLANE")
public class AirplaneDeliveryStrategy implements DeliveryStrategy {
  @Override
  public void deliver() {
    System.out.println("항공으로 배송합니다.");
  }
}
```

3.전략을 직접 주입받아 사용하는 서비스 클래스
---

```java
@Service
public class DeliveryService {
    
    private final Map<String, DeliveryStrategy> strategyMap;

    public DeliveryService(Map<String, DeliveryStrategy> strategyMap) {
        this.strategyMap = strategyMap;
    }

    public void deliver(String deliveryType) {
        DeliveryStrategy strategy = strategyMap.get(deliveryType);
        if (strategy == null) {
            throw new IllegalArgumentException("지원하지 않는 배송 타입: " + deliveryType);
        }
        strategy.deliver();
    }
}
```

전략 패턴 도입 전후 비교
---

| 항목             | if-else 방식                | 전략 패턴 방식                  |
|----------------| ------------------------- | ------------------------- |
| **유지보수 및 확장성** | ❌ 새로운 조건 추가 시 기존 코드 수정 필요 | ✅ 기존 코드는 그대로, 클래스만 추가하면 됨    |
| **가독성**        | ❌ 긴 조건문으로 인해 가독성 저하       | ✅ 각 전략은 역할이 명확, 읽기 쉬움     |
| **테스트 용이성**    | ❌ 전체 조건 로직 묶여 있어 테스트 어려움  | ✅ 전략별로 단위 테스트 가능          |

마무리
---

조건문은 개발 초기엔 빠르게 구현할 수 있는 좋은 수단이지만,
복잡도가 커지는 시스템에서는 유지보수의 큰 장애물이 될 수 있습니다.

전략 패턴은 단순한 디자인 패턴이지만,
적재적소에 적용했을 때 코드의 유연성과 확장성을 극적으로 개선할 수 있습니다.

더 이상 if-else 지옥에 갇히지 마세요.
전략 패턴으로 깔끔하고 유지보수 쉬운 코드를 만들어 보세요!


