---
layout: post
published: true
title: 무중단 배포 환경 구축하기 with AWS CodeDeploy
author: 김경오
categories: Infra
date: 2025-12-07
banner:
  image: https://github.com/user-attachments/assets/5e548f0d-f0f0-4e3e-90e2-8c4af15649b6
  opacity: 0.618
  background: "#000"
  height: "30vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: jenkins aws code-deploy amazon-sns aws-lambda
sidebar: []
---

안정적인 서비스 운영을 위해서는 배포 과정에서 발생할 수 있는 다운타임을 최소화하는 것이 중요합니다.  
서버가 2대 이상으로 이중화되어 있고 AWS ELB(Elastic Load Balancer)를 통해 트래픽이 분산 처리되고 있다고 가정해봅시다.  
수동 배포 시에는 배포 중인 서버로의 트래픽을 수동으로 차단하고, 배포 완료 후 다시 트래픽을 허용하는 작업을 반복해야 하는 번거로움이 있습니다.  
또한 수동 작업 과정에서 실수로 인한 서비스 장애나 요청 유실이 발생할 수 있는 위험도 존재합니다.

이러한 문제를 해결하고 안정적인 무중단 배포 환경을 구축하기 위해 AWS CodeDeploy를 고려해볼 수 있습니다.  
이번 포스팅에서는 CodeDeploy와 Jenkins를 활용하여 롤링 배포 전략을 구현한 과정을 공유합니다.

## 왜 CodeDeploy 인가?

### 1. 비용 효율성
CodeDeploy는 EC2 인스턴스에 배포할 경우 **추가 비용이 발생하지 않습니다**.  
AWS 인프라를 사용하고 있다면 CodeDeploy Agent만 설치하면 바로 사용할 수 있어, 별도의 배포 서버나 도구 없이도 무중단 배포 환경을 구축할 수 있습니다.

### 2. AWS 인프라와의 높은 호환성
CodeDeploy는 AWS의 네이티브 서비스로, ELB(Elastic Load Balancer)와 긴밀하게 통합되어 있습니다.  
배포 과정에서 자동으로 다음과 같은 작업이 수행됩니다:  
- **배포 시작 전**: 대상 인스턴스를 ELB에서 자동으로 등록 해제(deregister)하여 트래픽 차단
- **배포 완료 후**: 배포가 성공하면 다시 ELB에 자동 등록(register)하여 트래픽 허용

이러한 자동화로 인해 수동 작업 없이도 무중단 배포가 가능하며, 관리 포인트가 줄어들어 운영 부담이 감소합니다.

### 3. 안정적인 배포 전략 지원
롤링 배포, 블루-그린 배포 등 다양한 배포 전략을 지원하며, 배포 실패 시 자동 롤백 기능을 제공하여 서비스 안정성을 높일 수 있습니다.  

### 배포 아키텍처

<img src="https://github.com/user-attachments/assets/5e548f0d-f0f0-4e3e-90e2-8c4af15649b6" alt="architecture" width="1200">

AWS 인프라 세팅
---

### 1. CodeDeploy 용 IAM Role 생성

AWSCodeDeployRole 권한 정책을 가진 Role 을 생성합니다.

<img src="https://github.com/user-attachments/assets/70e6380b-7206-4839-8e8e-0407898dfa47" alt="aws role" >

### 2. EC2용 IAM Role 생성

AmazonS3FullAccess, AWSCodeDeployFullAccess 권한 정책을 가진 Role 을 생성합니다.

<img src="https://github.com/user-attachments/assets/a8964f25-5783-4ec1-a546-82f13dbaf0b0" alt="ec2 role" >

생성한 IAM Role 을 EC2 와 연결합니다.(타겟 EC2 선택 -> 보안 -> IAM 역할 수정)

<img src="https://github.com/user-attachments/assets/0417a385-1558-4d81-9b73-db8ecb807621" alt="ec2 role2" >  

### 3. Jenkins 용 IAM 유저 생성

AmazonS3FullAccess, AWSCodeDeployFullAccess 권한 정책을 가진 IAM 유저를 생성합니다.

<img src="https://github.com/user-attachments/assets/500b05f9-4c13-4a0e-8d97-5c2a242a7b06" alt="IAM user" >

Jenkins 서버에서 접속하기 위한 액세스 키 & 시크릿 키를 발급합니다.(IAM 유저 요약 -> 액세스 키 만들기)

