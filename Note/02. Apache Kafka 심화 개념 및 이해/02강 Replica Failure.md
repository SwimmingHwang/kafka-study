# Replica Failure

> 이전에 배운 내용

- ISR (In-Sync Replicas) 리스트 관리
  - Leader 가 관리함
  - **메시지가 ISR 리스트의 모든 Replica(복제본)에서 수신되면 Commit된 것으로 간주**
  - Kafka Cluster의 Controller가 모니터링하는 ZooKeeper의 ISR 리스트에 대한 변경 사항은 Leader가 유지
  - n개의 Replica가 있는 경우 n-1개의 장애를 허용할 수 있음
    - 나중에 다른 설정이 배우면 아닐 수 있음 
-  Follower가 실패하는 경우
  - Leader에 의해 ISR 리스트에서 삭제됨
  - Leader는 새로운 ISR을 사용하여 Commit 함
- Leader가 실패하는 경우
  - Controller는 Follower 중에서 새로운 Leader를 선출 
  - Controller는 새 Leader와 ISR 정보를 먼저 주키퍼에 Push 한 다음 로컬 캐싱을 위해 Broker에 푸시함



### ISR은 Leader가 관리한다. 

> **Leader 파티션이 있는 브로커가 Zookeeper에 ISR 업데이트**
> **Controller가 는 Partition Metadata에 대한 변경 사항에 대해서 Zookeeper로부터 수신**

- Follower가 너무 느리면 Leader는 ISR에서 Follower를 제거하고 ZooKeeper에 ISR을 유지

![image-20220426090951156](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220426090951156.png)



## Leader Failure

- Controller가 새로 선출한 Leader 및 ISR 정보는, Controller 장애로부터 보호하기 위해, ZooKeeper에 기록된 다음 클라이언트 메타데이터 업데이트를 위해 모든 Broker에 전파

![image-20220426091103524](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220426091103524.png)

## Broker Failure

Broker 4 대, Partition 4, Replication Factor가 3 일 경우를 가정

- 죽은 브로커의 follower 는 ISR에서 제거됨
  - 가만히 있음

- 리더가 죽었으니 **Topic1 파티션 3의 경우  101, 102 Follower 중 하나가 리더로 선출됨**

![image-20220426091135337](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220426091135337.png)



### Partition에 Leader가 없으면? 

Partition에 Leader가 없으면, 

- Leader가 선출될 때까지 해당 Partition을 사용할 수 없게 됨 
- Producer의 send() 는 retries 파라미터가 설정되어 있으면 재시도함 
- 만약 retries=0 이면, NetworkException이 발생함



# Summary

keyword : Replica Failure

- Follower가 실패하는 경우, Leader에 의해 ISR 리스트에서 삭제되고, Leader는 새로운 ISR을 사용하여 Commit함

- Leader가 실패하는 경우, Controller는 Follower 중에서 새로운 Leader를 선출하고, Controller는 새 Leader와 ISR 정보를 먼저 ZooKeeper에 Push한 다음 로컬 캐싱을 위해 Broker에 Push함
- Leader가 선출될 때까지 해당 Partition을 사용할 수 없게 됨
