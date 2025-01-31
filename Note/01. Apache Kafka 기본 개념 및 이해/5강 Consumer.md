

# Consumer

Partition 으로 부터 Record를 가져옴(poll)

- Consumer는 각각 고유의 속도로 Commit Log로부터 순서대로 Read(Poll)를 수행

![image-20220415175546260](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415175546260.png)

## Consumer Offset

- Consumer Group이 읽은 위치를 표시  (실제로 internal topic에 그 다음 위치를 저장한다)
- 사용하는 이유 : Consumer가 자동이나 수동으로 데이터를 읽은 위치를 commit 하여 다시 읽음 막으려고 
- __consumer_offsets 이라는 Internal Topic에 Consumer Offset을 저장하여 관리한다 



![image-20220415175744752](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415175744752.png)



* 한 토픽에 여러 컨슈머 가능, 한 토픽에 여러 컨슈머 그룹도 가능 



### Multi-Partitions with Single Consumer 
하나의 컨슈머가 여러 개의 파티션을 컨슘. 이 컨슈머는 Topic의 모든 파티션에서 모든 Record를 컨슘한다. 

- 하나의 컨슈머는 각 파티션에서의 Consumer Offset을 별도로 기록하면서 모든 파티션에서 컨슘함. 

![image-20220418125253276](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418125253276.png)



### Consuming as a Group

동일한 group.id로 구성된 모든 컨슈머들은 하나의 Consumer Group을 형성한다 

- **4개의 파티션이 있는 Topic 을 컨슘하는 4개의 컨슈머가 하나의 컨슈머 그룹에 있다면 각 컨슈머는 정확히 하나의 파티션에서 Record를 Consume한다.**

- **파티션은 항상 컨슈머 그룹 내의 하나의 컨슈머에 의해서만 사용된다** 

- 컨슈머는 주어진 Topic에서 0개 이상의 ㅁ낳은 파티션을 사용할 수 있다.

  


![image-20220415175744752](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418124149122.png)



### Multi Consumer Group

파티션을 분배하여 컨슘 

- 컨슈머 그룹의 컨슈머들은 **작업량을 어느 정도 균등하게 분할함**
- 동일한 토픽에서 컨슘하는 여러 컨슈머 그룹이 있을 수 있음
  - 동일한 토픽의 파티션을 여러 개의 컨슈머 그룹이 컨슘할 수 있다 

![image-20220415175744752](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418124420744.png)



### Record에 Key를 사용하면

**앞에서 했던 내용 -> Message 순서와 관련있음.** 

파티션 별로 동일한 Key를 가지는 메시지를 저장한다.

- 알고리즘에 의해 key가 AAA 라면 파티션 P0에만 구분되어 들어간다 
- DDD면 P3에만 들어간다 



![image-20220415175744752](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418124543988.png)



### Message Ordering

**파티션이 2개 이상인 경우 모든 메시지에 대한 전체 순서 보장 불가능**

- 파티션을 1개로 구성하면 모든 메시지에서 전체 순서 보장 가능 -> 처리량 저하 
  - 카프카가 병렬 처리를 위한 용도로 만들었기에 파티션을 1개만 쓰면 병렬 처리가 불가하다. 
  - 그럼, 파티션을 1개로 구성해서 순서를 보장해야 하는 경우가 얼마나 많을까? 뒤에 이어서 설명

![image-20220418125936641](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418125936641.png)





#### 파티션을 1개로 구성해서 모든 메시지에서 전체 순서 보장을 해야 하는 경우가 얼마나 많을까?

- 사용자 A, B, C 의 전체 액션의 순서는 중요하지 않다. 각 사용자의 액션의 순서가 중요하다 
- 데이터베이스 변경사항도 특정 테이블에서의 변경 사항만 보면 된다. 모든 테이블의 변경 사항을 순서가 보장될 필요는 없다. 
- 즉, **대부분의 경우 Key로 구분할 수 있는 메시지들의 순서 보장이 필요한 경우가 많다.** 



#### Key를 사용하여 파티션 별 메시지 순서 보장 

- 동일한 Key를 가진 메시지는 동일한 Partition에만 전달되어 Key 레벨의 순서 보장 가능
  - 멀티 파티션 사용 가능 =  처리량 증가
- 운영 중에 파티션 개수를 변경하면? **순서 보장 불가**
  - 해시 알고리즘에서 파티션 개수로 나누게 되는데(나머지 연산), 
    파티션 수가 바뀌면 나머지의 숫자가 바뀌게 되어 파티션 숫자가 뒤섞이고
    순서 보장이 불가능하게 되며 위험하다. (의도치 않은)

![image-20220418130609848](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418130609848.png)



### Cardinality 

특정 데이터 집합에서 유니크(Unique) 한 값의 개수 

**Key Cardinality 는 Consumer Group의 개별 Consumer가 수행하는 작업의 양에 영향**

파티션에 네개고 컨슈머가 네개다. 나이스하게 분산처리를 똑같이 할 수 있지만 이는 키가 분포가 이상적이지 않다면 특정 파티션에 부하가 쏠릴 수 있다. 

즉, 

- Key 선택이 잘 못되면 작업 부하가 고르지 않을 수 있다
- Key 는 Integer, String 등과 같은 단순한 유형일 필요가 없다. 
- Key는 Value와 마찬가지로, Avro, JSON 등 여러 필드가 있는 복잡한 객체일 수 있다. 
- **따라서, 파티션 전체에 Record를 고르게 배포하는 Key를 만드는 것이 중요하다.**



### Consumer Failure 

#### Consumer Rebalancing 

컨슈머 장애가 생기면 컨슈머를 리밸런싱하여 장애가 났던 컨슈머가 땡겨갔던 파티션을 다른 컨슈머가 컨슈밍하여 가져갈 수 있게 한다. 

- 조건은 동일하다
  - 파티션은 항상 컨슈머 그룹 내의 하나의 컨슈머에 의해서만 사용된다 
  - 컨슈머는 주어진 토픽에서 0개 이상의 많은 파티션을 사용할 수 있음. 

![image-20220418133314610](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220418133314610.png)

- 컨슈머 그룹의 다른 컨슈머가 실패한 컨슈머를 대신함 

나중에 심화 과정에서 어떤 과정(알고리즘)으로 파티션이 assign 되는지 배움. 



## Summary

- Offset, Consumer Group, 순서 보장, Consumer Rebalancing 

- 컨슈머가 자동이나 수동으로 데이터를 읽은 위치를 commit 하여 다시 읽음을 방지한다 

- __consumer_offsets 라는 Internal Topic에서 Consumer Offset을 저장하여 관리한다

- 동일한 group.id로 구성된 모든 Consumer들은 하나의 컨슈머 그룹을 형성한다 

- 다른 컨슈머 그룹의 컨슈머 들은 분리되어 독립적으로 작동한다

- 동일한 Key를 가진 메시지는 동일한 파티션에만 전달되어 Key레벨의 순서 보장이 가능하다 

- Key 선택이 잘 못되면 작업 부하가 고르지 않을 수 있다

- 컨슈머 그룹 내에서 Consumer가 실패한 컨슈머를 대신해 파티션에서 데이터를 가져와서 처리한다.

  
