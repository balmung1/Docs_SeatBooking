# 개인실습 - 룸예약시스템

회의실 예약현황관리, 예약, 취소 기능 제공

# Table of contents

회의실예약시스템
  - [서비스 시나리오](#서비스-시나리오)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)

# 서비스 시나리오

기능적 요구사항
1. 관리자는 RoomSystem에서 회의실을 등록/삭제한다.
1. 회의실이 등록/삭제 될때 알람을 보낸다.
1. 사용자는 회의실을 예약/취소 할 수 있다.
1. 사용자가 회의실을 예약/취소 할때 알람을 보낸다.
1. 사용자는 예약상태를 조회할 수 있다.
1. 사용자가 회의실을 예약취소 할 경우 패널티 점수가 부여된다.

비기능적 요구사항
1. 트랜잭션
    1. 사용자가 회의실 예약을 취소할 경우 실시간 벌점 부과. Sync 호출 
1. 장애격리
    1. mettingRoom과 reservation이 분리되어 회의실 등록/삭제 서비스가 수행되지 않더라도 사용자의 회의실 예약은 항시 서비스 되어야 한다. Async (event-driven), Eventual Consistency
1. 성능
    1. 사용자가 회의실 등록/예약현황 바로 알 수있도록 알림 서비스가 제공 되어야 한다. Event driven
    1. 사용자가 회의실 등록/예약현황을 확인할 수 있는 lookUp 서비스가 제공 되어야 한다. CQRS


# 분석/설계

## Event Storming 결과
![eventstorming](https://user-images.githubusercontent.com/63028480/92455138-48950600-f1fc-11ea-8df4-4a87e0ec5985.JPG)

    - 도메인 서열 분리 
        - Core Domain:  회의실(MeetingRoom), 예약(Reservation) 도메인 
        - Supporting Domain: 예약현황확인(LookUp) 도메인 CQRS, 벌점(Demerit) 도메인
        - General Domain:  알림(Notice) 도메인


### 기능적/비기능적 요구사항을 커버하는지 검증

![eventstorming2](https://user-images.githubusercontent.com/63028480/92455155-4d59ba00-f1fc-11ea-9254-84a792614b92.JPG)

    - 관리자가 회의실을 등록/삭제 한다. (ok)
    - 관리자가 회의실을 등록/삭제 하면 알림을 보낸다. (ok)

![eventstorming3](https://user-images.githubusercontent.com/63028480/92455175-521e6e00-f1fc-11ea-9cf0-0b1f14288033.JPG)

    - 사용자가 회의실을 예약하거나 예약을 취소 한다 (ok)
    - 예약 및 취소가 완료되면 알림을 보낸다. (ok)
    
### 비기능 요구사항에 대한 검증

![eventstorming4](https://user-images.githubusercontent.com/63028480/92455194-577bb880-f1fc-11ea-9286-fe5616f3e71c.JPG)

    - 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
    - 예약 취소시 벌점부여: ACID 트랜잭션 적용. 예약 취소 시 벌점부여에 대해서는 Request-Response 방식 처리
    - 나머지 모든 inter-microservice 트랜잭션: 예약상태, 회의실상태 등 모든 이벤트에 대해 데이터 일관성의 시점이 크리티컬하지 않기 때문에, Eventual Consistency 를 기본으로 채택함.


## 헥사고날 아키텍처 다이어그램 도출
    
![핵사고날](https://user-images.githubusercontent.com/63028480/92455209-5a76a900-f1fc-11ea-8974-c4f45e83071c.JPG)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트와 파이선으로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

좌석예약시스템은 5개의 마이크로서비스로 구현되어 있다.

1. 게이트웨이: https://github.com/balmung1/Gateway.git
1. 회의실 시스템: https://github.com/balmung1/MeetingRoom.git
1. 예약 시스템: https://github.com/balmung1/Reservation.git
1. 알림 시스템: https://github.com/balmung1/Notice.git
1. 벌점 시스템: https://github.com/balmung1/Demerit.git
1. 모든현황 시스템: https://github.com/balmung1/Lookup.git

cd meetingroom   
mvn spring-boot:run

cd reservation  
mvn spring-boot:run

cd notice   
mvn spring-boot:run

cd demerit
mvn spring-boot:run

cd lookup   
mvn spring-boot:run


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 pay 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다.

```
package RoomSystem;

import javax.persistence.*;

import RoomSystem.external.Demerit;
import RoomSystem.external.DemeritService;
import org.springframework.beans.BeanUtils;

import java.util.List;

@Entity
@Table(name = "Reservation_table")
public class Reservation {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private Long reservationId;
    private Long roomId;
    private String userId;
    private String roomStatus;
    private String mType;


    @PostPersist
    public void onPostUpdate() {
        if(mType.equals("reservation")){
            Reserved reserved = new Reserved();
            BeanUtils.copyProperties(this,reserved);
            reserved.publish();

        }else{
            ReservationCanceled reservationCanceled = new ReservationCanceled();
            BeanUtils.copyProperties(this,reservationCanceled);
            reservationCanceled.publish();

            Demerit demerit = new Demerit();

            demerit.setDemeritId(reservationCanceled.getId());
            demerit.setUserId(reservationCanceled.getUserId());

            DemeritService demeritService = Application.applicationContext.getBean(DemeritService.class);
            demeritService.demerit(demerit);
        }

    }


    public String getmType() {
        return mType;
    }

    public void setmType(String mType) {
        this.mType = mType;
    }

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public Long getReservationId() {
        return reservationId;
    }

    public void setReservationId(Long reservationId) {
        this.reservationId = reservationId;
    }

    public Long getRoomId() {
        return roomId;
    }

    public void setRoomId(Long roomId) {
        this.roomId = roomId;
    }

    public String getUserId() {
        return userId;
    }

    public void setUserId(String userId) {
        this.userId = userId;
    }

    public String getRoomStatus() {
        return roomStatus;
    }

    public void setRoomStatus(String roomStatus) {
        this.roomStatus = roomStatus;
    }

}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package RoomSystem;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface ReservationRepository extends PagingAndSortingRepository<Reservation, Long>{

}
```
- 적용 후 REST API 의 테스트
```
@FeignClient(name="Demerit", url="http://Demerit:8080")
public interface DemeritService {

    @RequestMapping(method= RequestMethod.POST, path="/demerits/save")
    public void demerit(@RequestBody Demerit demerit);

}

# 회의실 등록(관리자)
http http://meetingroom:8080/meetingRooms roomId=1 roomName=A101 location=F1

# 회의실 예약(사용자)
http patch http://reservation:8080/reservations/reservation reservationId=1 roomId=1 userId=ABC

# 회의실 예약 취소(사용자)
http patch http://reservation:8080/reservations/cancel reservationId=1 roomId=1 userId=ABC

# 예약 현황 확인(사용자)
http http://reservation:8080/reservations
```


## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 예약취소->벌점부여 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 벌점서비스를 호출하기 위하여 FeignClient를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (app) external.java
package RoomSystem.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

import java.util.Date;

/**
 * Created by uengine on 2018. 11. 21..
 */
@FeignClient(name="Demerit", url="http://Demerit:8080")
public interface DemeritService {

    @RequestMapping(method= RequestMethod.POST, path="/demerits/save")
    public void demerit(@RequestBody Demerit demerit);

}
```

- 예약취소 직후(@PostPersist) 벌점을 요청하도록 처리
```
# Reservation.java (Entity)
    @PostPersist
    public void onPostUpdate() {
        if(mType.equals("reservation")){
            Reserved reserved = new Reserved();
            BeanUtils.copyProperties(this,reserved);
            reserved.publish();

        }else{
            ReservationCanceled reservationCanceled = new ReservationCanceled();
            BeanUtils.copyProperties(this,reservationCanceled);
            reservationCanceled.publish();

            Demerit demerit = new Demerit();

            demerit.setDemeritId(reservationCanceled.getId());
            demerit.setUserId(reservationCanceled.getUserId());

            DemeritService demeritService = Application.applicationContext.getBean(DemeritService.class);
            demeritService.demerit(demerit);
        }

    }
```


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


회의실 등록 후 이를 알려주는 행위는 비 동기식으로 처리하여 예약/취소 시스템이 블로킹 되지 않아도록 처리한다.
 
- 이를 위하여 이벤트를를 카프카로 송출한다(Publish)
 
```
# MeetingRoom.java
// package RoomSystem;

    @PostPersist
    public void onPostPersist(){
        RoomRegistered roomRegistered = new RoomRegistered();
        BeanUtils.copyProperties(this, roomRegistered);
        roomRegistered.publish();
    }
```
- 회의실 예약 서비스에서는 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다

```
@Service
public class PolicyHandler {

    @Autowired
    ReservationRepository reservationRepository;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRoomRegistered_RoomList(@Payload RoomRegistered roomRegistered){

        if(roomRegistered.isMe()){
            Reservation reservation = new Reservation();

            reservation.setRoomId(roomRegistered.getRoomId());
            reservation.setRoomStatus("True");
            reservation.setReservationId(roomRegistered.getId());

            reservationRepository.save(reservation);
            System.out.println("##### Room Registered RoomList : " + roomRegistered.toJson());
        }
    }
    
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRoomDelete_RoomList(@Payload RoomDelete roomDelete){

        if(roomDelete.isMe()){
            Reservation reservation = new Reservation();

            reservation.setId(roomDelete.getId());

            reservationRepository.delete(reservation);

            System.out.println("##### Room Delete RoomList : " + roomDelete.toJson());
        }
    }
}

```
알림 시스템은 실제로 알람을 보낼 수 없으므로, 예약/취소 및 회의실등록/삭제 이벤트에 대해서 System.out.println 처리 하였다.
  
```
package RoomSystem;

@Service
public class PolicyHandler{
    
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRoomRegistered_Notice(@Payload RoomRegistered roomRegistered){

        if(roomRegistered.isMe()){
            System.out.println("##### RoomRegistered Notice : " + roomRegistered.toJson());
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverRoomDelete_Notice(@Payload RoomDelete roomDelete){

        if(roomDelete.isMe()){
            System.out.println("##### RoomDelete Notice : " + roomDelete.toJson());
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReserved_Notice(@Payload Reserved reserved){

        if(reserved.isMe()){
            System.out.println("##### Reserved Notice : " + reserved.toJson());
        }
    }
    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverReservationCanceled_Notice(@Payload ReservationCanceled reservationCanceled){

        if(reservationCanceled.isMe()){
            System.out.println("##### ReservationCanceled Notice : " + reservationCanceled.toJson());
        }
    }

}

```

Lookup(CQRS) 시스템은 예약/취소와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, Lookup 시스템이 유지보수로 인해 잠시 내려간 상태라도 예약/취소를 하는데 문제가 없다
```

# API게이트웨이
![gatway설정](https://user-images.githubusercontent.com/63028480/92490855-7bef8900-f22c-11ea-861f-54a8ed34fff9.JPG)
![핵사고날](https://user-images.githubusercontent.com/63028480/92455209-5a76a900-f1fc-11ea-8974-c4f45e83071c.JPG)

# 운영
## DevOps 등록
![pipelines](https://user-images.githubusercontent.com/63028480/92490875-81e56a00-f22c-11ea-804c-becafe2d7d04.JPG)

## kubectl pod 확인
![cubectl](https://user-images.githubusercontent.com/63028480/92490895-890c7800-f22c-11ea-852e-a781a5ac0974.JPG)

## CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 GCP를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 cloudbuild.yml 에 포함되었다.


## 동기식 호출 / 서킷 브레이킹 / 장애격리

## 오토스케일 아웃

## 무정지 재배포

- 모든 프로젝트의 readiness probe 및 liveness probe 설정 완료.

readinessProbe:
  httpGet:
    path: /actuator/health
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 10
livenessProbe:
  httpGet:
     path: /actuator/health
     port: 8080
  initialDelaySeconds: 120
  timeoutSeconds: 2
  periodSeconds: 5
  failureThreshold: 5


