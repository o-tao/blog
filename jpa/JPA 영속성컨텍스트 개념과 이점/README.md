# 영속성 컨텍스트
### 영속성 컨텍스트란?
- 엔티티를 영구적으로 저장하는 환경, JPA를 이해하는데 가장 중요한 용어입니다.
- EntityManager.persist(entity) → DB에 저장하는 게 아닌 엔티티를 영속성 컨텍스트에 저장합니다.
- DB저장은 트랜잭션(Transaction)에서 커밋(commit) 시점에 반영됩니다.
- 영속성 컨텍스트는 논리적 개념이며 눈에 보이지 않습니다. → 엔티티매니저를 통하여 영속성 컨텍스트에 접근

---

# 비영속 (new / transient)
- 영속성 컨텍스트와 전혀 관계없는 새로운 상태를 뜻합니다.

```java
import java.lang.reflect.Member;// 객체를 생성한 상태 (비영속) JPA와 전혀 관계없는 상태
Member member = new Member();
member.setid("member1");
member.setname("오팔봉");
```
영속상태가 아니므로 영속성 컨텍스트의 관리대상이 아니며 1차 캐시, 변경감지등의 기능이 적용되지 않습니다.

# 영속 (managed)
- 영속성 컨텍스트에 관리되는 상태
```java
// 객체를 저장한 상태 (영속)
em.persist(member);
```

## 영속성 컨텍스트의 이점
- ### 1차 캐시

  - [[JPA] JPA 1차 캐시란? 1차 캐시의 동작원리](https://tao-tech.tistory.com/7)   
1차 캐시 동작원리에 대하여 내용을 따로 다루었습니다.

- ### 동일성 보장

```java
import org.w3c.dom.ls.LSOutput;// 영속
Member findMember1 = em.find(Member.class, 101L);
Member findMember2 = em.find(Member.class, 101L);

System.out.println("result = " + (findMember1 == findMember2));

// 출력결과: result = true
```

1차 캐시로 반복 가능한 읽기(REPEATABLE READ) 등급의 트랜잭션 격리 수준을 데이터베이스가 아닌 애플리케이션 차원에서 제공합니다.

- ### 트랜잭션을 지원하는 쓰기 지연 (Transactional write-begind)
```java
// 트랜잭션 커밋 시점에 쿼리 생성 로직
public static void main(String[] args) {
    // 엔티티매니저팩토리 생성 persistence.xml에 접근
    EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");
    
    // 엔티티 매니저 생성
    EntityManager em = emf.createEntityManager();
    
    // 엔티티 트랜잭션 생성
    EntityTransaction tx = em.getTransaction();
    
    // 트랜잭션 시작
    tx.begin();
    
	try {
        // 비영속 상태
        Member member = new Member();
		member.setId(100L);
		member.setName("HelloJPA");

        // 영속 상태
		System.out.println("===== em.persist =====");
		em.persist(member); // 저장
		System.out.println("===== em.persist =====");

		System.out.println("===== transaction commit =====");
		tx.commit(); // 트랜잭션 커밋
		System.out.println("===== transaction commit =====");
        
	} catch (Exception e) {
        tx.rollback();
        
	} finally {
            em.close();
        
	}
            emf.close();
}
```

@Transaction 어노테이션 통해 접할 수 있지만 동작방식을 직관적으로 바라볼 수 있도록 직접 객체를 생성하였습니다.

![Alt text](/jpa/JPA 영속성컨텍스트 개념과 이점/images/1.png)

간단한 예시코드를 실행해 보면 객체를 저장(persist) 시점에 쿼리가 생성되지 않고 트랜잭션 커밋 시점에 쿼리가 생성되는 모습을 볼 수 있습니다.

- ### 변경감지 (Dirty Checking)
```java
transaction.bigin(); // 트랜잭션 시작

// 영속 엔티티 조회
Member memberA = em.find(Member.class, "memberA");

// 영속 엔티티 데이터 수정
MemberA.setUsername("오팔봉");
memberA.setAge(1);

transaction.commit(); // 트랜잭션 커밋
```

변경감지는 엔티티와 스냅샷을 비교합니다.   
1차 캐시에는 @id(PK), 엔티티, 스냅샷이 존재합니다. 값을 읽어온 최초의 상태를 스냅샷으로 저장합니다.   
JPA는 트랜잭션 커밋 시점에 내부적으로 *flush()가 호출되고 그때 JPA에서 내부적으로 @id(PK), 엔티티, 스냅샷을 비교합니다.   
그때 에를 들어 memberA의 객체가 바뀌었을 경우 update 쿼리를 쓰기 지연 sql저장소에 저장해둡니다.   
그리고 update 쿼리를 DB에 반영한 뒤 커밋이 진행됩니다.   

![Alt text](/jpa/JPA 영속성컨텍스트 개념과 이점/images/2.png)

> *flush?   
> 변경감지 → 수정된 엔티티 쓰기 지연 sql저장소에 등록 → 쓰기 지연 sql저장소의 쿼리를 데이터 베이스에 전송 (등록, 수정, 삭제)   
> 정리: 영속성 컨텍스트를 비우는게 아닌 영속성 컨텍스트의 변경 내용 (등록, 수정, 삭제)을 데이터 베이스에 동기화 하는 과정
> 
> 영속성 컨텍스트를 flush 하는 방법    
> em.flush(); → 직접 호출   
> 트랜잭션 커밋 → flush 자동 호출   
> JPQL 쿼리 실행 → flush 자동 호출

- ### 지연로딩 (Lazy Loading)
@ManyToOne 등의 종속관계에서 사용합니다.
```java
@ManyToOne(fetch = EAGER) // 즉시로딩 (default)
@ManyToOne(fetch = LAZY) // 지연로딩
```

지연로딩에 대해 알아보기 전 즉시로딩에 대해 먼저 알아보자면

#### 즉시로딩 (Eager)의 경우   
Member와 Team이 N:1 매핑으로 관계를 맺고 있을 경우 Member조회하는 시점에 바로 Team조회쿼리를 보내 한꺼번에 데이터를 불러옵니다.

![Alt text](/jpa/JPA 영속성컨텍스트 개념과 이점/images/3.png)

#### 지연로딩 (Lazy)의 경우
Member를 조회하더라도 Member를 조회하는 시점이 아닌 실제 Team을 사용하는 시점에 Team조회 쿼리를 보내도록 할 수 있습니다.

![Alt text](/jpa/JPA 영속성컨텍스트 개념과 이점/images/4.png)

# 준영속 (detached)
- 영속성 컨텍스트에 저장되었다가 분리된 상태

영속 → 준영속: 영속성 컨텍스트가 제공하는 기능을 사용하지 못합니다.

```java
// 준영속 상태로 만드는 방법
em.detach(entity); // 특정 엔티티만 준영속 상태로 전환
em.clear(); // 영속성 컨텍스트를 완전히 초기화
em.close(); // 영속성 컨텍스트를 종료
```

# 삭제 (removed)
- 삭제된 상태 (엔티티를 영속성컨텍스트와 데이터베이스에서 삭제)

```java
Member memberA = em.find(Member.class, "memberA");

em.remove(memberA); // 엔티티 삭제
```

변경감지 메커니즘과 같이 delete 쿼리가 생성되고 트랜잭션 커밋시점에 반영됩니다.