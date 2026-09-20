# <img width="50" height="50" alt="logo" src="https://github.com/user-attachments/assets/c146908d-e222-49fe-aac6-66c1bb3b74be" /> SpendOlive (스펜돌리브) - 개인 자산 및 지출 관리 서비스







> **팀 프로젝트 (Team Project)** | 개발 기간: 2026.06 ~ 2026.08

> **담당:** 풀스택 (아키텍처 공동 설계, 회원/결제 코어 시스템, 관리자 백오피스, 보안)


---



## 1. 프로젝트 개요 (Project Overview)

- **서비스 소개:** 사용자의 '고정 지출' 관리에 초점을 맞추어, 구독(OTT 등)의 그룹 공유부터 자동 결제 및 정산까지 안전하게 지원하는 퍼스널 지출 관리 서비스입니다.

- **개발 배경:** 구독 경제 시대에 파악하기 어려워진 정기 결제를 효율적으로 관리하는 플랫폼을 구상했습니다. 그중에서도 수동 송금의 번거로움과 금전적 트러블이 가장 잦은 'OTT 공유'에 착안했습니다. 단순한 유저 매칭을 넘어, 시스템을 통한 확실한 결제 및 정산 코어(에스크로)를 직접 구현함으로써 누구나 안심할 수 있는 스마트한 지출 관리의 기반을 구축하고자 기획했습니다.


---



## 2. 사용 기술 및 환경 (Tech Stack)

| 분야 | 기술 스택 |

| :--- | :--- |

| **Backend** | Java 21, Spring Boot, Spring Security, Spring JDBC(JdbcTemplate), JWT, Lombok |

| **Frontend** | JavaScript, JSP, JSTL, CSS |

| **Database & Cache** | Oracle 11g, Redis |

| **Tools/DevOps** | Apache Tomcat, Maven, Git, GitHub |

| **External API** | Kakao (Login/Share), Toss Payments, 금융결제원, Solapi, SMTP |

---



## 3. 담당 기능 (My Responsibilities)



- **아키텍처 설계 및 테크 리드 협업**

  - 테크 리드와 함께 프로젝트 전체 아키텍처 설계 및 기술적 조율 담당

- **핵심 비즈니스 및 결제 시스템 구축**

  - 금융결제원 및 Toss Payments API를 활용한 결제 및 정산(Settlement) 시스템 구현

  - 사용자의 자산 관리(Asset Management) 시스템 구축

- **관리자용 백오피스 기능 구현**

  - 회원 관리, 위반 신고 관리 및 자동/수동 경고 처리 시스템 구현

- **프론트엔드 설계 및 UI/UX 최적화**

  - 비동기 통신(AJAX/Fetch 등) 연동 모듈 설계 및 구현을 통한 코드 재사용성 및 유지보수성 향상

  - 공통 UI 컴포넌트(버튼 등)의 CSS 모듈화를 통한 디자인 일관성 확보 및 개발 효율성 향상

  - 간편 모드 구현(사용자 맞춤형 글꼴 크기 및 폰트 설정 기능)을 통한 접근성 개선
    
- **보안 및 데이터 보호**

  - Spring Security와 JWT를 활용한 애플리케이션 전체의 보안 대책 및 권한 관리



---



### 4. 트러블슈팅 (Troubleshooting)



> **주요 기술적 과제 해결 (Core Problem Solving)**
<br>



### Issue 1: 인메모리 락과 DB 비관적 락(Atomic Query)을 조합한 중복 결제 및 오버부킹 방지


**1. 문제 상황 (Problem)**

- 결제 과정에서 2가지 형태의 동시성(Concurrency) 문제가 발생할 위험이 존재했습니다.

- **[단일 유저]** 결제 로딩 중 새로고침(F5)을 누르거나 결제 버튼을 연속으로 클릭(따닥)할 경우, 동일한 POST 요청이 중복 전송되어 DB에 이중 결제되는 문제.

- **[다수 유저]** 남은 자리가 1개인 방에 다수의 유저가 동시에 결제를 시도할 경우, 트랜잭션 경합(Race Condition)으로 인해 정원을 초과해버리는 오버부킹(데이터 꼬임) 문제.



**2. 원인 분석 (Cause)**

- 애플리케이션(Java) 코드 측에서 SELECT를 이용해 남은 자리를 확인하고 INSERT를 실행하기까지의 과정 사이에 다른 트랜잭션이 끼어들 수 있는 틈(Gap)이 존재했던 것이 근본 원인이었습니다.



**3. 해결책 검토 과정 (Approach)**

- 복잡한 분산 락(Redisson 등)을 도입하면 시스템 오버헤드가 증가할 것이라 판단했습니다.