<img src="https://github.com/user-attachments/assets/a861afae-c5be-405c-8c2e-097972ed788d" alt="IAM user2" >

### 4. CodeDeploy 애플리케이션 및 배포 그룹 생성

CodeDeploy 메뉴에서 애플리케이션을 생성하고 배포할 EC2 인스턴스를 배포 그룹으로 설정합니다.  
1번에서 생성한 CodeDeploy Role 을 연결합니다.  
롤링 배포를 위해 OneAtATime 을 선택합니다.  
로드 밸런싱을 활성화 하면 배포 중인 서버는 트래픽을 차단하고 배포가 성공된 뒤에 다시 트래픽을 허용합니다.  

<img src="https://github.com/user-attachments/assets/328fd304-2165-41ca-98cc-b8830b826999" alt="codedeploy" >

로드밸런서 설정 내 드레인 타임 값을 조절하여 즉시 차단하지 않고 지연 시간을 줄 수 있습니다.(Default 300초)

<img src="https://github.com/user-attachments/assets/fe96dc06-b0ff-47e4-98d0-2327cd0eb0f0" alt="alb" >

### 5. CodeDeploy Agent 설치
원격 서버(EC2)에 ssh 접속 후 아래 스크립트로 CodeDeploy Agent를 설치합니다.
```shell
sudo yum update -y
sudo yum install ruby -y
sudo yum install wget -y
cd /home/ec2-user
wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo service codedeploy-agent start
sudo service codedeploy-agent status
```

### 6. 애플리케이션 설정
CodeDeploy 배포시 동작을 정의하기 위해 appspec.yml 파일과 hook 용 스크립트를 추가합니다.
- `appspec.yml` : CodeDeploy가 배포를 어떻게 수행할지 지시하는 설정 파일입니다.
- `upload_deploy_war.sh` : 번들 파일(.zip) install 성공시 이전 배포 버전을 아카이빙하고 신규 배포 파일로 교체합니다.
- `shutdown_application.sh` : 현재 동작중인 애플리케이션을 종료합니다.
- `startup_application.sh` : 신규 배포 버전으로 애플리케이션을 실행합니다.
- `validate.sh` : 애플리케이션이 정상 실행되었는지 검증합니다.
- `clean_and_backup.sh` : 배포중 에러가 발생하면 아카이빙한 이전 버전으로 롤백합니다.

<img src="https://github.com/user-attachments/assets/d9839107-c2f4-4fc6-9401-8adb5df66584" alt="packages">  

### appspec.yml
번들파일(.zip) 최상위 경로에 필수로 위치해야합니다.  
hooks 스크립트 정의 순서는 배포시 반영되지 않습니다.  
```yaml
version: 0.0
os: linux
files:
  - source: /
    destination: /home/ec2-user/code-deploy/demo
permissions:
  - object: /home/ec2-user/code-deploy
    owner: ec2-user
    group: ec2-user
hooks:
  BeforeInstall:
    - location: scripts/clean_and_backup.sh
      timeout: 300
      runas: ec2-user
  AfterInstall:
    - location: scripts/upload_deploy_war.sh
      timeout: 300
      runas: ec2-user
  ApplicationStop:
    - location: scripts/shutdown_application.sh
      timeout: 180
      runas: ec2-user
  ApplicationStart:
    - location: scripts/startup_application.sh
      timeout: 300
      runas: ec2-user
  ValidateService:
    - location: scripts/validate.sh
      timeout: 300
      runas: ec2-user
```

### 7. Jenkins Plugin 설치
Jenkins 에서 AWS CodeDeploy 를 사용하기 위해 아래 플러그인을 설치합니다.  
- [AWS Credentials Plugin](https://plugins.jenkins.io/aws-credentials/)
- [AWS CodeDeploy Plugin](https://plugins.jenkins.io/codedeploy/)

<img src="https://github.com/user-attachments/assets/9e4b3a88-f7fb-4c26-8bd1-b641be310233" alt="jenkins plugins" >

### 8. Jenkins Item 등록
1) Jenkins Item 추가

<img src="https://github.com/user-attachments/assets/f50d8033-872a-41b2-af48-58d2999556c7" alt="jenkins item" >

2) git checkout 설정

<img src="https://github.com/user-attachments/assets/de6a81fe-1b67-4291-8ebf-e638d0756f19" alt="git" >

