# 1차 캐시란?
- 영속성 컨텍스트 내부에는 엔티티(Entity)를 보관하는 저장소가 있는데 이것을 1차캐시라고 합니다.
- 엔티티 정 보를 키 벨류 값으로 가지고 있어 DB에 접근하는 횟수를 줄여 성능상의 이점을 이룰 수 있습니다.
- 엔티티매니저(EntityManager)로 변경, 조회하는 모든 엔티티는 1차 캐시에 저장되며 트랜잭션이 커밋되는 시점에 데이터베이스와 동기화 됩니다. (비교는 PK로 한다.)

---

## 1차 캐시 동작원리

![Alt text](/jpa/JPA1차캐시동작원리/images/1.png)

### find를 했을 때 해당 엔티티가 영속성 컨텍스트의 1차 캐시에 존재하고 있을 경우   
DB조회를 거치지 않고 1차 캐시에 저장되어 있는 엔티티를 반환합니다.

![Alt text](/jpa/JPA1차캐시동작원리/images/2.png)

### find를 했을 때 해당 엔티티가 영속성 컨텍스트의 1차 캐시에 존재하지 않을 경우
DB를 조회하고 조회한 엔티티를 1차 캐시에 저장 후 해당 엔티티를 반환해 줍니다.

이렇게 엔티티를 1차 캐시에 저장하여 같은 엔티티를 조회할 경우 영속성 컨텍스트에서 먼저 조회하여 DB의 접근을 줄여 성능상 이점을 이룰 수 있습니다.

간단한 테스트 코드를 통해 더욱 명확하게 알아보도록 하겠습니다.

---

```java
@Test
void cacheTest() {
    Member member = new Member();
    member.setUserEmail("member@email.com");
    member.setUserPw("1234");
    member.setUserId("member");
    member.setUserNm("member");

    memberRepository.save(member); // member 객체 저장
    em.clear(); // EntityManager Clear (영속성 컨텍스트 초기화)

    memberRepository.findOne(member.getMemberId()); // member 객체 조회1
    em.cleear(); // clear

    memberRepository.findOne(member.getMemberId()); // member 객체 조회2
}
```

위 코드를 보면 저장, 조회를 거칠 때 마다 영속성 컨텍스트를 초기화(em.clear();) 해주고 있습니다.

![Alt text](/jpa/JPA1차캐시동작원리/images/3.png)

테스트코드 실행 결과 저장, 조회1, 조회2 각각 DB접근이 일어난 것을 확인할 수 있습니다.

```java
@Test
void cacheTest() {
    Member member = new Member();
    member.setUserEmail("member@email.com");
    member.setUserPw("1234");
    member.setUserId("member");
    member.setUserNm("member");

    memberRepository.save(member); // member 객체 저장

    memberRepository.findOne(member.getMemberId()); // member 객체 조회1
    memberRepository.findOne(member.getMemberId()); // member 객체 조회2
}
```

영속성 컨텍스트를 초기화하지 않고 실행

![Alt text](/jpa/JPA1차캐시동작원리/images/4.png)

member객체를 저장할 때 DB에 접근 후 영속성컨텍스트에 저장되고 그림에서의 설명과 같이 1차 캐시에서 조회 후 반환되어 뒤의 member객체 조회 시 DB접근 없이 실행되는 것을 확인할 수 있습니다.