# Replica Recovery

> 결론 : ack parameter가 중요하다



### acks=all 의 중요성

- enable.idempotence=false : 순서보장을 하지 않는다 

![image-20220511085642636](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220511085642636.png)

### acks=all 중복이 생길 수 있는 Flow 

- 메시지 1,2,3,4 가 LeaderX에 들어온다 
  - Leader X는 1,2 까지만 커밋되어있다
- Follower Y 는 M3까지 copy했고 M2까지 커밋, Follower Z 는 M2 까지 copy, commit 한 상태 
  High Water Mark 는 M2까지 동일(화살표)

- **Broker X가 장애**
  - M3, M4를 커밋하기 전에 Broker X 장애가 발생했다 

![image-20220511092304390](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220511092304390.png)

- **새로운 Leader가 선출 됨**

- Controller가 Y를 Leader로 선출
  - Leader Epoch가 0 에서 1로 증가
- Z 는 M3을 fetch 함
- Y는 High Water Mark를 진행
  - 리더로써 follwer로 복제된 offset 이동 시키는 작업
- Z는 fetch 다시 수행하고 High  Water Mark를 수신하고 진행

![image-20220511092339114](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220511092339114.png)

- 그럼 M3는 Commit 된 적이 없는데 왜 Y는 M3를 가지고 있을 수 있는가?
  - M3는 High water mark를 찍지 못하고 복제까지만 된 상태인데, Leader에 있는 파티션 데이터를 지우지 않는다 
  - 리더의 데이터를 다시 새로운 리더 파티션으로 .. 
- Producer는 M3, M4에 대한 ack 미수신 M3, M4를 Y로 송신 재시도
- idempotence=false 이므로 M3는 중복 발생
- Z는 M3(중복), M4를 fetch함
- Z는 리더를 따라가니 fetch 다시 수행하고 High  Water Mark를 수신하고 진행



**즉, acks=all 은 at least once 이기 때문에 중복이 발생할 수 있다.**



> **Committed 의 의미 복습 **
>
> - ISR 목록의 모든 Replicas가 메시지를 성공적으로 가져오면 “Committed”
>
> **High Water Mark  복습**
>
> - ISR(Leader-Follower)간에 복제된 Offset
> - 가장 최근에 Committed 메시지의 Offset 추적 
> - replication-offset-checkpoint 파일에 체크포인트를 기록
>
> **Leader Epoch 복습** 
>
> - 새 Leader가 선출된 시점을 Offset으로 표시
>   예를 들어 리더가 동작하다가 리더가 죽었을 때 새로운 Epoch가 시작하게 됨 (0 에서 1)
> - Broker 복구 중에 메시지를 체크포인트로 자른 다음 현재 Leader를 따르기 위해 사용됨
> - Controller가 새 Leader를 선택하면 Leader Epoch를 업데이트하고 해당 정보를 ISR 목록의 모든 구성원에게 보냄
> - leader-epoch-checkpoint 파일에 체크포인트를 기록
>
> (In part1 > chaper 1 > 07강 In-sync Replicas )



### acks=1 이었다면?

- Y, Z 가 복제를 못했던 M4는 어떻게 되나?

  - M3 는 Y가 복제함 

- Leader X가 장애나기 전에 Producer는 M4에 대한 ack를 수신함 

  - Producer는 M4까지 다 보냈다고 인식
  - **-> Producer가 M4 송신을 재시도 하지 않게 되어 M4는 유실된다**

- Y가 Leader로 선출 Leader Epoch가 0에서 1로 증가

- Z는 M3를 fetch함

- Y는 High Water Mark를 진행

- Z는 fetch 다시 수행하고 High  Water Mark를 수신하고 진행

  

![image-20220511093935455](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220511093935455.png)

**즉, acks=1 은 at most once 이기 때문에 유실이 충분히 생길 수 있다**



## acks=all, 장애가 발생했던 X가 복구되면?

- X가 복구되면 주키퍼에 연결됨
- controller 로 부터 metadata를 받음
- Leader U로 부터 Leader Epoch를 fetch
- X는 리더 epoch 가 시작한 M2 까지 데이터만 살리고 잘못된 garbage데이터는 X Follower에서 지워버린다 

![image-20220511095156650](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220511095156650.png)



- X는 Leader Y로 부터 복제
- 복제가 한번 일어나면, ISR리스트에 복귀 





## Availability 와 Durability

가용성과 내구성 



- Topic 파라미터 - `unclean.leader.election.enable`

  - ISR 리스트에 없는 Replica를 Leader로 선출할 것인지에 대한 옵션 **(default : false)**

  - ISR 리스트에 Replica가 하나도 없으면 Leader 선출을 안 함 

  - 서비스 중단 ISR 리스트에 없는 Replica를 Leader로 선출함 ‒ 데이터 유실이 있을 수는 있음을 감안해야 함 

- Topic 파라미터  `min.insync.replicas` 

  - 최소 요구되는 ISR의 개수에 대한 옵션 **(default : 1)** 
    - 1은 Leader만 있으면 된다를 의미한다

  - ISR 이 min.insync.replicas 보다 적은 경우, Producer는 NotEnoughReplicas 예외를 수신

  - Producer에서 **acks=all**과 함께 사용할 때 더 강력한 보장 + **min.insync.replicas=2** 

  - n개의 Replica가 있고, min.insync.replicas=2 인 경우 n-2개의 장애를 허용할 수 있음
    - 3대이면 1대 장애만 허용가능하다

- 데이터 유실이 없게 하려면?
  - Topic : `replication.factor` 는 2 보다 커야 함(최소 3이상) 
  - Producer : acks 는 all 이어야 함 
  - Topic : `min.insync.replicas` 는 1 보다 커야 함(최소 2 이상)

- 데이터 유실이 다소 있더라도 가용성을 높게 하려면? 
  - Topic : `unclean.leader.election.enable` 를 true 로 설정





## Summary

가용성과 내구성 관련 파라미터

- replication.factor
- acks
- min.insync.replicas
- unclean.leader.election.enable
