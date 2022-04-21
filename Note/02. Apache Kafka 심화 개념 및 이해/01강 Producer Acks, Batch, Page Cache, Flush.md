> Producer는 Kafka 가 Message를 잘 받았는지 어떻게 알까?



## Producer Acks

producer parameter 중 하나 

- acks 설정은 요청이 성공할 때를 정의하는 데 사용되는 Producer에 설정하는 Parameter 
- **acks=0** : ack가 필요하지 않음. 이 수준은 자주 사용되지 않음. 메시지 손실이 다소 있더라도 빠르게 메시지를 보내야 하는 경우에 사용

- **acks=1** : (default 값) Leader가 메시지를 수신하면 ack를 보냄. Leader가 Producer에게 ACK를 보낸 후, Follower가 복제하기 전에 Leader에 장애가 발생하면 메시지가 손실. “**At  most once(최대 한 번)**” 전송을 보장 

- **acks=-1 : acks=all** 과 동일. 메시지가 Leader가 **모든 Replica까지 Commit 되면 ack를 보냄**.  Leader를 잃어도 데이터가 살아남을 수 있도록 보장. 그러나 대기 시간이 더 길고 특정 실패 사례에서 반복되는 데이터 발생 가능성 있음. “**At least once(최소 한 번)**” 전송을 보장

![image-20220421085246095](..\img\image-20220421085246095.png)

## Producer Retry

재전송을 위한 Parameters

- 재시도는 네트워크 또는 시스템의 일시적인 오류를 보완하기 위해 모든 환경에서 중요

![image-20220421085931489](..\img\image-20220421085931489.png)

- 보통  **retries를 조정하는 대신에 delivery.timeout.ms 조정으로 재시도 동작을 제어**
- acks=0에서 retry는 무의미 

## Producer Batch 처리 

메시지를 모아서 한번에 전송

- Batch 처리는 RPC(Remote Procedure Call)수를 줄여서 Broker가 처리하는 작업이 줄어들기 때문에 더 나은 처리량을 제공

![image-20220421090857379](..\img\image-20220421090857379.png)

- `linger.ms` 
  - 보통 배치사이즈를 정하지 않고 해당 파라미터로 세팅을 한다 
  - (default : 0, 즉시 보냄)
- `batch.size ` 
  - 보내기 전 Batch의 최대 크기
  - (default : 16 KB)

**Batch 처리의 일반적인 설정은 linger.ms=100 및 batch.size=1000000**



## Producer Delivery Timeout

send() 후 성공 또는 실패를 보고하는 시간의 상한 

![image-20220421091303595](..\img\image-20220421091303595.png)

- await send 여기 뭔지 설명 다시듣기 