- 따라서 단일 유저의 중복 요청은 애플리케이션 레벨(인메모리)에서 빠르게 쳐내고, 다수 유저의 오버부킹은 Redis의 아토믹 연산(decrement)을 활용해 락(Lock) 없이 빠르고 안전하게 동시성을 제어하는 하이브리드 방식을 채택했습니다


**4. 적용한 해결책 (Solution)**

- **단일 유저 제어 (Application Level Lock):** ConcurrentHashMap을 활용해 현재 처리 중인 결제 키를 인메모리로 관리하여 중복 접근을 차단했습니다.

- **다수 유저 제어 (Redis Atomic Counter):** Redis에 방의 남은 자리(Seats)를 저장하고, 결제 요청이 올 때마다 decrement()를 사용해 원자적으로 자리를 차감합니다. 결과가 0 미만이 될 경우, 즉시 increment()로 롤백하고 예외를 발생시켜 물리적으로 정원 초과를 방지했습니다.

```java

// 1. 단일 유저의 중복 결제(따닥) 차단 (Java 인메모리 락)

String processingKey = createProcessingKey(userId, roomId);

if (!processingPayments.add(processingKey)) {

    throw new PaymentProcessException("PAYMENT_PROCESSING", "이미 결제를 처리 중입니다.");

}



try {

  // 2. 다수 유저의 오버부킹 완전 차단 (Redis Atomic Counter 활용)

    String redisKey = "room:" + roomId + ":seats";


// 최초 접근 시, 남은 자리를 초기화 (TTL 1시간)

    if (Boolean.TRUE.equals(redisTemplate.opsForValue().setIfAbsent(

        redisKey, String.valueOf(Math.max(limit - currentMembers, 0))))) {

            redisTemplate.expire(redisKey, 1, TimeUnit.HOURS);

    }



    // 아토믹 연산(decrement)을 통한 남은 자리 차감 처리

    Long remainingSeats = redisTemplate.opsForValue().decrement(redisKey);

    if (remainingSeats == null || remainingSeats < 0) {

        // 枠がないため、マイナスした1を元に戻して(increment)例外処理

        redisTemplate.opsForValue().increment(redisKey);

        throw new PaymentProcessException("ROOM_FULL", "すでに定員に達した部屋です。");

    }



    try {

// 결제 및 방 입장 로직 실행 (생략)

    } catch (Exception e) {

// 결제 실패 시, 차감했던 자리를 다시 복구(롤백)

        redisTemplate.opsForValue().increment(redisKey);

        throw e;

    }

} finally {

  // 3. 모든 처리 종료 후 인메모리 락 해제

    processingPayments.remove(processingKey);

}

-- 2. 다수 유저의 오버부킹 완전 차단 (DB 조건부 INSERT 쿼리)

INSERT INTO ott_room_member_tb (room_id, member_login_id, status, ...)

SELECT ?, ?, 'ACTIVE', ...

FROM ott_room_tb

WHERE room_id = ?

  AND (SELECT COUNT(*) 

       FROM ott_room_member_tb 

       WHERE room_id = ? AND status = 'ACTIVE') < member_limit;

```

**5. 성과 및 배운 점 (Result)**

- 단일 유저의 중복 클릭 및 다수 유저의 오버부킹 문제를 100% 차단하여 결제 데이터의 무결성을 확보했습니다.

- 무거운 배타 락을 걸지 않고도, Redis의 싱글 스레드 특성(아토믹 연산)을 정확히 이해하고 적용함으로써 높은 성능과 데이터 정합성을 모두 만족하는 아키텍처를 설계했습니다.

<br>



### Issue 2: 외부 API와 내부 DB 간의 트랜잭션 불일치 해결 및 보상 트랜잭션 구현


**1. 문제 상황 (Problem)**

- Toss Payments(외부 결제 API) 승인에는 성공하여 고객 계좌에서 출금되었음에도, 이후 자사 서버 DB에 결제 내역이나 에스크로 정보를 저장(INSERT/UPDATE)하는 과정에서 예외가 발생할 위험이 존재했습니다.

- 이 경우, 고객은 돈을 지불했는데 서비스 내에서는 '미결제' 상태로 남게 되는 치명적인 데이터 불일치(Data Inconsistency)가 발생합니다.



**2. 원인 분석 (Cause)**

- 외부 API 호출(Network I/O)과 내부 DB 트랜잭션은 본질적으로 분리되어 있습니다.

- Spring의 @Transactional을 적용해도 DB 롤백만 수행될 뿐, 이미 완료된 외부 API(Toss)의 결제 승인은 자동으로 취소되지 않는다는 분산 트랜잭션의 한계가 근본 원인이었습니다.



