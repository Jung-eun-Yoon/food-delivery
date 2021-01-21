# matching
선생님-학생 매칭 서비스

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 ](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#DDD-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출과-Fallback-처리)
    - [이벤트드리븐 아키텍쳐의 구현](#이벤트드리븐-아키텍쳐의-구현)
    - [Poliglot](#폴리글랏-퍼시스턴스)
    - [Gateway](#Gateway)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-/-서킷-브레이킹-/-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [Persistence Volume](#Persistence-Volume)
    - [Self_healing (liveness probe)](#Self_healing-(liveness-probe))
    - [무정지 재배포](#무정지-재배포)
  - [신규MSA추가_학생포인트관리시스템](#신규MSA추가_학생포인트관리시스템)
    - [신규서비스 분석설계](#신규서비스-분석설계)
    - [신규서비스 구현](#신규서비스-구현)
    - [운영과 Retirement](#운영과-Retirement)

# 서비스 시나리오

기능적 요구사항
1. 학생이 금액을 제시하여 매칭 요청을 한다. (선생님을 선택할 수 없다)
1. 학생이 결제한다.
1. 금액이 결제되면 방문요청내역이 목록에 조회된다.
1. 선생님은 방문요청을 조회한 후 선생님 이름과 만남시간을 입력하여 방문을 확정한다.
1. 방문요청이 접수 되면, 쿠폰이 발행된다
1. 학생이 매칭을 취소할 수 있다.
1. 매칭요청이 취소되면 방문이 취소된다.
1. 방문이 취소 되면 발행된 쿠폰이 취소 된다.
1. 학생은 myPage에서 매칭 상태를 중간중간 조회할 수 있다.
1. 매칭요청 화면에서 상태를 조회할 수 있다. 
1. 매칭요청/결제요청/방문확정/결제취소/방문취소 시 상태가 변경된다. 

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 매칭건은 아예 매칭이 성립되지 않아야 한다. Sync 호출
1. 장애격리
    1. 방문관리 기능이 수행되지 않더라도 매칭요청은 365/24 받을 수 있어야 한다. Async(event-driven) Eventual Consistency
1. 성능
    1. 학생이 매칭시스템에서 확인할 수 있는 상태를 마이페이지(프론트엔드)에서 확인할 수 있어야 한다 CQRS
    1. 상태가 바뀔때마다 myPage에서는 변경된 상태를 조회할 수 있어야 한다. Event driven
    1. 방문 상태가 변경될 때 마다 매칭관리에서 상태가 변경되어야 한다. corelation


# 체크포인트

...


# 분석/설계


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/oXrpW7GBVxVEQ4xDYSfbyK6tNEo1/every/1fcbffb626305265cb4134a6bd8f5216

## 이벤트 도출
### 1차 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/75401933/100964027-11b85d00-356b-11eb-97b0-abd00e78c2c6.png)

### 최종 이벤트스토밍 결과
![image](https://user-images.githubusercontent.com/75401933/105022842-8e58b980-5a8d-11eb-868c-aae24f8db3ed.png)

```
- 도메인 서열 분리
    - Core Domain:  match, visit  : 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 match의 경우 1주일 1회 미만, visit의 경우 1개월 1회 미만
    - Supporting Domain:  visitReqLists , myPages : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
    - General Domain:   payment(결제) : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음
```

## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/45473909/105029093-3c1b9680-5a95-11eb-812d-b4b634e5fcec.png)

  - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
  - 호출관계에서 PubSub 과 Req/Res 를 구분함
  - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 8084 이다)

```
cd match
mvn spring-boot:run

cd visit
mvn spring-boot:run  

cd payment
mvn spring-boot:run 

cd mypage
mvn spring-boot:run  
```


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 match 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하였고, 모든 구현에 있어서 영문으로 사용하여 별다른  오류없이 구현하였다.

```
package matching;

import javax.persistence.*;

import matching.external.Payment;
import matching.external.PaymentService;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Match_table")
public class Match {

    @Id
    private Long id;
    private Integer price;
    private String status;

    @PostPersist
    public void onPostPersist(){
        MatchRequested matchRequested = new MatchRequested();
        BeanUtils.copyProperties(this, matchRequested);
        matchRequested.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
        Payment payment = new Payment();
        // mappings goes here

        //변수 setting
        payment.setMatchId(Long.valueOf(this.getId()));
        payment.setPrice(Integer.valueOf(this.getPrice()));
        payment.setPaymentAction("Approved");

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest(payment);
    }

    @PreUpdate
    public void onPreUpdate(){
        if("cancel".equals(status)) {
            MatchCanceled matchCanceled = new MatchCanceled();
            BeanUtils.copyProperties(this, matchCanceled);
            matchCanceled.publishAfterCommit();
        }
    }

    public Long getId() {
        return id;
    }
    public void setId(Long id) {
        this.id = id;
    }

    public Integer getPrice() {
        return price;
    }
    public void setPrice(Integer price) {
        this.price = price;
    }

    public String getStatus() { return status; }
    public void setStatus(String status) {
        this.status = status;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package matching;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface MatchRepository extends PagingAndSortingRepository<Match, Long>{
}
```

- 적용 후 REST API 의 테스트
```
# match 서비스의 접수처리
http localhost:8088/matches id=5000 price=50000 status=matchRequest
```
![1 match에서명령어날림](https://user-images.githubusercontent.com/45473909/105010823-898d0900-5a7f-11eb-82b3-ab7163311364.PNG)
```
# match 서비스의 접수상태확인
http localhost:8088/matches/5000
```
![2 match테이블에쌓임](https://user-images.githubusercontent.com/45473909/105010828-8b56cc80-5a7f-11eb-8a8e-9a96984d6ab6.PNG)
```
# payment 서비스의 상태확인
http localhost:8088/payments/5000
```
![3 payment에서match에서날린데이터확인](https://user-images.githubusercontent.com/45473909/105011427-48e1bf80-5a80-11eb-9c95-e3d2e760e931.PNG)
```
# match 서비스에 대한 visit 응답
http POST localhost:8088/visits matchId=5000 teacher=TEACHER visitDate=21/01/21
```
![6 visit에서선생님방문계획작성](https://user-images.githubusercontent.com/45473909/105011436-4aab8300-5a80-11eb-8d3e-5fbe98a20668.PNG)



## 동기식 호출과 Fallback 처리

분석단계에서의 조건 중 하나로 접수(match)->결제(payment) 간의 호출은 동기식으로 호출하고자  동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient를 이용하여 호출하도록 한다.

- 결제서비스를 호출하기 위하여 FeignClient 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현

```
# (payment) PaymentService.java

package matching.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

@FeignClient(name="payment", url="${api.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.POST, path="/payments")
    public void paymentRequest(@RequestBody Payment payment);

}
```

- match접수를 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# match.java (Entity)
  @PostPersist
    public void onPostPersist(){
        MatchRequested matchRequested = new MatchRequested();
        BeanUtils.copyProperties(this, matchRequested);
        matchRequested.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
        Payment payment = new Payment();
        // mappings goes here

        //변수 setting
        payment.setMatchId(Long.valueOf(this.getId()));
        payment.setPrice(Integer.valueOf(this.getPrice()));
        payment.setPaymentAction("Approved");

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest(payment);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 접수도 못받는다는 것을 확인


```
# 결제 (payment) 서비스를 잠시 내려놓음 (ctrl+c)

# 접수처리
http localhost:8088/matches id=5005 price=50000 status=matchRequest   #Fail
```
![11 payment내리면match안됨](https://user-images.githubusercontent.com/45473909/105013488-a7a83880-5a82-11eb-9417-92d92668b879.PNG)
```

# payment서비스 재기동
cd payment
mvn spring-boot:run

#match 처리
http localhost:8088/matches id=5006 price=50000 status=matchRequest  #Success
```
![11 payment올리면match됨](https://user-images.githubusercontent.com/45473909/105013494-a8d96580-5a82-11eb-95de-73a47f072920.PNG)

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 이벤트드리븐 아키텍쳐의 구현

### 비동기식 호출 

결제가 완료 된 후에 방문(visit) 시스템으로 이를 알려주는 행위는 동기식이 아닌 비동기식으로 처리하며, 방문시스템의 처리를 위하여 매칭요청/결제가 블로킹 되지 않도록 처리한다.
 
- 이를 위하여 결제요청이력에 기록을 남긴 후에 곧바로 결제 완료 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
package matching;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Payment_table")
public class Payment {

    @Id
    private Long matchId;
    private Integer price;
    private String paymentAction;

    @PostPersist
    public void onPostPersist(){
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();


    }


```

- 방문 서비스에서는 결제완료 이벤트를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

```
package matching;

import matching.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{
    @Autowired VisitReqListRepository VisitReqListRepository;
    @Autowired VisitRepository VisitRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_(@Payload PaymentApproved paymentApproved){

        if(paymentApproved.isMe()){
            System.out.println("##### listener  : " + paymentApproved.toJson());


            //승인완료 시 승인완료된 리스트를 visitReqList에 받아서 보여줄 수 있도록 id setting

            VisitReqList visitReqList = new VisitReqList();

            visitReqList.setId(paymentApproved.getMatchId());
            VisitReqListRepository.save(visitReqList);

        }
    }

```


### 시간적 디커플링 / 장애격리 

방문(visit) 시스템은 결제(payment) 시스템과 완전히 분리되어있으며 이벤트 수신에 따라 처리되기 때문에, 방문 시스템이 유지보수로 인해 잠시 내려간 상태라도 방문요청(match) 및 결제(payment)하는데에 문제가 없다


- 방문 서비스(visit)를 잠시 놓은 후 매칭 요청 처리
```
# 매칭요청 처리
http POST http://localhost:8081/matches id=101 price=5000 status=matchRequest   #Success
```
![image](https://user-images.githubusercontent.com/75401910/105030156-bd275d80-5a96-11eb-87d0-7c16955c76ff.PNG)

- 결제서비스가 정상적으로 조회되었는지 확인
```
http http://localhost:8083/payments   #Success
```
![image](https://user-images.githubusercontent.com/75401933/105035459-5efe7880-5a9e-11eb-9e60-d824d2f1a4cc.png)

- 방문 서비스 다시 가동
```
cd visit
mvn spring-boot:run
```

- 가동 전/후의 방문상태 확인
```
# 신규 접수된 매칭요청건에 대해 선생님과 방문일자 매칭
http POST http://localhost:8082/visits matchId=101 teacher=Smith visitDate=20210101 
http localhost:8082/visits     
```
![image](https://user-images.githubusercontent.com/75401933/105036115-65412480-5a9f-11eb-8cf8-ea4e46376a46.png)


### SAGA / Corelation

방문(visit) 시스템에서 상태가 방문확정 또는 방문취소로 변경되면 매치(match) 시스템 원천데이터의 상태(status) 정보가 update된다.  

```
# mypage > PolicyHandler.java

  @StreamListener(KafkaProcessor.INPUT)
  public void wheneverVisitAssigned_(@Payload VisitAssigned visitAssigned){

      if(visitAssigned.isMe()){
          System.out.println("##### listener wheneverVisitAssigned  : " + visitAssigned.toJson());

          //방문 assign 이벤트를 수신하였을 때 해당 ID를 찾아서 상태값을 visitAssigned로 변경
          MatchRepository.findById(visitAssigned.getMatchId()).ifPresent(Match ->{
              System.out.println("##### wheneverVisitAssigned_MatchRepository.findById : exist" );

              Match.setStatus(visitAssigned.getEventType()); //상태값은 모두 이벤트타입으로 셋팅함
              MatchRepository.save(Match);
          });

      }
  }


  @StreamListener(KafkaProcessor.INPUT)
  public void wheneverVisitCanceled_(@Payload VisitCanceled visitCanceled) {

      if (visitCanceled.isMe()) {
          System.out.println("##### listener  : " + visitCanceled.toJson());

          //방문취소 이벤트를 수신하였을 때 해당 ID를 찾아서 상태값을 visitCanceled로 변경
          MatchRepository.findById(visitCanceled.getMatchId()).ifPresent(Match -> {
              System.out.println("##### wheneverVisitCanceled_MatchRepository.findById : exist");
              Match.setStatus(visitCanceled.getEventType());
              MatchRepository.save(Match);
          });
      }
  }

```


### CQRS

매칭 상태가 변경될 때 마다 mypage에서 event를 수신하여 mypage의 매칭상태를 조회하도록 view를 구현하였다.   

```
# mypage > PolicyHandler.java

@StreamListener(KafkaProcessor.INPUT)
public void wheneverVisitCanceled_(@Payload VisitCanceled visitCanceled){

    if(visitCanceled.isMe()){
        System.out.println("##### listener  : " + visitCanceled.toJson());

        MyPageRepository.findById(visitCanceled.getMatchId()).ifPresent(MyPage ->{
            System.out.println("##### wheneverVisitCanceled_MyPageRepository.findById : exist" );
            MyPage.setStatus(visitCanceled.getEventType());
            MyPageRepository.save(MyPage);
        });
    }
}
@StreamListener(KafkaProcessor.INPUT)
public void wheneverVisitAssigned_(@Payload VisitAssigned visitAssigned){

    if(visitAssigned.isMe()){
        System.out.println("##### listener wheneverVisitAssigned  : " + visitAssigned.toJson());

        MyPageRepository.findById(visitAssigned.getMatchId()).ifPresent(MyPage ->{
            System.out.println("##### wheneverVisitAssigned_MyPageRepository.findById : exist" );

            MyPage.setStatus(visitAssigned.getEventType()); //상태값은 모두 이벤트타입으로 셋팅함
            MyPage.setTeacher(visitAssigned.getTeacher());
            MyPage.setVisitDate(visitAssigned.getVisitDate());
            MyPageRepository.save(MyPage);
        });

    }
}
@StreamListener(KafkaProcessor.INPUT)
public void wheneverPaymentApproved_(@Payload PaymentApproved paymentApproved){

    if(paymentApproved.isMe()){
        System.out.println("##### listener  : " + paymentApproved.toJson());

        MyPage mypage = new MyPage();
        mypage.setId(paymentApproved.getMatchId());
        mypage.setPrice(paymentApproved.getPrice());
        mypage.setStatus(paymentApproved.getEventType());
        MyPageRepository.save(mypage);
    }
}
@StreamListener(KafkaProcessor.INPUT)
public void wheneverPaymentCanceled_(@Payload PaymentCanceled paymentCanceled){

    if(paymentCanceled.isMe()){
        System.out.println("##### listener  : " + paymentCanceled.toJson());


        MyPageRepository.findById(paymentCanceled.getMatchId()).ifPresent(MyPage ->{
            System.out.println("##### wheneverPaymentCanceled_MyPageRepository.findById : exist" );

            MyPage.setStatus(paymentCanceled.getEventType()); //상태값은 모두 이벤트타입으로 셋팅함
            MyPageRepository.save(MyPage);
        });
    }
}
@StreamListener(KafkaProcessor.INPUT)
public void wheneverMatchCanceled_(@Payload MatchCanceled matchCanceled){

    if(matchCanceled.isMe()){
        System.out.println("##### listener  : " + matchCanceled.toJson());

        MyPageRepository.findById(matchCanceled.getId()).ifPresent(MyPage ->{
            System.out.println("##### wheneverMatchCanceled_MyPageRepository.findById : exist" );

            MyPage.setStatus(matchCanceled.getEventType()); //상태값은 모두 이벤트타입으로 셋팅함
            MyPageRepository.save(MyPage);
        });

    }
}
    
```
- mypage의 view로 조회

![image](https://user-images.githubusercontent.com/75401933/105024191-21462380-5a8f-11eb-8abc-b169dd9d8c3a.png)


## 폴리글랏 퍼시스턴스

match 는 다른 서비스와 구별을 위해 별도 hsqldb를 사용 하였다. 이를 위해 match내 pom.xml에 dependency를 h2database에서 hsqldb로 변경 하였다.

```
#match의 pom.xml dependency를 수정하여 DB변경

  <!--
  <dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
  </dependency>
  -->

  <dependency>
    <groupId>org.hsqldb</groupId>
    <artifactId>hsqldb</artifactId>
    <version>2.4.0</version>
    <scope>runtime</scope>
  </dependency>

```


## Gateway

```
server:
  port: 8088

---

spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: match
          uri: http://localhost:8081
          predicates:
            - Path=/matches/**
        - id: visit
          uri: http://localhost:8082
          predicates:
            - Path=/visits/**,/visitReqLists/**
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/payments/**
        - id: mypage
          uri: http://localhost:8084
          predicates:
            - Path=/myPages/**,/myPages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true

---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: match
          uri: http://match:8080
          predicates:
            - Path=/matches/**
        - id: visit
          uri: http://visit:8080
          predicates:
            - Path=/visits/**,/visitReqLists/**
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/payments/**
        - id: mypage
          uri: http://mypage:8080
          predicates:
            - Path=/myPages/**,/myPages/**
```

- Gateway 서비스 실행 상태에서 8088과 8081로 각각 서비스 실행하였을 때 동일하게 match 서비스 실행되었다.
```
http localhost:8088/matches id=50 price=50000 status=matchRequest
```
![8088포트](https://user-images.githubusercontent.com/45473909/105039570-0f22b000-5aa4-11eb-9090-45662dcd79d0.PNG)

```
http localhost:8081/matches id=51 price=50000 status=matchRequest
```
![8081포트](https://user-images.githubusercontent.com/45473909/105039551-0a5dfc00-5aa4-11eb-86c0-c3fc63d5b0f6.PNG)



# 운영

## CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 deployment.yml, service.yml 에 포함되었다

<img width="678" alt="01 CI설정" src="https://user-images.githubusercontent.com/66051393/105039224-a4717480-5aa3-11eb-9d28-99fa5a809750.png">

<img width="886" alt="02 CD설정" src="https://user-images.githubusercontent.com/66051393/105039282-b521ea80-5aa3-11eb-94c3-1ec50475300d.png">

<img width="647" alt="03 CD설정_상세" src="https://user-images.githubusercontent.com/66051393/105039330-c5d26080-5aa3-11eb-8b05-cabb28c6eaf1.png">



## 동기식 호출 / 서킷 브레이킹 / 장애격리


서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함
시나리오는 매칭요청(match)-->결제(payment) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.


1. Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 600 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
- application.yml

```
#application.yml
...
      bindings:
        event-in:
          group: match
          destination: matching
          contentType: application/json
        event-out:
          destination: matching
          contentType: application/json

api:
  payment:
    url: http://payment:8080

hystrix:
  command:
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610
```

2. 피호출 서비스(결제:payment) 의 임의 부하 처리 - 400 밀리에서 증감 300 밀리 정도 왔다갔다 하게
- (payment) 결제이력.java (Entity)
```
#Payment.java
...
    @PostPersist
    public void onPostPersist(){
        PaymentApproved paymentApproved = new PaymentApproved();
        BeanUtils.copyProperties(this, paymentApproved);

        //바로 이벤트를 보내버리면 주문정보가 커밋되기도 전에 배송발송됨 이벤트가 발송되어 주문테이블의 상태가 바뀌지 않을 수 있다.
        // TX 리스너는 커밋이 완료된 후에 이벤트를 발생하도록 만들어준다.
        TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
            @Override
            public void beforeCommit(boolean readOnly) {
                paymentApproved.publish();
            }
        });
        try {
            //istio test를 위해 payment sleep 추가가
           Thread.currentThread().sleep((long) (400 + Math.random() * 300));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
```

3. 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
 - 동시사용자 100명
 - 60초 동안 실시
```
 - siege -c100 -t60S -r5 -v --content-type "application/json" 'http://match:8080/matches POST {"id": 600, "price":1000, "status": "matchRequest"}' 
```
<img width="635" alt="03 화면증적" src="https://user-images.githubusercontent.com/66051393/105108628-ffd05080-5afd-11eb-826d-66a7b252c09a.png">

서킷브레이크가 발생하지 않아 아래와 같이 여러 조건으로 부하테스트를 진행하였으나, 500 에러를 발견할 수 없었음

```
 - siege -c255 -t1M -r5 -v --content-type "application/json" 'http://match:8080/matches POST {"id": 600, "price":1000, "status": "matchRequest"}' 
 
 - siege -c255 -t2M -r5 -v --content-type "application/json" 'http://match:8080/matches POST {"id": 600, "price":1000, "status": "matchRequest"}' 
```


## 오토스케일 아웃

앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다.

visit 구현체에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 10프로를 넘어서면 replica 를 10개까지 늘려준다:

kubectl autoscale deploy visit --min=1 --max=10 --cpu-percent=15

<img width="504" alt="01 화면증적" src="https://user-images.githubusercontent.com/66051393/105040263-f8308d80-5aa4-11eb-9686-0afedeaa5a48.png">


kubectl exec -it pod siege -- /bin/bash
siege -c20 -t120S -v http://visit:8080/visits/600

부하에 따라 visit pod의 cpu 사용률이 증가했고, Pod Replica 수가 증가하는 것을 확인할 수 있었음

<img width="536" alt="02 화면증적" src="https://user-images.githubusercontent.com/66051393/105040477-3cbc2900-5aa5-11eb-94b8-7f2eb33102fa.png">


## Persistence Volume

visit 컨테이너를 마이크로서비스로 배포하면서 영속성 있는 저장장치(Persistent Volume)를 적용함

• PVC 설정 확인

kubectl describe pvc azure-pvc

<img width="546" alt="01-1 화면증적(decribe)" src="https://user-images.githubusercontent.com/66051393/105042326-73933e80-5aa7-11eb-8c4f-94b46c811e56.png">

• PVC Volume설정 확인
mypage 구현체에서 해당 pvc를 volumeMount 하여 사용 (kubectl get deployment mypage -o yaml)

<img width="583" alt="02 화면증적" src="https://user-images.githubusercontent.com/66051393/105042760-f87e5800-5aa7-11eb-9447-2ecb7d427623.png">

• mypage pod에 접속하여 mount 용량 확인

<img width="482" alt="03 mount_설정확인" src="https://user-images.githubusercontent.com/66051393/105042971-41361100-5aa8-11eb-8fa7-65efbe12fb8c.png">


## Self_healing (liveness probe)
mypage구현체의 deployment.yaml 소스 서비스포트를 8080이 아닌 고의로 8081로 변경하여 재배포한 후 pod 상태 확인

• 정상 서비스포트 확인

<img width="557" alt="01 증적자료" src="https://user-images.githubusercontent.com/66051393/105043345-c4effd80-5aa8-11eb-83db-df351905d102.png">

• 비정상 상태의 pod 정보 확인

<img width="581" alt="03 증적자료_POD비정상으로재기동" src="https://user-images.githubusercontent.com/66051393/105043596-0ed8e380-5aa9-11eb-9c46-dabe5736df9c.png">


## 무정지 재배포

먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함
seige 로 배포작업 직전에 워크로드를 모니터링 함.

```
siege -c10 -t30S -r10 --content-type "application/json" 'http://match:8080/matches POST {"id": "101"}'

```
1. CI/CD를 통해 새로운 배포 시작
1. seige 의 화면으로 넘어가서 Availability 가 100% 미만으로 떨어졌는지 확인
![image](https://user-images.githubusercontent.com/75401933/105041017-d7b50300-5aa5-11eb-90dd-5031cd846d81.png)

1. 배포기간중 Availability 가 평소 100%에서 80% 대로 떨어지는 것을 확인. 원인은 쿠버네티스가 성급하게 새로 올려진 서비스를 READY 상태로 인식하여 서비스 유입을 진행한 것이기 때문. 이를 막기위해 Readiness Probe 를 설정함:
1. CI/CD를 통해 새로운 배포 시작
1. 동일한 시나리오로 재배포 한 후 Availability 확인:
![image](https://user-images.githubusercontent.com/75401933/105041119-f4e9d180-5aa5-11eb-9afb-e7af9c06fcce.png)

배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.









# 신규MSA추가_학생포인트관리시스템



## 신규서비스 분석설계
선생님 방문을 요청하는 학생고객 관리를 위하여 학생 포인트 관리 기능을 추가하기로 하였다. 
추가된 기능적 요구사항은 다음과 같다.
1. 각 시스템에 학생정보 attribute 추가 관리한다.
1. 방문이 assign 되었을 때 학생의 포인트가 100만큼 증가한다. (신규학생의 경우 학생정보를 신규 생성한 후 100포인트를 생성한다.)
1. 방문이 취소되었을 경우 학생의 포인트를 100 차감한다.



### 변경된 이벤트스토밍
![변경된이벤트스토밍](https://user-images.githubusercontent.com/75401933/105131136-b26ad800-5b2b-11eb-9e2a-1918de08addc.png)



### 헥사고날아키텍쳐의 변화
![변경된헥사고날아키텍쳐](https://user-images.githubusercontent.com/75401933/105131271-f4941980-5b2b-11eb-847c-37adbf21ad81.png)
신규추가된 학생 포인트 관리시스템은 정책구독만 하며 event를 발행하지는 않는다.



## 신규서비스 구현
기존 마이크로서비스에 수정을 발생시키지 않도록 req/res 방식이 아닌 event를 subscribe 하는 방식으로 구현하였다. 기존 운영중인 마이크로 서비스에는 student attribute를 신규추가한 것 외에 변경사항은 없으며, 기존 MSA의 아키텍처나 DB구조에 영향을 주지 않는다. 



## 운영과 Retirement
req/res 방식으로 구현하지 않았기 때문에 서비스가 불필요해져도 Deployment에서 제거되면 기존 MSA에 어떤 영향도 주지 않는다.
(결제(payment) MSA의 경우 match-payment간 req/res 방식으로 구현되어 있기 때문에 API 변화 또는 Retirement 시 매치(match) MSA의 변경을 초래한다)

```
# Payment API 변화 시 Match.java의 수정

    @PostPersist
    public void onPostPersist(){

        Payment payment = new Payment();
        //변수 setting
        payment.setMatchId(Long.valueOf(this.getId()));
        payment.setPrice(Integer.valueOf(this.getPrice()));
        payment.setStudent(String.valueOf(this.getStudent()));
        payment.setPaymentAction("Approved");

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest(payment);
                
                --> 

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest2(payment);
    }
```

```
# Payment Retirement 시 Match.java의 수정

    @PostPersist
    public void onPostPersist(){
        /**
        Payment payment = new Payment();
        //변수 setting
        payment.setMatchId(Long.valueOf(this.getId()));
        payment.setPrice(Integer.valueOf(this.getPrice()));
        payment.setStudent(String.valueOf(this.getStudent()));
        payment.setPaymentAction("Approved");

        MatchApplication.applicationContext.getBean(PaymentService.class)
                .paymentRequest(payment);
        **/
    }
```
