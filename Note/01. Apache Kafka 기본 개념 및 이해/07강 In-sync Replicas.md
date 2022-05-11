# In-Sync Replicas

- 지난 시간에 파티션을 복제하여 다른 브로커 상에서 복제물을 만든다는 것을 배웠음



## In-Sync Replicas(ISR)

> Leader 장애시 Leader를 선출하는데 사용

- ISR 은 High water Mark라고 하는 지점까지 동일한 Replica(Leader와 Follower 모두)의 목록
- 진짜로 잘 복제해 가고 있는를 나타내는 지표 (=내가 이해한 바로는, 리더가 될 수 있는 Replica 후보자 느낌)
  - `replica.log.max.messages=4` 
    - 복제를 잘 해가고 있는지, 즉 차
- LOG-END-OFFSET : Leader의 commit offset
- Fully-Replicated Committed : Follower중에 잘 따라 잡는 애가 완전 잘 복제하고 그 복사한 위치가 range 4 안에 있다면 High Water Mark 지점을 의미. 
  - 여기서 commit 은 consumer와 달리 여기까지 복제를 했다는 표시
- Leader는 당연히 ISR
- 못따라 잡고 있는 애들(OSR, Out-of-Sync)을 Leader로 선출하면 안 된다. 

![image-20220420085357196](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220420085357196.png)



### Replica.log.max.messages 사용시 문제점

메시지 유입량이 갑자기 늘어날 경우

- 메시지가 일정하게 kafka 로 들어올 때는 5개 이상 지연되는 경우가 없어서 ISR이 정상 동작
- 갑자기 유입량이 늘어나면 지연으로 판단하고 OSR 상태로 변경시킴

-> 불필요한 에러, Producer에서의 Retry 가 발생한다

그래서

**replica.log.time.mas.ms** 로 판단해야 함! 

- Follower가 Leader로 Fetch요청을 보내는 Interval 확인
  - 이 값이 10000이라면 Follower가 Leader로 Fetch 요청을 10000ms 내에만 요청하면 정상으로 판단

>  Confluent에서는 replica.log.time.max.ms 옵션만 제공(복잡성 제거)



## ISR 은 Leader가 관리 

- ISR 은 해당 파티션의 리더가 떠있는 **브로커**가 관리한다

1. Follower가 느리면 리더는 ISR에서 Follower 103 제거, Zookeeper에 알려줘서 ISR101, 102 유지
2. controller는 파티션 metadate에 대한 변경 사항에 대해 주키퍼로 부터 수신
   - controller : 브로커 중 한대
   - 변경사항을 다른 브로커들에게 파티션 ISR을 알려줌

![image-20220420090556744](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220420090556744.png)



### Controller 

Kafka Cluster 내의 브로커 중 하나가 Controller가 됨 

- Controller는 zookeeper 를 통해 Broker Liveness 를 모니터링 
- Controller는 Leader와 Replica 정보를 Cluster 내의 다른 브로커들에게 전달
- **즉, 컨트롤러는 주키퍼에 Replicas정보의 복사본 유지, 모든 브로커들에게 동일한 정보를 빠른 액세스를 위해 캐시함**
- 리더가 장애나면 (리더가 있는 그 파티션의 브로커가 죽으면) Leader Election 선출을 수행한다. 
- Controller가 장애나면 주키퍼가 Active 브로커들 중에서 선출함





## Consumer 관련 Position 들 

- Last Committed Offset(Current Offset) : Consumer가 최종 Commit한 Offset 
- Current Position : Consumer가 읽어간 위치(처리 중, Commit 전) 
- High Water Mark(Committed) : ISR(Leader-Follower)간에 복제된 Offset 
- Log End Offset : Producer가 메시지를 보내서 저장된, 로그의 맨 끝 Offset

![image-20220421083307011](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421083307011.png)



### Committed 의 의미 

- ISR 목록의 모든 Replicas가 메시지를 성공적으로 가져오면 "Committed"
- **Consumer는 Commited메시지만 읽을 수 있음**
- 리더는 메시지를 Commit 할 시기를 결정 (?) 뒤에 나옴 
- Committed 메시지는 모든 Follower에서 동일한 Offset을 갖도록 보장 
- **어떤 레플리카라도 리더인지에 관계 없이 모든 컨슈머는 해당 5번 offset에서 같은 데이터를 볼 수 있음** 
- Broker가 다시 시작할 때 Committed 메시지 목록을 유지하도록 하기 위해, Broker의 모든 Partition에 대한 마지막 Committed Offset은 replicationoffset-checkpoint라는 파일에 기록됨
  - 여기를 디져서 확인하고 복제해감

![image-20220421083718141](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421083718141.png)

## Replicas 동기화

### High Water Mark 

- ISR(Leader-Follower)간에 복제된 Offset
- 가장 최근에 Committed 메시지의 Offset 추적 
- replication-offset-checkpoint 파일에 체크포인트를 기록

### Leader Epoch 

- 새 Leader가 선출된 시점을 Offset으로 표시
  - 예를 들어 리더가 동작하다가 리더가 죽었을 때 새로운 Epoch가 시작하게 됨 (0 에서 1)
- Broker 복구 중에 메시지를 체크포인트로 자른 다음 현재 Leader를 따르기 위해 사용됨 
- Controller가 새 Leader를 선택하면 Leader Epoch를 업데이트하고 해당 정보를 ISR  목록의 모든 구성원에게 보냄 
- leader-epoch-checkpoint 파일에 체크포인트를 기록



## Message Commit 과정

- Follower는 Leader 로 부터 fetch 만 수행 한다 
- Fetcher Thread가 모든 브로커 별로 존재한다
  - 리더로 부터 데이터를 fetch 하는 역할을 한다
  - 한 브로커에 리더 파티션도 있지만 follower 파티션도 있을 수 있기에 모든 브로커에 Fetcher Thread가 존재한다 



1. Offset 5 까지 복제가 완료되어 있는 상황에서, Producer가 메시지를 보내면 Leader가 offset 6 에 새 메시지를 추가

![image-20220421084434343](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421084434343.png)

2. 각 Follower들의 Fetcher Thread가 독립적으로 fetch를 수행하고, 가져온 메시지를 offset 6 에 메시지를 Write

![image-20220421084503547](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421084503547.png)

3. 각 Follower들의 Fetcher Thread가 독립적으로 다시 fetch를 수행하고 null 을 받음 **Leader는 High Water Mark 이동**

![image-20220421084529119](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421084529119.png)

4. 각 Follower들의 Fetcher Thread가 독립적으로 다시 fetch를 수행하고 High Water  Mark를 받음

![image-20220421084611022](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421084611022.png)



## Summary

키워드 : ISR, Committed, High Water Mark, Controller

- In-Sync Replicas(ISR)는 High Water Mark라고 하는 지점까지 동일한 Replicas  (Leader와 Follower 모두)의 목록 
- High Water Mark(Committed) : ISR(Leader-Follower)간에 복제된 Offset 
- Kafka Cluster 내의 Broker중 하나가 Controller가 됨 
- Controller는 ZooKeeper를 통해 Broker Liveness를 모니터링