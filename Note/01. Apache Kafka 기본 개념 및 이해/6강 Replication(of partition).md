---
typora-copy-images-to: ..\img
---

# Replication

> Broker에 장애가 발생하면 어떻게 될까?

![image-20220419084402504](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220419084402504-16503254448151.png)

- 기본적으로, Producer는 메시지를 못 보내고 Consumer는 메시지를 가져갈 수 없게 된다
- Partition 관련 데이터, LOG-END-OFFSET, CURRENT-OFFSET 은 어떻게 될까? 



다른 Broker에서 Partition을 새로 만들 수 있으면 장애 해결이 되나? 

- 기존 브로커에 있는 쌓아둔 데이터는? 기존 메시지는 버릴 것인가? 

![image-20220419084855253](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220419084855253-16503257363892.png)

이를 해결하기 위해 만든 기술 

## Replication(복제) of Partition

장애를 대비하기 위한 기술

- 파티션을 복제하여 다른 브로커 상에서 Replicas를 만들어서 장애를 미리 대비한다 
- Producer가 write 할 때 마다 바로바로 복제를 한다 
- 총 세개의 Replica 예시 그림
- Leader, Follower **파티션**이 있다
  - **복제하는 기준은 파티션 단위!!** (내가 헷갈렸던 부분) 

![image-20220419085128397](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220419085128397-16503258895374.png)

- Producer는 Leader에만 write 하고 Consumer는 Leader 로 부터만 Read함 
- Follower 는 Broker장애 시 안정성을 제공하기 위해서만 존재한다
- Follower는 Leader의 Commit Log에서 데이터를 가져오기 요청 (Fetch Request - 데이터 달라고 요청)로 복제한다
  - Apache Kafka 2.4 부터는 Follower가 Fetching(Read) 할 수 있게 설정할 수 있는 옵션을 제공한다



## Leader 장애 

>  리더에 장애가 발생하면? 

- Kafka 클러스터는 Follower 중에서 새로운 Leader를 선출한다 
- Clients(Producer/Consumer)는 자동으로 새 Leader로 전환하여 read/write한다





## Partition Leader에 대한 자동 분산

> 하나의 Broker에만 파티션의 리더들이 몰려있으면? 

- Hot Spot 
- 특정 브로커에만 Clients가 있어 부하가 집중된다 
- 이런 상황은 피해야 한다

![image-20220419085706136](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220419085706136-16503262273435.png)



### Hot Spot 을 방지하기 위한 옵션

- `auto.leader.rebalance.enable` : 기본값 enable
- `leader.imbalance.check.interval.seconds` : 기본값 300sec
  - 300초 텀으로 리더 불균형 있는지 체크
- `leader.imbalance.per.broker.percentage` : 기본값 10 
  - 다른 브로커 보다 10% 이상 더 많이 가져가면 불균형이라고 판단



## Rack Awareness

- ***랙(Rack Enclosure)은 내부에 국제 표준 규격의 장비(서버, 통신장비 등)을 설치하여 시스템 구성에 필요한 환경을 제공하고 장비의 보호 등의 기능을 수행하는 장비***
- AZ 자체가 죽으면 ?! 
  - 다른 랙에 분산해서 설치를 해야 한다 
    - 다른 랙(위치) 에 있다고 브로커에 알려줘야 한다
      - `broker.rack=ap-northeast-2a` 
    - **한 rack 이 죽어도 다른 rack으로 정상 구동되게 분산할 필요가 있다**



- 동일한 rack 혹은 Available Zone 상의 Broker들에 동일한 rack name 지정 
- 복제본(Replica-Leader/Follower) 은 최대한 Rack 간에 균형을 유지하여 Rack 장애 대비 
- Topic 생성시 또는 Auto Data Balancer/Self Balancing Cluster 동작 때만 실행
  - **Broker Cluster**가 토픽 생성시 rack name을 보고 분산시켜준다.

![image-20220419090518551](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220419090518551-16503267197486.png)



## Summary 

- 파티션을 복제하여 다른 브로커 상에서 복제물을 만들어 장애 미리 대비
- Replicas 는 Leader Partition, Follower Partition 이 있다
- Producer는 Leader에만 write하고 Consumer는 Leader로 부터 만 Read함
- Follower는 Leader의 Commit Log에서 Fetch Request로 복제
- 복제본은 최대한 Rack 간 균형을 유지하여 Rack 장애를 대비하는 Rack Awareness 기능이 있다.
