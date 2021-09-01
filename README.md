# starcoffee


![MSA_CoffeChain_logo_3](https://user-images.githubusercontent.com/26760226/106547009-b6e1b880-654f-11eb-8c4b-4526f2200c72.jpg)

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
    - [Deploy / Pipeline](#Deploy--Pipeline)
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


- Actor, Command 부착



- Policy 부착



- Aggregate 부착



- View 추가 및 Bounded Context 묶기


- 완성 모형: Pub/Sub, Req/Res 추가(점선은 Pub/Sub, 실선은 Req/Resp)


### 헥사고날 아키텍처 다이어그램 도출 (Polyglot)

![헥사고날](https://user-images.githubusercontent.com/87048655/131712015-85a0258e-9212-4dde-a906-e53b3dc18783.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐

### 기능적 요구사항 검증

![image](https://user-images.githubusercontent.com/75309297/106581014-bcf28c00-6585-11eb-867d-df5c2fe91896.png)

    - 고객이 커피를 주문한다. (OK)
    - 커피가 만들어진다. (OK)
    - 커피가 만들어 지면 재고가 Stock에 전달된다. (OK)
    
![image](https://user-images.githubusercontent.com/75309297/106581647-6d609000-6586-11eb-88ce-cf81b4681b47.png)

    - 고객이 커피를 주문한다. (OK)
    - 커피가 만들어진다. (OK)
    - 커피가 생산이 완료되면 고객이 order에서 조회 할 수 있다.(OK)
      
![image](https://user-images.githubusercontent.com/75309297/106582664-96cdeb80-6587-11eb-8a21-d7f7aba5492d.png)

    - 고객이 주문을 취소할 수 있다.(OK)
    - 주문이 취소되면 커피생산을 취소한다.(OK)
    - 고객이 order에서 조회 할 수 있다. (OK)

![image](https://user-images.githubusercontent.com/75309297/106582947-e7454900-6587-11eb-8819-d65f48ae10bd.png)

    - 고객이 MyPage에서 커피주문 내역을 볼 수 있어야 한다.(OK)
       
### 비기능 요구사항

    - 생산취소가 되지않은 건에 대해서는 절대 주문취소를 하지 않는다. (점주가 손해보고는 못 사는 성격)
    - 재고변경(차감)이 되지않은 건에 대해서는 재고변경 프로세스가 진행되면 안된다.
    - 재료창고의 크기를 감안하여 모든 메뉴의 재고는 100개를 시작으로 차감된다.  

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
http POST localhost:8081/orders orderId=1 price=1000 status="order start"

# payment 서비스의 결제처리
http POST localhost:8088/payments orderId=1 status="paying"

# make 서비스의 생산처리
http localhost:8088/makes orderId=1 status="making"

# 주문 상태 확인    
http localhost:8081/orders/1
HTTP/1.1 200
Content-Type: application/hal+json;charset=UTF-8
Date: Thu, 19 Aug 2021 02:05:39 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/1"
        },
        "self": {
            "href": "http://localhost:8081/orders/1"
        }
    },
    "orderId": 0,
    "price": 1000,
    "status": "order start"
}

```

## 폴리글랏 퍼시스턴스
make MSA의 경우 H2 DB인 주문과 제작와 달리 Hsql으로 구현하여 MSA간 서로 다른 종류의 DB간에도 문제 없이 동작하여 다형성을 만족하는지 확인하였다. 
pom.xml 설정
![polyglot](https://user-images.githubusercontent.com/87048655/131714280-180bbafb-8b9d-4e0b-ba9e-6db8d6d2ef0c.png)

## Gateway 적용

gateway > resources > applitcation.yml 설정
![gateway](https://user-images.githubusercontent.com/87048655/131714550-fe3f9561-a732-4587-8853-44af97422baf.png)

gateway 테스트

```bash
http POST localhost:8080/orders orderId=2 price=2000 status="order"
```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(order) -> 결제(payment) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
(app) order > external > PaymentService.java

![feign client](https://user-images.githubusercontent.com/87048655/131715139-b165d549-4464-4630-8833-75ba31830c71.png)



- 주문 취소 시 제고 변경을 먼저 요청하도록 처리
```java
// (app) Order.java (Entity)

    @PreUpdate
    public void onPreUpdate(){

       msacoffeechainsample.external.Product product = new msacoffeechainsample.external.Product();
       product.setId(orderCanceled.getProductId());
       product.setOrderId(orderCanceled.getId());
       product.setProductName(orderCanceled.getProductName());
       product.setStatus(orderCanceled.getStatus());
       product.setQty(orderCanceled.getQty());
        
       // req/res
       OrderApplication.applicationContext.getBean(msacoffeechainsample.external.ProductService.class)
                    .cancel(product.getId(), product);
    }
```


- 동기식 호출이 적용되서 제품 서비스에 장애가 나면 주문 서비스도 못받는다는 것을 확인:

```bash
#제품(payment) 서비스를 잠시 내려놓음 (ctrl+c)

#주문취소 (order)
http PATCH http://localhost:8081/orders/1 status="Canceled"    #Fail
```


```bash
#제품(product) 서비스 재기동
cd product
mvn spring-boot:run

#주문취소 (order)
http PATCH http://localhost:8081/orders/2 status="Canceled"    #Success
```


## CQRS

CQRS 구현을 위해 고객의 예약 상황을 확인할 수 있는 Mypage를 구성.

## 비동기식 호출 / 시간적 디커플링 / 장애격리 


결제(payment)이 이루어진 후에 생산(order)로 이를 알려주는 행위는 비 동기식으로 처리하여 제품(product)의 처리를 위하여 주문이 블로킹 되지 않아도록 처리한다.
 
- 주문이 되었다(Ordered)는 도메인 이벤트를 카프카로 송출한다(Publish)
 
![10_비동기 호출(주문_제조)](https://user-images.githubusercontent.com/77084784/106619371-e0343000-65b3-11eb-9599-ca40b275751b.jpg)

- 제품(product)에서는 주문(ordered) 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 주문접수(Order)는 송출된 주문완료(ordered) 정보를 제품(product)의 Repository에 저장한다.:
 
![11_비동기 호출(주문_제조)](https://user-images.githubusercontent.com/77084784/106619501-01951c00-65b4-11eb-88e9-8870bad805f7.jpg)

제품(product) 시스템은 주문(order)/제고(stock)와 완전히 분리되어있으며(sync transaction 없음), 이벤트 수신에 따라 처리되기 때문에, 제품(product)이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다.(시간적 디커플링):
```bash
#제품(product) 서비스를 잠시 내려놓음 (ctrl+c)

#주문하기(order)
http http://localhost:8081/orders item="Americano" qty=1  #Success

#주문상태 확인
http GET http://localhost:8081/orders/1    # 상태값이 'Completed'이 아닌 'Requested'에서 멈춤을 확인
```
![12_time분리_1](https://user-images.githubusercontent.com/77084784/106619595-196ca000-65b4-11eb-892e-a0ad2fa1b7f0.jpg)
```bash
#제품(product) 서비스 기동
cd product
mvn spring-boot:run

#주문상태 확인
http GET http://localhost:8081/orders/1     # 'Requested' 였던 상태값이 'Completed'로 변경된 것을 확인
```
![12_time분리_2](https://user-images.githubusercontent.com/77084784/106619700-330de780-65b4-11eb-818e-70152aba4400.jpg)

# 운영

## Deploy / Pipeline
- 네임스페이스 만들기
```bash
kubectl create ns coffee
kubectl get ns
```
![kubectl_create_ns](https://user-images.githubusercontent.com/26760226/106624530-1922d380-65b9-11eb-916a-5b6956a013ad.png)

- 폴더 만들기, 해당 폴더로 이동
``` bash
mkdir coffee
cd coffee
```
![mkdir_coffee](https://user-images.githubusercontent.com/26760226/106623326-d7ddf400-65b7-11eb-92af-7b8eacb4eeb3.png)

- 소스 가져오기
``` bash
git clone https://github.com/MSACoffeeChain/main.git
```
![git_clone](https://user-images.githubusercontent.com/26760226/106623315-d6143080-65b7-11eb-8bf0-b7604d2dd2db.png)

- 빌드 하기
``` bash
cd order
mvn package
```
![mvn_package](https://user-images.githubusercontent.com/26760226/106623329-d7ddf400-65b7-11eb-8d1b-55ec35dfb01e.png)

- 도커라이징 : Azure 레지스트리에 도커 이미지 푸시하기
```bash
az acr build --registry skccteam03 --image skccteam03.azurecr.io/order:latest .
```
![az_acr_build](https://user-images.githubusercontent.com/26760226/106706352-e9181680-6632-11eb-8f22-0fbf80a9a575.png)

- 컨테이너라이징 : 디플로이 생성 확인
```bash
kubectl apply -f kubernetes/deployment.yml
kubectl get all -n coffee
```
![kubectl_apply](https://user-images.githubusercontent.com/26760226/106624114-a7e32080-65b8-11eb-965b-b1323c52d58e.png)

- 컨테이너라이징 : 서비스 생성 확인
```bash
kubectl expose deploy order --type="ClusterIP" --port=8080 -n coffee
kubectl get all -n coffee
```
![kubectl_expose](https://user-images.githubusercontent.com/26760226/106623324-d7455d80-65b7-11eb-809c-165bfa828bbe.png)

## 동기식 호출 / 서킷 브레이킹 / 장애격리
* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 생산(product)-->재고(stock) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 재고 사용 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# application.yml
feign:
  hystrix:
    enabled: true
    
hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

* siege 툴 사용법:
```
 siege가 생성되어 있지 않으면:
 kubectl run siege --image=apexacme/siege-nginx -n coffee
 siege 들어가기:
 kubectl exec -it pod/siege-5c7c46b788-4rn4r -c siege -n coffee -- /bin/bash
 siege 종료:
 Ctrl + C -> exit
```
* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c10 -t60S -r10 -v --content-type "application/json" 'http://10.0.209.210:8080/products POST {"orderId":1, "status":"Requested", "productName":"Ame", "qty":1}'
```
- 부하 발생하여 CB가 발동하여 요청 실패처리하였고, 밀린 부하가 stock에서 처리되면서 다시 product를 받기 시작

![image](https://user-images.githubusercontent.com/6468351/106703226-31ccd100-662d-11eb-9463-a10bb211cd70.png)

- report

![image](https://user-images.githubusercontent.com/6468351/106702534-da7a3100-662b-11eb-99f8-b54962eff735.png)

- CB 잘 적용됨을 확인

## 오토스케일 아웃

## 무정지 재배포
- 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscale 이나 CB 설정을 제거함
- seige 로 배포작업 직전에 워크로드를 모니터링 함
```bash
kubectl apply -f kubernetes/deployment_readiness.yml
```
- readiness 옵션이 없는 경우 배포 중 서비스 요청처리 실패 <br>
![1](https://user-images.githubusercontent.com/26760226/106704039-bec45a00-662e-11eb-9a26-dc5d0c403d03.png)

- deployment.yml에 readiness 옵션을 추가 <br>
![2](https://user-images.githubusercontent.com/26760226/106704044-bff58700-662e-11eb-8842-4d1bbbead1ef.png)

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
![image](https://user-images.githubusercontent.com/75309297/106708030-8f651b80-6635-11eb-979a-bee010a28e86.png)

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
