# Topic, Partition, Segment

- Topic : 논리적인 표현으로 Kafka 안에서 메시지가 저장되는 장소 
- Producer Application
  - 메시지를 만들어서(produce) 토픽으로 보낸다
- Consumer Application
  - Topic의 메시지를 polling해서 가져와서 소비(consume)하는 애플리케이션
- **Consumer Group** 
  - **Topic의 메시지를 사용하기 위해 협력하는 Consumer들의 집합** 
  - 하나의 컨슈머는 하나의 컨슈머 그룹에 포합되며, 컨슈머 그룹 내의 컨슈머들은 협력하여 Topic의 메시지를 분산 병렬 처리한다 

![image-20220412090034855](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220412090034855.png)



## Producer와  Consumer의 분리(Decoupling)

- Producer와 Consumer는 서로 알지 못하며, Producer와 Consumer는 각각 고유의 속도로 Commit Log에 Write Read 를 수행
- 서로 다른 위치에서 따로따로 움직인다



## Commit Log

추가만 가능하고 변경 불가능한 **데이터 스트럭처** 

- 추가만 가능하고 변경이 불가능 함. 
- 데이터(event)는 항상 로그 끝에 추가되고 변경되지 않음 

![image-20220412090604227](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220412090604227.png)

- 0...~10 까지를 offset이라고 부른다 



- log-end-offset 
  - producer가 write를 하는 즉 add가 되는 끝 오프셋 
- current-offset
  - consumer가 가져가는 위치 
- 두 위치의 차이를 consumer lag 

![image-20220412091031427](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220412091031427.png)



## Topic

- Kafka 
  - 안에서 메시지가 저장되는 장소, 논리적인 표현
- Partition
  - **Commit Log**,  하나의 토픽은 하나 이상의 파티션으로 구성 
    병렬처리 (Throughout 향상)를 위해서 다수의 파티션을 사용 
- Segment
  - 메시지(데이터)가 저장되는 실제 물리 파일 



### Physical View 

- 토픽은 생성 시 파티션 개수를 지정하고, 각 파티션은 Broker들에 분산되며 Segment 파일로 구성된다

  - 분산은 카프카 클러스터 내에서 최적의 위치에 분산시킨다 

- 카프카 클러스터는 여러 개의 브로커로 이루어져 있으며 확장/축소가 가능하다 

  

![image-20220412092149646](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220412092149646.png)

- A 토픽에 파티션을 3개로 지정하면 각 파티션들이 브로커들에 분산된다 
- 세그먼트 파일은 용량 혹은 시간을 정해서 분리하고 rolling 하면서 생성한다
- 즉, 파티션 당 하나의 Segment 가 Active 되어있음 
  - 마지막 세그먼트에 데이터가 계속 쓰여짐



## Summary

- 토픽, 파티션, 세그먼트 특징 이해 
  - 토픽 생성 시 파티션 개수를 지정하고 개수 변경은 가능하나 운영시에는 변경을 권장하지 않는다
  - 파티션 번호는 0부터 시작하고 오름차순이다 
  - 토픽 내의 파티션들은 서로 독립적이다 
    - offset이 각각 다르다 
  - offset은 하나의 파니션 에서만 의미를 가진다 
  - offset값은 계속 증가하고 0으로 돌아가지 않는다 
  - Event(Message)의 순서는 하나의 파티션 내에서만 보장
  - 파티션에 저장된 데이터는 Immutable (변경이 불가능) 
  - 파티션에 Write 되는 데이터는 맨 끝에 추가되어 저장된다
  - 파티션은 세그먼트 파일로 구성되다
    - 롤링 정책에 용량 혹은 시간을 정할 수 있음
