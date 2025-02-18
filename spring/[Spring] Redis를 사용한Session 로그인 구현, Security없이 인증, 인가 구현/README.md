애플리케이션의 인증과 인가는 사용자와 시스템 간의 신뢰를 형성하는 중요한 요소이며, 이를 통해 사용자의 신원 확인 및 접근 권한을 관리함으로써 보안성을 높일 수 있습니다.   
이번 포스팅에서는 `Session과 Redis`의 개념을 간단히 살펴본 후, `Redis를 세션 스토리지로 사용하는 로그인 구현`과 Spring Security 없이 `Custom Annotation을 통한 인증/인가`를 구현하는 방법을 알아보겠습니다.   

# Session 이란?

<img width="640" alt="Image" src="https://github.com/user-attachments/assets/23c2f195-d2ca-4240-9640-a2d0e9be4da0" />

Session이란, 클라이언트와 서버 간의 상태를 유지하기 위한 방법으로, 사용자가 로그인하여 인증된 후 해당 사용자의 정보를 일정 기간 동안 서버가 기억할 수 있도록 합니다.   
HTTP는 기본적으로 비상태성`stateless`을 지니고 있어, 각 요청은 서로 독립적입니다. 따라서 사용자가 로그인한 후에도, 서버는 매번 새로운 요청으로 간주하여 사용자를 기억하지 못합니다.   
Session은 이러한 비상태성을 보완하기 위해 도입된 개념으로, `특정 사용자를 식별하고 상태를 유지`할 수 있도록 합니다.   

## Session의 한계점

- 서버 자원 소모
  - 모든 사용자의 세션 정보를 서버가 관리하여, 사용자가 많아질수록 메모리와 CPU 리소스가 많이 소비됩니다.   
- 스케일링의 어려움
  - Session 정보가 서버 메모리에 저장되기 때문에 서버를 확장할 때 같은 세션 데이터를 공유하는 데 어려움이 생깁닌다.   
- 클라이언트-서버 분리
  - REST API를 사용하는 경우, HTTP의 비상태성을 유지하는 것이 권장되는데, 세션은 상태 유지를 요구하여 해당 정보를 관리하는 서버에서는 RESTful 아키텍처와 충돌할 수 있습니다.

Session의 스케일링 문제를 해결하기 위해서 스티키 세션`Sticky Session`또는 세션 클러스터링`Session Clustering` 방식을 사용하게 됩니다.   

## 스티키 세션(Sticky Session)?

Sticky Session은 클라이언트가 항상 같은 서버에 연결되어 세션을 유지하는 방식입니다.   

<img width="640" alt="Image" src="https://github.com/user-attachments/assets/a27b6502-b40c-4979-bb7b-2b84c4607f4c" />   

Sticky Session은 클라이언트가 항상 같은 서버에 연결되도록 하여, 특정 서버에서 세션 상태를 유지하는 방식입니다.   
이는 초기 설정이 간단할 수 있지만, 특정 서버가 장애를 일으키거나 다운될 경우 해당 서버에 연결된 클라이언트의 세션 정보는 손실됩니다. 또한, 부하가 특정 서버에 몰리는 문제도 발생할 수 있습니다.   
특정 서버에 과부하가 걸리는 경우 로드 밸런서`Load Balancer`는 이를 감지하여 해당 서버로 향하는 트래픽을 다른 서버로 라우팅 하는데, 이럴 경우 기존 세션 데이터가 유실되며 이용하던 사용자는 재로그인을 하는 등 사용자 경험을 저하시킬 수 있습니다.

이를 보완하여 정합성 이슈를 해결하고, 가용성과 트래픽 분산까지 확보할 수 있는 세션 관리방식을 위해 세션 클러스터링 방식을 사용합니다.

## 세션 클러스터링 (Session Clustering)?

Session Clustering은 여러 서버에서 세션 데이터를 공유하고 관리하는 방식입니다.   

