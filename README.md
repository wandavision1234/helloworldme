# starcoffee
![coffee_log](https://user-images.githubusercontent.com/87048655/131767943-7f467f82-105a-4c4c-bdae-06026d8c6fc4.png)

# Table of contents

- [서비스 시나리오](#서비스-시나리오)
- [체크포인트](#체크포인트)
- [분석/설계](#분석설계)
- [구현](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [Gateway 적용](#gateway-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-fallback-처리)
    - [비동기식 호출 / 시간적 디커플링 / 장애격리](#비동기식-호출--시간적-디커플링--장애격리)
- [운영](#운영)
    - [Deploy](#Deploy)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출--서킷-브레이킹--장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
    - [Config Map](#config-map)
    - [Self-healing (Liveness Probe)](#self-healing-liveness-probe)

# 서비스 시나리오
커피주문 시스템인 starcoffee의 기능적, 비기능적 요구사항은 다음과 같습니다. 
사용자가 커피를 주문한 후 결제를 완료합니다. 담당자는 결제내역을 확인한 후 커피를 만듭니다. 고객은 주문 현황을 확인할 수 있습니다.

기능적요구사항
 1. 고객은 커피를 주문한다.
 2. 고객은 주문 후 결제 한다.
 3. 결제가 완료 되면 커피를 만든다.
 4. 결제가 취소되면 커피를 만들지 않는다.
 5. 고객이 주문상태를 조회 할 수 있다.

비기능적요구사항
 1. 결제가 완료 되지 않은 주문건은 커피를 만들지 않아야 한다.(Request-Response 방식 처리)
 2. 커피 만들 때 장애가 발생하더라도 주문은 계속 받아야 한다.(pub/sub)
 
# 체크포인트
- Saga
- CQRS
- Correlation
- Req/Resp
- Gateway
- Deploy/ Pipeline
- Circuit Breaker
- Autoscale (HPA)
- Zero-downtime deploy (Readiness Probe)
- Config Map/ Persistence Volume
- Polyglot
- Self-healing (Liveness Probe)

## 이벤트 스토밍 결과
MSAEZ에서 Event Storming 수행
Event 도출

### 이벤트 도출

- Event 도출
![01 이벤트도출](https://user-images.githubusercontent.com/87048655/131767590-3049111f-c900-44e1-a901-8d993c0f3763.png)

- Policy 부착
![02  policy 추가](https://user-images.githubusercontent.com/87048655/131767591-373d1387-6d8d-41b7-8383-d5b482afd992.png)

- Command 부착
![03  command추가](https://user-images.githubusercontent.com/87048655/131767593-72f97002-6599-4ceb-a615-95b3e9466031.png)

- Actor 부착
![04 actor추가](https://user-images.githubusercontent.com/87048655/131767595-cb616d69-ccb4-4fa1-8f72-27ad9dd9f635.png)

- Aggregate 부착
![05 aggregate추가](https://user-images.githubusercontent.com/87048655/131767596-d2159de7-742c-4530-a1af-10a21e858431.png)

- Bounded Context 묶기
![06 bounded context도출](https://user-images.githubusercontent.com/87048655/131767597-21ef6544-a7b5-4069-b5da-41b6b15b7f43.png)

- View 추가
![07 cqrs](https://user-images.githubusercontent.com/87048655/131768198-f2570b31-c25a-444f-be94-54629748360c.png)

- 완성 모형: Pub/Sub, Req/Res 추가(점선은 Pub/Sub, 실선은 Req/Resp)
![08 Context 매핑](https://user-images.githubusercontent.com/87048655/131767601-2b0b35b1-bb14-4405-8bfe-8f780bffb934.png)


### 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![헥사고날](https://user-images.githubusercontent.com/87048655/131712015-85a0258e-9212-4dde-a906-e53b3dc18783.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

### 기능적 요구사항 검증
 1. 고객은 커피를 주문한다.(O)
 2. 고객은 주문 후 결제 한다.(O)
 3. 결제가 완료 되면 커피를 만든다.(O)
 4. 결제가 취소되면 커피를 만들지 않는다.(O)
 5. 고객이 주문상태를 조회 할 수 있다.(O)
       
### 비기능 요구사항
 1. 결제가 완료 되지 않은 주문건은 커피를 만들지 않아야 한다.(O)
 2. 커피 만들 때 장애가 발생하더라도 주문은 계속 받아야 한다.(O)

# 구현
서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084 이다)

```bash
cd order
mvn spring-boot:run

cd payment
mvn spring-boot:run 

cd make
mvn spring-boot:run  

cd mypage
mvn spring-boot:run  
```

## DDD 의 적용

각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 make 마이크로 서비스). 
```

package starcoffee;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;
import java.util.Date;

@Entity
@Table(name="Make_table")
public class Make {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long orderId;
    private Long menuId;
    private String status;

    @PostPersist
    public void onPostPersist(){
        CoffeeMade coffeeMade = new CoffeeMade();
        BeanUtils.copyProperties(this, coffeeMade);
        coffeeMade.publishAfterCommit();

    }
    @PostUpdate
    public void onPostUpdate(){
        CoffeeCancled coffeeCancled = new CoffeeCancled();
        BeanUtils.copyProperties(this, coffeeCancled);
        coffeeCancled.publishAfterCommit();

    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }
    public Long getMenuId() {
        return menuId;
    }

    public void setMenuId(Long menuId) {
        this.menuId = menuId;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```

Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 
데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
![RestRepository ](https://user-images.githubusercontent.com/87048655/131713334-5bc80cb3-67db-4933-abb9-182e03b0becc.png)

- 적용 후 REST API 의 테스트
```
# order 서비스의 주문처리
http POST localhost:8081/orders orderId=1 price=1000 status="order in"

# payment 서비스의 결제처리
http POST localhost:8088/payments orderId=1 status="paying"

# make 서비스의 생산처리
http POST localhost:8088/makes orderId=1 status="making"

# 주문 상태 확인    
http http://localhost:8088/orders/1
HTTP/1.1 201 Created
Content-Type: application/json;charset=UTF-8
Date: Thu, 02 Sep 2021 00:27:35 GMT
Location: http://localhost:8081/orders/1
transfer-encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "orderId": 1,
    "price": 100,
    "status": "order in"
}

```

## 폴리글랏 퍼시스턴스
make MSA의 경우 H2 DB인 주문과 제작와 달리 Hsql으로 구현하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다. 

pom.xml 설정 <br>

![polyglot](https://user-images.githubusercontent.com/87048655/131714280-180bbafb-8b9d-4e0b-ba9e-6db8d6d2ef0c.png)

- 오더(order) 주문

![poliglot1](https://user-images.githubusercontent.com/87048655/131765372-cedd53bf-68fa-4fed-a905-b70642f9411f.png)

- 생산(make) 조회<br>
![poliglot2](https://user-images.githubusercontent.com/87048655/131765472-6fbf278d-d603-4397-93e4-1c55a3c28163.png)

## Gateway 적용

gateway > resources > applitcation.yml 설정 <br>
![gateway](https://user-images.githubusercontent.com/87048655/131714550-fe3f9561-a732-4587-8853-44af97422baf.png)


```bash
http POST localhost:8088/orders orderId=1 price=100 status="order in"
```

```bash
http localhost:8088/orders
HTTP/1.1 200 OK
Content-Type: application/hal+json;charset=UTF-8
Date: Thu, 02 Sep 2021 01:11:40 GMT
transfer-encoding: chunked

{
    "_embedded": {
        "orders": [
            {
                "_links": {
                    "order": {
                        "href": "http://localhost:8081/orders/1"
                    },
                    "self": {
                        "href": "http://localhost:8081/orders/1"
                    }
                },
                "orderId": 1,
                "price": 100,
                "status": "order in"
            }
        ]
    },
    "_links": {
        "profile": {
            "href": "http://localhost:8081/profile/orders"
        },
        "self": {
            "href": "http://localhost:8081/orders{?page,size,sort}",
            "templated": true
        }
    },
    "page": {
        "number": 0,
        "size": 20,
        "totalElements": 1,
        "totalPages": 1
    }
}
```



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(order) -> 결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
(app) order > external > PaymentService.java <br>

![feign client](https://user-images.githubusercontent.com/87048655/131715139-b165d549-4464-4630-8833-75ba31830c71.png)



- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리  <br>
![order-payment호출](https://user-images.githubusercontent.com/87048655/131720220-12bebf26-66c0-4703-b93d-b8461456c9c0.png)

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:

```bash
#결재(payment) 서비스를 잠시 내려놓음 (ctrl+c)

# 주문요청 (order)
http POST http://localhost:8081/orders orderId=1 price=1000 status="order start"
```
![동기식호출(에러)](https://user-images.githubusercontent.com/87048655/131766191-3323187b-0dd1-4397-8b1b-6e2daba023ec.png)

```bash
#결재(payment) 서비스 재기동
cd payment
mvn spring-boot:run

#주문요청 (order)
http POST http://localhost:8081/orders orderId=1 price=1000 status="order start"
```
![동기식호출(정상)](https://user-images.githubusercontent.com/87048655/131766370-b8fb1238-64a9-4621-96b2-223e08a21fb8.png)



## CQRS

CQRS 구현을 위해 고객의 예약 상황을 확인할 수 있는 Mypage를 구성. <br>

![cqrs](https://user-images.githubusercontent.com/87048655/131765858-c454f9de-c44c-4b9c-afde-05ba5f7dd2b9.png)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 


결제(payment)이 이루어진 후에 생산(make)으로 이를 알려주는 행위는 비 동기식으로 처리하여 생산(make)의 처리를 위하여 주문(order)이 블로킹 되지 않아도록 처리한다. <br>
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish) <br>
![비동기식_payment](https://user-images.githubusercontent.com/87048655/131721010-91ac60ac-feee-45f0-b688-2e16edad78a1.png)

- 생산 서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다 <br>
![make_handler](https://user-images.githubusercontent.com/87048655/131721373-2b28c28c-2254-4204-a0b2-4cbf9eee3486.png)
 
생산 시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 생산시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:(시간적 디커플링): <br>

```bash
#생산(make) 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders orderId=1 price=1000 status="order start"

```
![비동기식호출(정상)](https://user-images.githubusercontent.com/87048655/131766588-6abacc61-1551-4ad0-8423-e63add5b8cfb.png)

```bash
http GET http://localhost:8081/orders/1     # 'order start'상태
```
![비동기식호출(정상_mypage반영전)](https://user-images.githubusercontent.com/87048655/131766843-ef083536-98cb-4e6f-999a-a9038f6f4864.png)

```bash
#생산(make) 서비스 기동
cd make
mvn spring-boot:run

#주문상태 확인
http localhost:8084/mypages/4   # 'order start' 였던 상태값이 'making'로 변경된 것을 확인
```
![비동기식호출(정상_mypage반영후)](https://user-images.githubusercontent.com/87048655/131767068-ed3e7256-5944-41c1-9129-9302fe85f07f.png)

# 운영

## Deploy
- AWS IAM설정
``` bash
aws configure
AWS Access Key ID [None]: (IAM에서 생성했던 Acess Key) 
AWS Secret Access Key [None]: (IAM에서 생성했던 Secret Acess Key) 
Default region name [None]: (리전) 
Default output format [None]: 
```
- EKS Cluster생성 하기
``` bash
eksctl create cluster --name starcoffee1 --version 1.17 --nodegroup-name standard-workers --node-type t3.medium --nodes 2 --nodes-min 1 --nodes-max 4  
```
- ECR image repository 생성
``` bash
aws ecr create-repository --repository-name starcoffee-order --image-scanning-configuration scanOnPush=true --region ap-northeast-2
aws ecr create-repository --repository-name starcoffee-make --image-scanning-configuration scanOnPush=true --region ap-northeast-2
aws ecr create-repository --repository-name starcoffee-payment --image-scanning-configuration scanOnPush=true --region ap-northeast-2
aws ecr create-repository --repository-name starcoffee-mypage --image-scanning-configuration scanOnPush=true --region ap-northeast-2
aws ecr create-repository --repository-name starcoffee-gateway --image-scanning-configuration scanOnPush=true --region ap-northeast-2
```

-  AWS ECR Login 설정
``` bash
aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 980880891166.dkr.ecr.ap-northeast-2.amazonaws.com
```

배포 진행한다.
```
mvn package

- 이미지 생성
docker build -t  980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-order:v15 .
docker build -t  980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-make:1 .
docker build -t  980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-payment:v3 .
docker build -t  980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-mypage:v2 .
docker build -t  980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-gateway:v1 .

- 도커 푸시
docker push 980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-order:v15
docker push 980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-make:v1 
docker push 980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-payment:v3
docker push 980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-mypage:v2 
docker push 980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-gateway:v1 

- 컨테이너라이징
kubectl create deploy order --image=980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-order:v15
kubectl create deploy make --image=980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-make:v1
kubectl create deploy payment --image=980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-payment:v6
kubectl create deploy mypage --image=980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-mypage:v2
kubectl create deploy gateway --image=980880891166.dkr.ecr.ap-northeast-2.amazonaws.com/starcoffee-gateway:v1

- 서비스생성(Gateway는 LoadBalancer 로 생성)
kubectl expose deploy order --type=ClusterIP --port=8080
kubectl expose deploy make --type=ClusterIP --port=8080
kubectl expose deploy payment --type=ClusterIP --port=8080
kubectl expose deploy mypage --type=ClusterIP --port=8080
kubectl expose deploy gateway --type=LoadBalancer --port=8080

-생성 정보 확인하기
kubectl get all
```
![kubectl_get_all](https://user-images.githubusercontent.com/87048655/131770160-0a88b384-6a4b-49a2-8c2d-63ab93f7ac51.png)


## 동기식 호출 / 서킷 브레이킹 / 장애격리
* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함
Spring Spring FeignClient + Hystrix 옵션을 사용하여 테스팅 진행 신청(order) → 결제(payment) 시 연결을 REST API로 Response/Request로 구현되어 있으며, 과도한 신청으로 결제가 문제가 될 때 서킷브레이커로 장애격리

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
- order > application.yml

![HISTRIX](https://user-images.githubusercontent.com/87048655/131716947-76478178-f89a-4690-a6cb-e57a59fb2aed.png)

- 강제 부하설정<br>
![payment_부하처리](https://user-images.githubusercontent.com/87048655/131771061-8ccb63c2-ccff-4d46-bb2f-d674616276fb.png)

* siege
```
 siege 생성
 kubectl create deploy siege --image=ghcr.io/acmexii/siege-nginx:latest

 siege 들어가기:
 kubectl exec pod/siege-c54d6bdc7-8lc8f-it -- /bin/bash
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
 동시사용자 100명, 60초 동안 실시
```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://10.100.157.15:8080/orders POST {"orderId":1, "price":123, "status":"Order Start"}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 payment에서 처리되면서 다시 order 받기 시작

![siege오류발생중](https://user-images.githubusercontent.com/87048655/131718991-a9ce9aa2-9896-4ef3-9d04-ada9f87c2cb2.png)

- report<br>
![siege결과](https://user-images.githubusercontent.com/87048655/131719101-0c483b16-a6f9-493f-abce-9ef8d09aadee.png)


- 동시사용자 1명일 때는 100% 수용가능
```
siege -c1 -t60S -r10 -v --content-type "application/json" 'http://10.100.100.106:8080/orders POST {"orderId":1, "price":123, "status":"Order Start"}'
```
![circit_break_t1](https://user-images.githubusercontent.com/87048655/131770856-3a6b7370-7f57-44a8-9114-4b5fc8c277fc.png)
![circit_break_t1(rslt)](https://user-images.githubusercontent.com/87048655/131770686-9590f4c2-9c16-4137-98c8-4e7cf5c8b8c6.png)


## 오토스케일 아웃
- 오더(order) 요청 증가 시, reploca 를 동적으로 늘려줄 수 있도록 리소스 설정한다.
- order > kubernetes > deployment.yml<br>
![hpa_YML설정](https://user-images.githubusercontent.com/87048655/131775498-6dd21395-823c-424d-8b08-3eee16ce1da1.png)

- deploy 적용여부 확인
```bash
kubectl get deploy order -o yaml
```
- metrics 설치
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
```

- 오더(order)서비스에 대한 replica를 동적으로 늘려주도록 HPA를 설정한다. 설정은 CPU사용량이 1%를 넘어서면 replica를 3개까지 늘려준다.
```bash
kubectl autoscale deployment order --cpu-percent=5 --min=1 --max=3
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
 동시사용자 100명, 60초 동안 실시 * n번(CPU 사용량이 1% 초과하여 replica를 생성할 때 까지)
```
siege -c100 -t60S -r10 -v --content-type "application/json" 'http://10.100.157.15:8080/orders POST {"orderId":1, "price":123, "status":"Order Start"}'
```
- kubectl get pod, hpa 명령어로 확인
- kubectl get deploy order -w로 모니터링 진행



## 무정지 재배포
- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscale 이나 CB 설정을 제거함
- seige 로 배포작업 직전에 워크로드를 모니터링 함
```bash
kubectl apply -f kubernetes/deployment_readiness.yml
```
- readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패 <br>
![readness주석](https://user-images.githubusercontent.com/87048655/131774648-5e5b78a3-155d-4854-94fb-1073d425d026.png)

- deployment.yml에 readiness 옵션을 추가 <br>
![readness설정](https://user-images.githubusercontent.com/87048655/131774652-1ff0b360-ddaf-4dad-8111-c77a5d63debd.png)

- readiness적용된 deployment.yml 적용
```bash
kubectl apply -f kubernetes/deployment.yml
```
- 새로운 버전의 이미지로 교체
```bash
az acr build --registry skccteam03 --image skccteam03.azurecr.io/customercenter:v1 .
kubectl set image deploy customercenter customercenter=skccteam03.azurecr.io/customercenter:v1 -n coffee
```

- 기존 버전과 새 버전의 store pod 공존 중 <br>
![3](https://user-images.githubusercontent.com/26760226/106704049-bff58700-662e-11eb-8199-a20723c5245d.png)

- Availability: 100.00 % 확인 <br>
![4](https://user-images.githubusercontent.com/26760226/106704050-c08e1d80-662e-11eb-9214-9136748e1336.png)

## Config Map

### Service ClusterIP 확인
![image](https://user-images.githubusercontent.com/64818523/106609778-4c5d6680-65a9-11eb-8b31-8e11b3e22162.png)

### order ConfigMap 설정
  - order/src/main/resources/apllication.yml 설정

  * default쪽

![image](https://user-images.githubusercontent.com/64818523/106609096-8ed27380-65a8-11eb-88a2-e1b732e17869.png)

  * docker 쪽

![image](https://user-images.githubusercontent.com/64818523/106609301-c7724d00-65a8-11eb-87d3-d6f03c693db6.png)

- order/kubernetes/Deployment.yml 설정

![image](https://user-images.githubusercontent.com/64818523/106609409-dd800d80-65a8-11eb-8321-aa047e8a68aa.png)


### product ConfigMap 설정
  - product/src/main/resources/apllication.yml 설정

  * default쪽
  
![image](https://user-images.githubusercontent.com/64818523/106609502-f8eb1880-65a8-11eb-96ed-8eeb1fc9f87c.png)

  * docker 쪽
  
![image](https://user-images.githubusercontent.com/64818523/106609558-0bfde880-65a9-11eb-9b5a-240566adbad1.png)

- product/kubernetes/Deployment.yml 설정

![image](https://user-images.githubusercontent.com/64818523/106612752-c93e0f80-65ac-11eb-9509-9938f4ccf767.png)


### config map 생성 후 조회
```
kubectl create configmap apiorderurl --from-literal=url=http://10.0.54.30:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n coffee
```
![image](https://user-images.githubusercontent.com/64818523/106609630-1f10b880-65a9-11eb-9c1d-be9d65f03a1e.png)

```
kubectl create configmap apiproducturl --from-literal=url=http://10.0.164.216:8080 --from-literal=fluentd-server-ip=10.xxx.xxx.xxx -n coffee
```
![image](https://user-images.githubusercontent.com/64818523/106609694-3485e280-65a9-11eb-9b59-c0d4a2ba3aed.png)

- 설정한 url로 주문 호출
```
http POST localhost:8081/orders productName="Americano" qty=1
```

![image](https://user-images.githubusercontent.com/75309297/106707026-e10ca680-6633-11eb-83fa-3bcc907389ad.png)

- configmap 삭제 후 app 서비스 재시작
```
kubectl delete configmap apiorderurl -n coffee
kubectl delete configmap apiproducturl -n coffee

kubectl get pod/order-74c76b478-xx7n7 -n coffee -o yaml | kubectl replace --force -f-
kubectl get pod/product-66ddb989b8-r82sm -n coffee -o yaml | kubectl replace --force -f-
``` 

- configmap 삭제된 상태에서 주문 호출   
```
kubectl exec -it httpie -- /bin/bash
http POST http://10.0.101.221:8080/orders productName="Tea" qty=3
```
![image](https://user-images.githubusercontent.com/64818523/106706737-765b6b00-6633-11eb-9e73-48aa1190acdb.png)

- configmap 삭제된 상태에서 Pod와 deploy 상태 확인
```
kubectl get all -n coffee
```
![image](https://user-images.githubusercontent.com/64818523/106706899-b4588f00-6633-11eb-9670-169421b045ed.png)

- Pod와 상태 상세 확인
```
kubectl get pod order-74c76b478-mlpf4 -o yaml -n coffee
```
![image](https://user-images.githubusercontent.com/64818523/106706929-c33f4180-6633-11eb-843c-535c0b37904d.png)


## Self-healing (Liveness Probe)

![liveness주석](https://user-images.githubusercontent.com/87048655/131774897-42d3c10d-7112-4f0f-b201-00cd3f0d6548.png)

- deployment.yml 에 Liveness Probe 옵션 추가
```
cd ~/coffee/product/kubernetes
vi deployment.yml

(아래 설정 변경)
livenessProbe:
	tcpSocket:
	  port: 8081
	initialDelaySeconds: 5
	periodSeconds: 5
```
![liveness설정](https://user-images.githubusercontent.com/87048655/131774899-54c552f1-a759-4815-b5a4-505324bbea30.png)

- product pod에 liveness가 적용된 부분 확인
```
kubectl describe deploy product -n coffee
```
![image](https://user-images.githubusercontent.com/75309297/106708456-3f3a8900-6636-11eb-93af-07b754f2944a.png)

- product 서비스의 liveness가 발동되어 5번 retry 시도 한 부분 확인
```
kubectl get pod -n coffee
```

![image](https://user-images.githubusercontent.com/75309297/106707672-f7ffc880-6634-11eb-9b35-032348772306.png)