**3. 해결책 검토 과정 (Approach)**

- 완벽한 분산 트랜잭션 제어를 위해 2PC(Two-Phase Commit) 방식을 고려했으나, 외부 결제 서비스와 강결합할 수 없는 구조적 한계가 있었습니다.

- 이에 애플리케이션 레벨에서 보상 트랜잭션(Compensating Transaction) 패턴을 직접 구현하여, DB 저장 실패 시 능동적으로 외부 결제를 취소하는 'Saga 패턴'의 기본 개념을 적용하기로 결정했습니다.



**4. 적용한 해결책 (Solution)**

- DB INSERT/UPDATE 로직을 try-catch 블록으로 감싸고, catch 발생 시 즉시 Toss 결제 취소 API(cancelApprovedPayment)를 호출하도록 설계했습니다.

- 취소 성공/실패 여부에 따라 명확한 예외 메시지를 던져 프론트엔드 및 사용자에게 정확한 상황을 인지시키도록 처리했습니다.

```java

try {
    // 1. 내부 DB 트랜잭션 실행 (결제 상태 업데이트, 에스크로 정보 저장 등)
    paymentRepository.updatePaymentStatus(paymentInfo);
    paymentRepository.insertEscrow(escrowInfo);
    // ... 기타 관련 데이터 저장 처리
    
} catch (Exception databaseException) {
    // 2. DB 저장 실패 시: 이미 승인된 Toss 결제를 즉시 취소 (보상 트랜잭션 실행)
    boolean cancelled = cancelApprovedPayment(paymentKey);

    String message = cancelled
            ? "결제 정보 저장에 실패하여 Toss 승인을 자동으로 취소했습니다."
            : "결제 정보 저장 및 Toss 승인 취소에 실패했습니다. 관리자의 확인이 필요합니다.";

    throw new PaymentProcessException(
            "PAYMENT_SAVE_FAILED", 
            message, 
            databaseException);
}

```

**5.성과 및 배운 점 (Result)**

- 결제 시스템에서 가장 치명적인 문제인 '고객의 금전적 피해(팬텀 결제)'를 원천 차단하고 결제 데이터의 정합성을 100% 보장했습니다.

- 외부 서비스와 내부 시스템 간의 에러 전파(Error Propagation) 과정을 이해하고, 안전한 페일세이프(Fail-safe) 메커니즘을 직접 설계하는 아키텍처 설계 역량을 길렀습니다.



<br>



### Issue 3: 결제/정산 코어의 라이프사이클(State Machine) 설계 및 환불 처리의 데이터 정합성 확보



**1.문제 상황 (Problem)**

- OTT 쉐어 서비스의 자동 결제/정산을 스케줄러(배치 처리)로 운영하던 중, 2가지 중대한 비즈니스 로직 결함(Edge Case)이 발견되었습니다.

- **[과잉 청구 리스크]** 'OTT 시작일 10일 전'을 자동 결제일로 지정한 탓에, 방이 꽉 차서 서비스가 시작되기 '전'에 자동 결제일이 도래하여 첫 달에 유저에게 이중 결제(Double Billing)가 발생할 위험이 있었습니다.

- **[환불로 인한 정산 풀 오염]** 유저가 중간에 이탈할 경우 플랫폼 손실로 직결되므로, 단순 환불 처리를 해버리면 에스크로(보관금) 및 플랫폼 수익 데이터와 불일치가 일어나 호스트에게 잘못된 금액이 송금될 위험이 있었습니다.



**2. 원인 분석 (Cause)**

- 결제와 정산 플로우가 '날짜(Date)' 기반의 단순 스케줄러에 의존하고 있었으며, 각 데이터 행의 '상태(Status)'에 따른 엄격한 제어(State Machine)가 부재했던 것이 근본 원인이었습니다.


**3. 해결책 검토 과정 (Approach)**

- 스케줄러 내부에 복잡한 if-else 날짜 판별 로직을 넣는 것은 기술 부채가 될 것이라 판단했습니다.

- 대신, 상태 전이 모델(State Machine)을 도입하여 결제/정산의 라이프사이클을 명확히 정의하고, 환불 처리는 시스템 자동화가 아닌 '관리자 승인 기반 트랜잭션'으로 분리하는 안전한 설계를 선택했습니다.



**4. 적용한 해결책 (Solution)**

- **[첫 달 과잉 청구 방지 - FIRST 상태 도입]** 꽉 찬 방의 상태를 일시적으로 FIRST로 설정. 스케줄러가 자동 결제 대상을 추출할 때 FIRST 상태의 방은 건너뛰게 함으로써 첫 달 이중 결제를 완벽히 차단. 이후 안전한 타이밍에 ACTIVE로 전이시켰습니다.