<img width="640" alt="Image" src="https://github.com/user-attachments/assets/17a13664-ad22-4861-93aa-4453bb8e02a3" />

위 이미지는 `All-to-all Session Replication`방식을 사용하여, 각 서버가 서로의 세션 데이터를 동기화하는 방식입니다.   
사용자가 다른 서버에 요청을 보낼 때도 동일한 세션 데이터를 사용할 수 있어, 세션의 일관성을 유지할 수 있습니다.   
하지만 세션 클러스터링을 구현하기 위해 여러 서버 간에 세션 데이터를 동기화해야 하므로 네트워크 트래픽이 증가하고, 동기화 과정에서 서능 저하가 발생할 수 있습니다.   
또한, 세션 정보가 각 서버에 저장되므로 서버 장애 시 해당 서버의 세션 데이터를 다른 서버로 빠르게 이전해야 하며, 이 과정에서 세션 복구 시간이 중요하고, 장애 처리를 위한 추가적인 관리가 필요합니다.   

이러한 Session의 한계점을 보완하기 위해 Redis를 Session 스토리지로 활용할 수 있습니다.

[인증, 인가 세션과 토큰의 장단점과 차이점](https://tao-tech.tistory.com/18)

# Redis 란?

<img width="640" alt="Image" src="https://github.com/user-attachments/assets/e549cdcb-50c3-4a0d-8e1c-e498fb1fe44b" />

Redis는 메모리 기반의 데이터 저장소로, 빠르고 효율적인 데이터 관리가 가능합니다. `key-value`형태로 데이터를 메모리에 저장하여 디스크 기반의 데이터베이스보다 더욱 빠르게 데이터를 읽고 쓸 수 있는 장점이 있습니다.   
또한, 분산 시스템으로 설계되어 여러 서버에서 데이터를 효율적으로 관리하고 확장할 수 있습니다.   

## 특징

- 빠른 성능
  - 모든 데이터를 메모리에 저장하여, 디스크 I/O가 필요한 다른 데이터베이스보다 빠릅니다. 이 덕분에 세션 데이터를 실시간으로 처리해야 하는 경우 매우 유용합니다.  
- 확장성
  - 분산 시스템으로 쉽게 확장할 수 있습니다. 여러 서버에 걸쳐 데이터를 분산 저장할 수 있기 때문에, 세션 데이터를 여러 서버에서 공유하고 관리할 수 있습니다.   
- 다양한 데이터 구조 지원
  - 단순한 key-value 저장 외에도 리스트, 셋, 해시, 정렬된 셋 등 다양한 데이터 구조를 지원합니다. 이로 인해 세션 데이터를 보다 효율적으로 저장하고 처리할 수 있습니다.
- 영속성 옵션
  - 기본적으로 메모리 기반이지만, 데이터를 디스크에 비동기적으로 저장하여 영속성도 지원합니다. 이 기능은 세션 정보를 안정적으로 관리하는데 유리합니다.   

## Redis를 Session 스토리지로 사용하는 이유

<img width="640" alt="Image" src="https://github.com/user-attachments/assets/eedb38ba-5713-4708-b445-263aa81b4a1b" />

저장소가 RDB라면 디스크 I/O 작업이 많아져 서버의 메모리 부하가 커져 성능에 영향을 끼칠 수 있습니다.   
그러나, Redis는 빠른 성능과 높은 확장성을 제공하는 메모리 기반의 데이터 저장소입니다. 디스크 I/O 작업을 하지 않아 메모리 낭비를 줄일 수 있습니다.   
세션 데이터를 Redis에 저장하면, 여러 서버 간에 데이터를 효율적으로 관리하고 공유할 수 있으며, 서버가 확장될 때 추가적인 세션 정보 관리가 용이해집니다.     
Redis는 분산 시스템으로 여러 서버 간의 세션 데이터를 동기화하고 영속성을 제공하여 세션 정보의 유실을 방지할 수 있습니다.
