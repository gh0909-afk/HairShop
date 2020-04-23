# HairShop

![image](https://user-images.githubusercontent.com/63028469/79933589-71a23d80-848b-11ea-9a14-c8cdac44a89c.png)


# 개인과제 - 헤어샵 예약 시스템

- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [개인과제 - 헤어샵 예약 시스템](#개인과제 - 헤어샵 예약 시스템)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  
# 서비스 시나리오

HairShop 예약 system

기능적 요구사항
1. 고객이 원하는 시간을 선택해서 예약한다.
1. 예약하면 헤어샵으로 예약 내역이 전달된다.
1. 헤어샵에서 예약을 확인하고 스타일리스가 시간이 배정된다.
1. 고객이 예약을 취소할 수 있다.
1. 예약이 취소되면 시간이 배정된 스타일리스트가 일정이 취소된다.
1. 예약상태가 변경될 때 마다 문자로 알림을 보낸다.
1. 배정된 예약자 현황을 확인 할 수 있다.


비기능적 요구사항
1. 트랜잭션
    1. 예약된 시간에 스타일리스트가 없을 경우 예약이 되지 않아야 한다. Sync 호출
1. 장애격리
    1. 예약시간 관리 기능이 수행되지 않아도 예약은 365일 24시간 받을 수 있어야한다  Async (event-driven), Eventual Consistency
    1. 예약시스템이 과중되면 사용자를 잠시동안 받지 않고 예약을 잠시후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 예약상태가 바뀔때마다 문자로 알림을 줄 수 있어야 한다.


# 체크포인트

- 분석 설계


  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

## TO-BE 조직 (Vertically-Aligned)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  하단 그림 참조

### 조직 및 요구사항 도출 도출
![image](https://user-images.githubusercontent.com/63028469/79945074-074bc600-84a8-11ea-8ee8-b0641097ef43.png)


### 이벤트도출, 액터 커맨드 부착, 어그리게잇, 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/63028469/79934430-7b2ca500-848d-11ea-8d6a-85730096b78a.png)

    - 도메인 서열 분리 
        - Core Domain:  예약(front), 헤어샵관리 -> 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 예약 시스템의 경우 1주일 1회 미만, 헤어샵관리의 경우 1개월 1회 미만
        - Supporting Domain:     예약자현황 View -> 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 1주일 1회 이상을 기준으로 함.
        - General Domain:   고객관리 -> 문자발송 서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 (핑크색으로 이후 전환할 예정)

### 폴리시 부착과 컨텍스트 매핑

![image](https://user-images.githubusercontent.com/63028469/79939797-f183d400-849a-11ea-8c9a-55ddff693df0.png)


### 완성된 1차 모형

![image](https://user-images.githubusercontent.com/63028469/79939921-46bfe580-849b-11ea-8664-7f6a45d639d2.png)


    - View Model 추가


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
    - 고객이 원하는 시간을 선택해서 예약한다 (ok)
    - 예약하면 헤어샵으로 예약 내역이 전달된다 (ok - event driven)
    - 헤어샵에서 예약을 확인하고 스타일리스트가 시간이 배정된다 (ok)
    - 시간이 비어있는 스타일리스트가 선택된다 (No -sync)
    - 고객이 예약을 취소할 수 있다 (ok)
    - 예약이 취소되면 시간이 배정된 스타일리스트가 일정이 취소된다 (ok)
    - 예약자 현황을 조회한다 (view)

### Sync 방식을 추가 및 스타일리스트 관리 요구사항이 적용된 완성된 2차 모형 (설계 내용 수정 최종본)

![image](https://user-images.githubusercontent.com/63028469/80064209-7db4fa80-8572-11ea-8f1c-686e0f13bd48.png)


    - 예약시 스타일리스트 배정처리 : 서비스는 예약하려는 시간에 맞는 스타일리스트가 배정이 되어야 진행되기 때문에 예약시 스타일리스트 배정에 대해서는  Request-Response 방식 처리한다.
    - 예약 기능은 서비스 제공의 측면이 강하며, 한 번 등록 시 여러명의 고객들이 예약을 하기 때문에 예약(Front)에 대해 헤어샵관리 서비스는 Async (event-driven), Eventual Consistency 방식으로 처리한다.
    - 예약 시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시후에 하도록 유도한다 Circuit breaker를 사용하여 
    - 헤어샵에 예약된 고객의 명단을 헤어샵관리시스템(프론트엔드)에서 확인할 수 있어야 한다 CQRS
    - inter-microservice 트랜잭션: 모든 이벤트에 대해 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.    


## 헥사고날 아키텍처 다이어그램 도출
    
![image](https://user-images.githubusercontent.com/63028469/79945868-b937c200-84a9-11ea-8098-8875eb7dc303.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
cd HRSystem
mvn spring-boot:run

cd Management
mvn spring-boot:run 

cd Reservation
mvn spring-boot:run  
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: (예시는 Reservation마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하였다. 모델링 시에 영문화 완료하였기 때문에 그대로 개발하는데 큰 지장이 없었다.

```
package HairShop;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="ReservationSystem_table")
public class ReservationSystem {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Integer reserveId;
    private String userName;
    private String time;

    @PostPersist
    public void onPostPersist(){
        Reserved reserved = new Reserved();
        BeanUtils.copyProperties(this, reserved);
        reserved.publish();


    }

    @PostRemove
    public void onPostRemove(){
        ReservationCanceled reservationCanceled = new ReservationCanceled();
        BeanUtils.copyProperties(this, reservationCanceled);
        reservationCanceled.publish();


    }


    public Integer getReserveId() {
        return reserveId;
    }

    public void setReserveId(Integer reserveId) {
        this.reserveId = reserveId;
    }
    public String getUserName() {
        return userName;
    }

    public void setUserName(String userName) {
        this.userName = userName;
    }
    public String getTime() {
        return time;
    }

    public void setTime(String time) {
        this.time = time;
    }
}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package HairShop;

import org.springframework.data.repository.CrudRepository;
public interface ReservationSystemRepository extends CrudRepository<ReservationSystem, Integer> {


}
```

## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 예약관리시스템(hairShopManagementSystems)-> 디자니어 정보(hrSystems) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 디자이너 정보를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
# (hairShopManagementSystems) HrSystemService.java

@Service
@FeignClient(name ="hrSystems", url="http://52.231.118.11:8080")
public interface HrSystemService {

    @RequestMapping(method = RequestMethod.POST, value = "/hrSystems", consumes = "application/json")
    void selectStylist(HrSystem hrSystem);

}
```

- 예약시간관리 데이터 생성 직후(@PostPersist) 담당 디자이너를 선택하도록 처리
```
#HairShopManagementSystem.java (Entity)

 @PostPersist
    public void onPostPersist(){
        StylistConfirmed stylistConfirmed = new StylistConfirmed();
        BeanUtils.copyProperties(this, stylistConfirmed);
        stylistConfirmed.publish();

        HrSystem hrSystem = new HrSystem();
        hrSystem.setStylistName(this.stylist);
        // mappings goes here

        //스타일리스트 정보 확인
        HrSystemService hrSystemService = Application.applicationContext.getBean(HrSystemService.class);
        HrSystemService.selectStylist(hrSystem);

    }
```

# 운영

## CI/CD 설정


각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 Azure를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하에 azure-pipeline.yml 에 포함되었다.

- devops를 활용하여 pipeline을 구성하였고, CI CD 자동화를 구현하였다.
![image](https://user-images.githubusercontent.com/63028469/80063502-0337ab00-8571-11ea-9c9a-1a2ca6de172f.png)

- 아래와 같이 pod 가 정상적으로 올라간 것을 확인하였다.
![image](https://user-images.githubusercontent.com/63028469/80063656-53af0880-8571-11ea-9484-76f4ff6dd745.png)


- 아래와 같이 쿠버네티스에 모두 서비스로 등록된 것을 확인할 수 있다.
![image](https://user-images.githubusercontent.com/63028469/80063704-72ad9a80-8571-11ea-873d-3bdbef527d64.png)