- **[정산 라이프사이클 확립 및 다단계 환불 처리]** 정산 상태를 YET(대기) -> READY(송금 준비) -> DONE(완료)로 엄격하게 전이시켰습니다. 환불은 관리자가 상황을 판단한 후 실행하며, Toss 결제 취소 API를 호출함과 동시에 결제 내역(payment_tb), 에스크로 보관금(escrow_payout_tb), 플랫폼 수익(revenue_tb)의 모든 관련 상태를 단일 트랜잭션 내에서 REFUNDED로 일괄 업데이트하여 정산 풀에서 완전히 격리시켰습니다.
```java

// 관리자에 의한 환불 트랜잭션 (상태의 완전 격리)
@Transactional(rollbackFor = Exception.class)
public void executeRoomRefund(SettlementPaymentVO payment) throws Exception {
    // 1. 외부 결제 API (Toss) 승인 취소 실행
    if(!cancelApprovedPayment(payment.getPaymentKey())) {
        throw new PaymentProcessException("Toss 결제 취소에 실패했습니다.");
    }
    
    // 2. 내부 DB 다단계 상태 업데이트 (REFUNDED로 변경하여 정산 대상에서 제외)
    paymentRepository.updatePaymentstatusRefund(payment.getPayment_id()); // 결제 상태 업데이트
    
    SettlementRefundVO refund = new SettlementRefundVO();
    refund.setPayment_id(payment.getPayment_id());
    refund.setRefund_status("COMPLETED");
    // ... (기타 환불 메타데이터 세팅)
    
    paymentRepository.insertRefund(refund); // 환불 내역 분리 저장
}
```

**5.성과 및 배운 점 (Result)**

- FIRST 상태 도입을 통해 복잡한 날짜 계산 없이 과잉 청구(이중 결제) 버그를 100% 해결했습니다.

- 에스크로 기반의 복잡한 자금 이동 환경에서 환불이나 중도 이탈 등 엣지 케이스가 발생해도 데이터 모순이 없는, 견고한 코어 정산 시스템(State Machine)을 설계하는 도메인 모델링 능력을 갖췄습니다.



---



## 5. 관련 링크 (Links)

- **GitHub Repository:** [Link](https://github.com/LeeChungMoo965/group-spendolive/tree/master/src/main)

- **화면설계 Figma :**[Link](https://www.figma.com/design/jXBp0uN1p2c65oKGgrZjmq/%EB%B0%B1%EC%97%94%EB%93%9C-%EC%B5%9C%EC%A2%85-%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-UI?node-id=0-1&t=va79lufa0lhW4jKj-1)

## 6. 환경 구축 (Getting Started)

### 필수 요건 (Prerequisites)
- Java 21
- Spring Boot (WAR Packaging)
- Apache Maven 3.8.x 이상 (또는 IDE 내장 Maven)
- Oracle Database 11g
- Redis

### 환경 변수 설정 (Environment Variables)
보안을 위해 DB 비밀번호 및 각종 외부 API 키가 포함된 `application.properties` 파일은 리포지토리 추적에서 제외되었습니다.
로컬 환경에서 실행할 때는 `src/main/resources/` 경로에 `application.properties`를 직접 생성하고, 리포지토리에 함께 업로드된 `application-template.properties` 파일을 참고하여 환경에 맞게 값을 설정해 주세요.

**[연동이 필요한 외부 API]**
- Google SMTP (이메일 인증용)
- Kakao Developers (소셜 로그인용)
- 금융결제원 OpenBanking API (계좌 연동용)
- Toss Payments API (결제 처리용)
- Solapi (SMS 발송 처리용)

### 실행 순서 (How to Run)
1. 본 리포지토리를 클론합니다.
   `git clone https://github.com/LeeChungMoo965/group-spendolive.git`
2. IDE(IntelliJ IDEA, Eclipse 등)를 열고, 프로젝트를 Maven 프로젝트로 임포트합니다.
3. 로컬 환경의 Oracle DB 및 Redis 서버를 실행합니다.
4. 프로젝트 내에 포함된 DDL 스크립트를 실행하여 데이터베이스 테이블을 생성합니다.
5. `application.properties`의 설정(DB 접속 정보 및 각종 API 키)을 완료합니다.
6. `SpendoliveApplication.java`를 실행합니다. (※ 본 프로젝트는 프론트엔드 뷰 템플릿으로 JSP를 사용하고 있으므로 WAR 패키징 방식으로 동작합니다.)


## ERD (Entity Relationship Diagram)

<img width="500" height="240" alt="Relational_1" src="https://github.com/user-attachments/assets/a03ccd1a-473a-465a-992c-94973c2b2ebc" />
