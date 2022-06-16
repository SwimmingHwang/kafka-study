# Consumer Rebalance



> 복습

## Consumer의 동작 방식

-  100ms interval로 poll, 컨슈머가 각각 offset 저장 



## Consumer Load Balancing 

- 동일한 group.id로 구성된 모든 Consumer들은 하나의 Consumer Group을 형성
- Consumer Group의 Consumer들은 작업량을 어느 정도 균등하게 분할함
- 동일한 Topic 에서 consume하는 여러 Consumer Group이 있을 수 있음



## Consumer Group Coordination

- join group 요청 카프카로 나 브로커에 붙을거야 라는 요청 

- 컨슈머보다 파티션 수가 ㅁ낳을 때 
- 컨슈머가 남아 돌면 
  - 컨슈머에 파티션 매핑  
  - C6은 할당을 못받음 
  - -> 그룹 코디네이텋나테 다시 보내짐 



#### 왜 Group Coordinator(a Broker) 가 직접 파티션을 할당하지 않는가?

-  Kafka의 한 가지 원칙은 가능한 한 많은 계산을 클라이언트에 수행하도록 하여, Broker의 부담을 줄이는 것 

- 많은 Consumer Group과 Consumer들이 있고 Broker 혼자서 Rebalance를 위한 계산을 한다고 생각해 보면... 
  - Broker에 엄청난 부담 
  - 이러한 계산을 Broker가 아닌 클라이언트에게 오프로드(Offload)하는 것이 가장 바람직함



## Consumer Rebalancing Trigger 

불필요한 Rebalancing 은 피해야 함 

- **Rebalancing Trigger**
  - Consuer가 Consumer 그룹에서 탈퇴
  - 신규 Consumer가 Consumer Group에 합류
  - Consumer가 Topic 구독을 변경
  - Consumer Group은 Topic 메타데이터의 변경 사항을 인지 (ex 파티션 증가) 
- Rebalancing Process
  - 신규로 붇는 경우
    - 컨슈머가 조인 Request 를 날려서 -> 인지 
  - 빠진 경우
    - Group Coordinator는  **heartneats**의 플래그를 사용하여 Consumer에게 Rebalance 신호를 보냄 
    - 컨슈머 한테 날리는데 반응이 없으면 빠졌다고 생각
  - Consumer가 일시 중지하고 offset을 Commit
  - Consumer는 Consumer Group의 새로운 ”Generation"에 다시 합류
  - Partition 재할당
  - Consumer는 새 Partition에서 다시 Consume을 시작



> **Consumer Rebalancing시 Consumer들은 메시지를 Consume하지 못함 따라서, 불필요한 Rebalancing은 반드시 피해야 함**



## Consumer Heartbeats

Consumer 장애를 인지하기 위함 

- Consumer는 poll()과 별도로 백그라운드 Thread에서 Heartbeats를 보냄 
  - heartbeat.interval.ms (기본값 : 3 초) 
- 아래 시간 동안 Heartbeats가 수신되지 않으면 Consumer는 Consumer Group에서 삭제
  -  session.timeout.ms (기본값 : 10 초) 
- poll()은 Heartbeats와 상관없이 주기적으로 호출되어야 함 
  - max.poll.interval.ms (기본값 : 5 분)



**heartbeat랑 poll은 별개로 호출 됨** 





## 과도한 Rebalancing을 피하는 방법

성능 최적함 필수

1. Consumer Group멤버 고정

   - Group의 각 Consumer에게 고유한 group.instance.id 를 할당합니다. 

   - Consumer는 LeaveGroupRequest를 사용하지 않아야 함 

   - Rejoin(재가입)은 알려진 group.instance.id 에 대한 Rebalance를 trigger하지 않음
   - "id를 쓰는 이유는 안 빠진다. 잠시 죽었다가 다시 rejoin할거다

2. session.timeout.ms 튜닝

   - **heartbeat.interval.ms를 session.timeout.ms의 1/3로 설정** 
   - group.min.session.timeout.ms (Default: 6 seconds) 와 group.max.session.timeout.ms (Default: 5 minutes) 의 사이값
   - 장점 : Consumer가 Rejoin(재가입)할 수 있는 더 많은 시간을 제공 
   - 단점 : Consumer 장애를 감지하는 데 시간이 더 오래 걸림

3. max.poll.interval.ms 튜닝

   - Consumer에게 poll()한 데이터를 처리할 수 있는 충분한 시간 제공
     - 데이터를 왕창 가져올 수 있으니, poll시간 로직이 짧다면 폴 하다가 컨슈머 그룹에서 빠질 수 있으니.. 
   - 너무 크게 하면 안 됨

**항상 trade off 가 있다. 테스트 하면서 튜닝이 필요하다!**





## Summary

- Consumer의 파라미터인 partition.assignment.strategy 로 Partition 할당 방식 조정 
- Consumer Rebalancing의 Trigger 조건들 
- 과도한 Consumer Rebalancing을 피해야 함
