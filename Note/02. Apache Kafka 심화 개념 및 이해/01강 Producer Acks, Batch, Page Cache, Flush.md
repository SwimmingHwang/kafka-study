> Producer는 Kafka 가 Message를 잘 받았는지 어떻게 알까?



## Producer Acks

producer parameter 중 하나 

- acks 설정은 요청이 성공할 때를 정의하는 데 사용되는 Producer에 설정하는 Parameter 
- **acks=0** : ack가 필요하지 않음. 이 수준은 자주 사용되지 않음. 메시지 손실이 다소 있더라도 빠르게 메시지를 보내야 하는 경우에 사용

- **acks=1** : (default 값) Leader가 메시지를 수신하면 ack를 보냄. Leader가 Producer에게 ACK를 보낸 후, Follower가 복제하기 전에 Leader에 장애가 발생하면 메시지가 손실. “**At  most once(최대 한 번)**” 전송을 보장 

- **acks=-1 : acks=all** 과 동일. 메시지가 Leader가 **모든 Replica까지 Commit 되면 ack를 보냄**.  Leader를 잃어도 데이터가 살아남을 수 있도록 보장. 그러나 대기 시간이 더 길고 특정 실패 사례에서 반복되는 데이터 발생 가능성 있음. “**At least once(최소 한 번)**” 전송을 보장

![image-20220421085246095](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421085246095.png)

## Producer Retry

재전송을 위한 Parameters

- 재시도는 네트워크 또는 시스템의 일시적인 오류를 보완하기 위해 모든 환경에서 중요

![image-20220421085931489](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421085931489.png)

- 보통  **retries를 조정하는 대신에 delivery.timeout.ms 조정으로 재시도 동작을 제어**
- acks=0에서 retry는 무의미 

## Producer Batch 처리 

메시지를 모아서 한번에 전송

- Batch 처리는 RPC(Remote Procedure Call)수를 줄여서 Broker가 처리하는 작업이 줄어들기 때문에 더 나은 처리량을 제공

![image-20220421090857379](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421090857379.png)

- `linger.ms` 
  - 보통 배치사이즈를 정하지 않고 해당 파라미터로 세팅을 한다 
  - (default : 0, 즉시 보냄)
- `batch.size ` 
  - 보내기 전 Batch의 최대 크기
  - (default : 16 KB)

**Batch 처리의 일반적인 설정은 linger.ms=100 및 batch.size=1000000**



## Producer Delivery Timeout

send() 후 성공 또는 실패를 보고하는 시간의 상한 

![image-20220421091303595](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220421091303595.png)

- batch : 데이터를 배치처리에서 일괄처리 데이터를 쌓는 시간 
- await send : 쏘는 시간  
- retry.backoff.ms : retry 하고 쉬고 retry하는.. 시간 



## Message Send 순서 보장 

진행중(in-flight) 인 여러 요청(request)을 재시도하면 순서가 변경될 수 있음

- max.in.flight.requests.per.connection = 동시에 날아갈 수 있는 요청 수 (네트워크에서 동시에 날아갈 수 있음) 

![image-20220425085257393](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220425085257393.png)

-> 방지하려면 enable.idempotence = true 로 설정하면 순서 보장이 된다  
즉, 하나의 배치가 실패하면 **같은 파티션으로** 들어오는 후속 배치들도 실패된다 (OutofOrderSequenceException)



## Page Cache 와 Flush

- 메시지는 파티션에 기록된다 
- 파티션은 로그 세그먼트 파일로 구성이 된다 (default : 1GB마다 새로운 Segement생성)
- 브로커는 성능을 위해 Log Segment는 OS Page Cache에 기록이 된다 
  - 이후에 디스크에 write 된다 
- 브로커가 받은 메시지 형태와 디스크에 저장되는 포맷과 producer가 보내준 데이터의 포맷이 완벽하게 동일하다 
  - 브로커는 데이터 byte array를 가공하지 않는다 
- 로그 파일에 저장된 메시지의 데이터 형식은 Broker가 Producer로부터 수신한 것, 그리고 Consumer에게 보내는 것과 정확히 동일하므로, Zero-Copy가 가능
  - 네트워크를 통해 들어와서 os page 캐시에 저장할 때 heap 영역(kafka 가 java라서) 에 복사를 하고 os page 캐시에 저장하는게 아니라 네트워크를 통해 들어와서 os page 캐시에 바로 저장하는 것을 zero-copy라고 함. 그래서 heap 메모리랑 cpu가 많이 필요하지 않다 
  - Zero-copy 전송은 데이터가, User Space에 복사되지 않고, CPU 개입 없이 Page Cache와 Network Buffer 사이에서 직접 전송되는 것을 의미. 이것을 통해 Broker Heap 메모리를 절약하고 또한 엄청난 처리량을 제공

- Page Cache는 다음과 같은 경우 디스크로 Flush됨 
  - Broker가 완전히 종료 ◦ OS background “Flusher Thread” 실행
    - 자동으로 os 레벨에서 disk로 내리는 작업



### Flush 되기 전에 Broker장애가 발생하면?

- OS가 데이터를 디스크로 Flush하기 전에 Broker의 시스템에 장애가 발생하면 해당 데이터가 손실됨 
- Partition이 Replication(복제)되어 있다면, Broker가 다시 온라인 상태가 되면 필요시 Leader Replica(복제본)에서 데이터가 복구됨
- **Replication이 없다면, 데이터는 영구적으로 손실될 수 있음**



### Kafka 자체 Flush정책 

많은 옵션이 있음

- 마지막 Flush 이후의 메시지 수(log.flush.interval.messages) 또는 시간(log.flush.interval.ms)으로 Flush(fsync)를 트리거하도록 설정할 수 있음 
- **Kafka는 운영 체제의 background Flush 기능(예: pdflush)을 더 효율적으로 허용하는 것을 선호하기 때문에** 이러한 설정은 **기본적으로 무한(기본적으로 fsync 비활성화)으로 설정** 
- 이러한 설정을 **기본값으로 유지하는 것을 권장** 
- *.log 파일을 보면 디스크로 Flush된 데이터와 아직 Flush되지 않은 Page Cache (OS  Buffer)에 있는 데이터가 모두 표시됨 
- Flush된 항목과 Flush되지 않은 항목을 표시하는 Linux 도구(예: vmtouch)도 있음



# Summary 

keyword : Acks, Batch, Idempotence, Page Cache

- Producer Acks : 0, 1, all(-1) 
- Batch 처리를 위한 옵션 : linger.ms, batch.size 
- 메시지 순서를 보장하려면 Producer에서 enable.idempotence를 true로 설정 
- 성능을 위해 Log Segment는 OS Page Cache에 기록됨
