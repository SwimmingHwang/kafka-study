우리 서버 설정 
log.segment.bytes
log segment 파일이 닫히게 되는 크기(바이트)
기본 값: 1GB
log.segment.ms
log segment 파일이 닫히게 되는 시간 값.
이 시간이 지나면 log segment 파일이 닫힘
브로커가 시작할 때 측정 기준이 되는 시간 값도 같이 시작됨
기본 값: 없음
log.retention.ms
카프카가 얼마동안 메시지를 보존할 지 설정하는 ms 값.
log.retention.hours와 log.retention.minutes 로 설정할 수 도 있으나, 여러 값이 지정된 경우 작은 시간단위 값을 기본으로 설정하기에, ms 값을 이용하는 것이 안전하다.
ex) log.retention.minutes = 1 이고 log.retention.ms = 30000 이면 30초로 설정된다
기본 값: 1주일(168 시간)
log.retention.bytes
저장된 메시지들의 전체 크기(바이트)를 기준으로 처리.
이 값은 모든 파티션에 적용됨.
ex) 5개의 파티션으로 구성된 토픽에 log.retention.bytes를 10MB 로 설정하면 해당 토픽에 보관되는 메시지의 전체 크기는 최대 50MB 가 됨.

```
############################# Log Retention Policy #############################

# The following configurations control the disposal of log segments. The policy can
# be set to delete segments after a period of time, or after a given size has accumulated.
# A segment will be deleted whenever *either* of these criteria are met. Deletion always happens
# from the end of the log.

# The minimum age of a log file to be eligible for deletion due to age
log.retention.hours=72

# A size-based retention policy for logs. Segments are pruned from the log unless the remaining
# segments drop below log.retention.bytes. Functions independently of log.retention.hours.
#log.retention.bytes=1073741824

# The maximum size of a log segment file. When this size is reached a new log segment will be created.
log.segment.bytes=1073741824

# The interval at which log segments are checked to see if they can be deleted according
# to the retention policies
log.retention.check.interval.ms=300000

############################# Zookeeper #############################
```

- 파일로 3일 남아있음. 
- 