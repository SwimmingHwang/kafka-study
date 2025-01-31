# Producer

- Apache Kafka 주요 요소
  - Producer : 메시지를 생산해서 KAfka Topic으로 메시지를 보내는 애플리케이션
  - Consumer : Topic 의 메시지를 가져와서 소비하는 애플리케이션
  - Consumer Group : Topic의 메시지를 사용하기 위해 협력하는 Consumer들의 집합
  - 하나의 컨슈머는 하나의 컨슈머 그룹에 포함되며 
    컨슈머 그룹 내의 컨슈머들은 협력하여 topic의 메시지를 병렬처리 한다 



#### Producr 와 Consumer의 기본 동작 방식

- Producer와 Consumer은 서로 알지 못함.
- Producer와 Consumer는 각각 고유의 속도로 Commit Log에 Write Read를 수행
- 다른 컨슈머 그룹에 속한 컨슈머 들은 서로 관련이 없다 
- Commit Log에 있는 event(message)를 동시에 다른 위치에서 read 할 수 있다 





## Record(Message) 구조

- Message == Record == Event == Data 

![image-20220415095439365](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415095439365.png)

- Key와 Value는 Avro, JSO 등 다양한 형태가 가능함 



## Serializer/Deserializer

- Kafka는 Record(데이터)를 Byte Array로 저장  

![image-20220415095703863](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415095703863.png)



### Producer Sample Code 

- Key와 Value용 Serializer 를 각각 설정할 수 있다

```java
private Properties props = new Properties();

props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "broker101:9092,broker102:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, org.apache.kafka.common.serialization.StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, io.confluent.kafka.serializersKafkaAvroSerializer.class);

KafkaProducer producer = new KafkaProducer(props);
```



## Producing to Kafka 

- 프로듀서 애플리케이션에서 이뤄지는 다양한 활동 
- 아키텍처 

![image-20220415100326036](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415100326036.png)



### Partitioner 의 역할

- 메시지를 Topic의 어떤 파티션으로 보낼지 결정
- 파티셔너가 다양한 key 도형 : key 구분
- Key가 null이 아니다 라는 전제조건하에 실행됨

![image-20220415100936236](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415100936236.png)







- 만약 키가 Null 이면 Kafka2.4이전의 DefaultPartitioner는 라운드로빈 정책으로 동작
  - 2.4 이후에는 Sticky정책으로 동작 하나의 Batch가 닫힐 때 까지 하나의 파티션에게 record를 보내고 랜덤으로 파티션을 선택한다. 
  - 이는 Batch 배치 처리 로 3개씩 묵어서 두번만 날리면 된다. 효과적임


![image-20220415101245766](https://raw.githubusercontent.com/SwimmingHwang/kafka-study/main/Note/img/image-20220415101245766.png)

- 파티셔너는 자체 개발도 가능하다 





## Summary

- Message구조 

  - Header와 Key,Value로 구성한다

- Serializer/Deserializer

  - Record를 Byte Array로 저장한다 
  - Producer는 Serializer, Consumer는 Deserializer사용

- Producer는 Message의 Key 존재 여부에 따라 파티셔너를 통한 메시지 처리 방식이 다르다

- Partitioner 메시지를 토픽의 어떤 파티션으로 보낼지 결정한다

  