3) 빌드 및 번들 파일에 포함시킬 파일(appspec.yml, scripts/*) 타겟 경로로 복사

<img src="https://github.com/user-attachments/assets/4fcba20f-9be9-4a1c-9741-40ac17376e0e" alt="build" >

4) Jenkins Codedeploy step 추가

<img src="https://github.com/user-attachments/assets/2a41d4b8-2e10-4f23-8993-7acb546b8a29" alt="deploy" >

<img src="https://github.com/user-attachments/assets/d85ed70e-f176-4010-a4ec-24780146c2a4" alt="detail" >

Access/Secret Keys 값으로 3단계에서 생성한 IAM 유저의 키값을 입력합니다.

<img src="https://github.com/user-attachments/assets/d470702a-7292-48bf-9665-9ea77dc1acaf" alt="credentials" >  

배포 실행
---

Jenkins Item 을 빌드하면 AWS Console > CodeDeploy 메뉴에서 배포 수명 주기 이벤트를 실시간으로 확인할 수 있습니다.

<img src="https://github.com/user-attachments/assets/c2ac48f9-979a-45e1-a464-58d52c12f23c" alt="codedeploy1" >

<img src="https://github.com/user-attachments/assets/5418461d-07eb-47e0-ba7d-3761861ad5c3" alt="codedeploy2" >  

디버깅
---

원격 서버(EC2) 내 로그 경로
- **CodeDeploy 자체 로그** : 
  - `/var/log/aws/codedeploy-agent/codedeploy-agent.log`
- **CodeDeploy Agent 이벤트/상태 로그** : 
  - `/opt/codedeploy-agent/deployment-root/deployment-logs/codedeploy-agent-deployments.log`
- **배포 ID별 스크립트 로그** : 
  - `/opt/codedeploy-agent/deployment-root/<deployment-group-id>/<deployment-id>/logs/scripts.log`

배포 완료 Slack 알림 (선택)
---

<img src="https://github.com/user-attachments/assets/1d59bfc7-7751-4177-8a67-f1cc4c86796e" alt="slack notification" width="500">

자세한 내용은 아래 링크를 참고해주세요.  
👉 [https://galid1.tistory.com/749](https://galid1.tistory.com/749)  

정리
---

이번 포스팅에선 이중화된 서버 환경에서 AWS CodeDeploy와 Jenkins를 활용하여 무중단 배포 인프라를 구축하는 과정을 정리해보았습니다.  

**1. 무중단 배포 달성**
- ELB 자동 등록/해제를 통해 배포 중에도 서비스 중단 없이 안정적으로 배포할 수 있습니다.
- 롤링 배포 전략으로 한 대씩 순차적으로 배포하여 항상 최소 1대 이상의 서버가 요청을 처리합니다.

**2. 자동화를 통한 운영 효율성**
- CodeDeploy Agent가 배포 프로세스를 자동으로 수행하여 수동 작업에 따른 휴먼 에러를 방지합니다.
- Jenkins와 통합하여 CI/CD 파이프라인을 완성할 수 있습니다.

**3. 배포 시 고려사항**

- **롤백 전략**: 
  - appspec.yml의 hook 스크립트에서 배포 실패 시 이전 버전으로 자동 롤백하는 로직을 구현해야 합니다.

- **DB 스키마 변경**: 
  - Ver 1 → Ver 2 배포 과정에서 DB 스키마가 변경될 경우, **하위 호환성**을 반드시 고려해야 합니다.
    - 하위 호환 가능: DB 선 배포 후 WAS 점진적 배포
    - 하위 호환 불가능: 블루-그린 배포 전략 적용 고려 (리소스 비용 증가 트레이드오프)

- **드레인 타임 조정**: 
  - ELB의 Connection Draining 시간을 적절히 설정하여 진행 중인 요청이 안전하게 처리되도록 합니다.

참고
---

[https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/welcome.html](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/welcome.html)  
[https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/getting-started-create-service-role.html#getting-started-create-service-role-console](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/getting-started-create-service-role.html#getting-started-create-service-role-console)  
[https://jojoldu.tistory.com/314?category=777282](https://jojoldu.tistory.com/314?category=777282)  
[https://galid1.tistory.com/746](https://galid1.tistory.com/746)  
[https://jenakim47.tistory.com/63](https://jenakim47.tistory.com/63)  
[https://velog.io/@chanyoung1998/AWS-CodeDeploy-%EC%98%A4%EB%A5%98-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0](https://velog.io/@chanyoung1998/AWS-CodeDeploy-%EC%98%A4%EB%A5%98-%ED%95%B4%EA%B2%B0%ED%95%98%EA%B8%B0)  
