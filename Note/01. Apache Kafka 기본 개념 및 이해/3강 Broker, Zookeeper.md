# Broker, Zookeeper

![image-20220413083631724](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220413083631724.png)

- 카프카 클러스터는 여러 개의 브로커로 구성될 수 있다 



## Kafka Broker

Kafka Broker는 파티션에 대한 Read 및 Write를 관리하는 소프트웨어 

- **Kafka Server**라고 부르기도 함 
- Topic 내 파티션 들을 분산, 유지 및 관리
- 각각의 브로커들은 ID 로 식별 됨(단, **ID는 숫자**)
- 앞 강의에서 설명했든 토픽의 일부 파티션들을 포함한다 
  - 하나의 토픽이 망가져도 전체 파티션에 문제가 생김을 방지하는 용도도 있음 
- **Kafka Cluster** : 여러 개의 브로커들로 구성된다 
- Client는 특정 브로커에연결하면 전체 클러스터에 연결된다
- **최소 3대 이상**의 브로커를 하나의 클러스터로 구성해야 하며 4대 이상을 권장한다 



> Kafka Broker ID 와 Partition ID는 아무런 관계가 없다 -> 어느 순서에나 있을 수 있다 





그럼, 클라이언트가 브로커에 접속을 할 때 어떻게 하느냐? 

## Bootstrap Servers

Broker Servers를 의미 

- 모든 카프카 브로커는 Bootstrap(부트스트랩) 서버라고 부름
  - 카프카 클라이언트(Producer/Consumer)가 하나의 브로커에 연결하면 자동으로 전체 프로커들 리스트를 전달해줌
    - 이정보를 통해 내가 접속해야 할 메시지를 주거나 받아가야할 토픽이 뭔지, 파티션이 어디에 있는지를 알게됨
    - 그리고나서 필요한 브로커에 연결을 한다 

![image-20220413084645259](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220413084645259.png)



## Zookeeper

Broker 관리, 카프카 입장에서 브로커들을 관리해준다. 목록, 설정 유지, 그리고 싱크도 해 줌

**Zookeeper는 Broker를 관리 (Broker 들의 목록/설정을 관리) 하는 소프트웨어**

- 토픽 생성/제거, 브로커 추가/제거 와 같은 변경사항에 대해 카프카에 알림 
- Zookeeper없이는 Kafka가 작동할 수 없음 
  - KIP-500 을 통해 2022년에 주키퍼 제거한 정식 버전 출시 예정
    - 카프카 향상 제안 500번 
- 홀수의 서버로 작동하게 설계되어 있음 
  - 최소 3, 권장 5 (큰 클러스터의 경우?)
  - (? 무슨 서버가 홀수로? **주키퍼 서버?(O)** 아니면 카프카 서버? 아니면 브로커?) 

- 주키퍼에는 Leader(writes)가 있고 나머지 서버에는 Follwer(reads)



### Zookeeper 아키텍쳐

Leader/Follower 기반 Master/Slace 아키텍처 

- 분산형 Configuration 정보 유지 
- 분산 동기화 서비스 제공
- 대용량 분산 시스템을 위한 네이밍 레지스트리를 제공하는 소프트웨어
- 토픽이 몇개고, 파티션이 몇개고 정보들을 주키퍼가 가지고 있고 
  메인으로 Leader가 가지고 있고 Follwer가 복제해서 가지고 있고
  밑에 브로커들한테 동기화해서 내려준다  
- 분산 작업을 제어하기 위한 Tree 형태의 데이터 저장소를 가지고 있음 
  - 브로커 간에 정보 공유 및 동기화 등을 수행한다 

- 주키퍼 클러스터를 주키퍼 앙상블이라고 부름 (최소3, 권장5)

![image-20220413090038607](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220413090038607.png)



### Zookeeper Failover 

Quorum 알고리즘 기반

- Ensemble은 주키퍼 서버의 클러스터
- 쿼럼 은 "정족수"를 의미하며, 합의체가 의사를 진행시키거나 의결을 하는데 필요한 최소 한도의 인원수를 뜻함

- 분산 코디네이션 환경에서 **예상치 못한 장애**가 발생해도 분산시스템의 일관성을 유지시키기 위해서 사용함
  - 앙상블이 3대로 구성되어 있다면 쿼럼은 2가 된다. 
    따라서 1대가 장애가 발생하면 2대가 살아남아 정상 동작 한다 (과반수 이상이니)
  - 앙상블이 5대면 쿼럼은 3이 된다
    따라서 2대가 장애 말생하더라도 정상동작, 3대가 죽으면 정상동작 X 
  - 4대면 2대 죽으면 정상동작 X, 과반을 넘지 못해서 
  - **즉, 3대를 쓰나 4대를 쓰나 2대가 죽으면 똑같이 정상동작 못해서 주키퍼 서버는 홀수로 씀**



## Summary

Zookeeper와 Broker는 서로 다르다 

- Broker는 Partition에 대한 Read 및 Write를 관리하는 소프트웨어 
- Broker는 Topic 내의 Partition 들을 분산, 유지 및 관리 
- 최소 3대 이상의 Broker를 하나의 Cluster로 구성해야 함 è 4대 이상을 권장함 
- Zookeeper는 Broker를 관리 (Broker 들의 목록/설정을 관리)하는 소프트웨어 
- Zookeeper는 홀수의 서버로 작동하게 설계되어 있음 (최소 3, 권장 5)
