## KSQL
- KSQL에서 특정 토픽을 메시지를 모두 읽고자 할 때: KSQL CLI에서 auto.offset.reset을 earliest로 설정을 한다.
- KSQL은 ANSI SQL을 준수하지 않음
- KSQL은 KStream 기반인 Java 라이브러리
- GlobalKTable: 모든 스트림 애플리케이션에서 공유할 수 있는 KTable. 즉, 클러스터 내 모든 애플리케이션 인스턴스에서 동일한 데이터를 사용할 수 있음. 일반적으로 KTable은 파티션 단위로 처리되므로, 각각의 파티션마다 서로 다른 값이 저장됨. GlobalKTable은 모든 파티션에서 동일한 값을 가지며 스트리밍 애플리케이션에서 참조할 수 있는 공유 상태 데이터를 유지하고자 할 때 유용

## KStream
- KStream-KTable join의 결과는 KStream
- KStream에서 exactly_once가 보장되는 데이터 흐름은 kafka to kafka
- KStream에서 application.id는 consumer의 기본 group.id이기도 하며 모든 내부 topic의 접두사임
- KStreams는 스트리밍 애플리케이션, 특히 input kafka topic과 output kafka topic 간의 변환하는 애플리케이션을 구축하기 위한 라이브러리

## Consumer
- Consumer Group에서 Consumer 갯수는 Partition 갯수에 따라 결정(1:1이 베스트)
- RoundRobinAssignor - Consumer1,2 / Partition3 / Topic1,2 --> C1은 t1의 p0,p2와 t2의 p1을 할당 받고, C2는 t1의 p1과 t2의 p0,p2를 할당받음
- at-most once에서는 데이터 처리 전에 offset을 commit하는 것을 추천
- 특정 topic에 대해 메시지를 이미 읽었다면 offsets이 committed되어서 auto.offset.reset 옵션은 무시됨
- assign()은 특정 partition을 consumer에게 수동으로 할당 / subscribe()는 topic 이름을 지정하여 partition을 자동으로 할당(동시에 호출할 수 없음, 동일한 consumer에서 불가)
- real-time processing에서는 records-lag-max가 가장 중요함
- kafka-console-consumer CLI를 사용할 때 기본 옵션은 랜덤으로 group id를 사용
- consumer-group에서 repartition이 일어나는 경우 - consumer group에서 하나의 consumer 종료, consumer group에 새로운 consumer 추가, topic의 partition 증가
- producer가 acks=1 옵션을 사용하여 topic partition의 리더 broker에게 메시지를 전송함. 이 경우 팔로우 broker에 복제되지 않았음. 이 상황에서 어떤 조건으로 consumer는 메시지를 읽을 수 있을까? --> Consumer는 High Watermark의 값까지만 읽을 수 있음(acks=1의 경우, 최고 오프셋보다 작을 수 있음) 만약 conusmer가 High Watermark 이후의 오프셋까지 데이터를 읽으려고 하면, 해당 데이터가 복제되지 않았을 수도 있기 때문에 데이터 무결성에 문제가 발생할 수 있음
- consumer가 data를 polling하는 것을 멈추고 shutdown 시키고 싶으면 consumer.wakeUp()을 호출하고 catch a WakeUpException 하면 됨
- record batch를 처리하는데 약 6분이 걸리지만 consumer 실행 중 rebalance가 일어남 해결방법은? --> .poll() method를 호출하지 않은 경우에 consumer가 죽을 것으로 간주됨. max.poll.interval.ms(기본값300000)를 늘리면 해결 가능 (max.poll.interval.ms은 poll() 메소드 호출 사이의 최대 대기 시간을 지정)


## Producer
- max.in.flight.requests.per.connection을 증가시키면 메시지의 순서가 보장되지 않음
- producer가 매번 batch가 full되는 상황에서 throughput을 증가 시킬려면 batch.size를 증가시키고 compression을 가능케 하면 됨(압축을 활성화하면 보다 작은 배치를 만들 수 있음 / batch가 가득차 있으므로 Linger.ms(producer가 메시지를 보내기 전에 기다리는 최대 시간)는 의미가 없음)
- producer가 key가 없는 로그 메시지를 전송하는 경우 bootstrap.server, acks, key.serializer, value.serializer를 필수적으로 구성해야함
- 재시도 가능한 오류: not_enough_replicas, not_leader_for_partition
- producer의 batching chance를 증가시키려면 batch.size가 아닌 linger.ms가 중요함
- 클라이언트가 broker에 연결되면 전체 클러스터에 연결되고 leader가 변경되는 경우 클라이언트는 사용 가능한 브로커에 대한 메타데이터 요청을 자동으로 수행하여 topic에 대한 새로운 리더를 찾음. 따라서 파티션 리더 broker가 다운 되더라도

## Stateless & Stateful
- stateful: Aggregate, Count, Reduce, Joining
- stateless: Flatmap, Peek, Fliter, Foreach, GroupByKey, GroupBy, SelectKey
- stateless는 메세지의 처리가 메시지에 의존함을 의미. ex) JSON에서 Avro로 변환하거나 스트림을 필터링하는 행위

## Connector
- max.tasks parameter보다 테이블이 적으면 테이블 갯수만큼 task가 진행됨
- import data from external databases --> source connector / export data from Kafka to external databases --> sink connector
- Consumer는 __consumer_offsets topic에 직접 쓰지 않고 Group Coordinator Broker와 해당 topic을 관리하도록 선택된 브로커와 상호 작용함

## Schema Registry
- Schema Registry는 호환되지 않는 스키마 변경에 대한 보호 장치
- 클라이언트는 HTTPS, HTTP 인터페이스를 통해서 Schema Registry와 상호작용
- Schema Registry는 Avro 스키마를 저장하고 검색하기 위한 RESTful 인터페이스를 제공하는 별도의 애플리케이션
- Kafka Distribution을 사용할 때 Schema Registry는 별도의 JVM 구성 요소로서 존재함

## Broker
- leader와 replicas에 대해 데이터를 확실하게 전달하기 위해서는 asks=all, replication factor=3, min.insync.replicas=2가 좋음(min.insync.replicas는 topic에 대한 설정)
- replicas 3인 클러스터에서 하나의 브로커를 종료시키고 재시작하면 모든 데이터가 다른 리더 브로커에서 복제될 때까지 온라인 상태가 되지 않음
- kafka는 zero-copy를 통해 페이지 캐시에서 직접 데이터 전송, 메시지 변환 x 통해서 producer, consumer 사이에서 뛰어난 성능을 보임

## Partition
- 파티션 수를 늘리면 새 메시지 키가 다르게 해시됨. 동일한 키가 동일한 파티션으로 이동한다는 보장이 사라짐.

## Rest proxy
- producer에서 base64로 인코딩해서 특정 topic으로 데이터를 보내더라도, Rest proxy는 바이드로 변환한 다음 바이트 페이로드를 kafka로 보냄. 따라서 conumser는 binary 데이터를 받게 됨

## etc
- kubernetes로 관리하는 Docker Container에서 KStream 애플리케이션 재시작할 때 상태 복제 및 데이터 처리 프로세스 동작이 오래 걸림 --> 복구 속도를 높이려면 KStream 상태를 Persistent Volume에 저장하여 상태의 누락된 부분만 복구하면 좋다
- Avro는 name, namespace, fields는 필수요소, doc은 선택적 요소
- consumer는 KafkaAvroDeserializer를 사용하여 Avro 메시지를 역직렬화함. 메시지 스키마가 로컬 캐시에 없으면 Schema Registry에서 스키마를 가져옴. 그래도 없으면 예외 발생
- Avro

    - 쓰기 스키마는 부호화할 데이터에 사용할 스키마로서 producer가 사용. 읽기 스키마는 부호화된 데이터를 복호화할 때 consumer가 사용한다. 이 때, 쓰기/읽기 스키마는 동일한 버전이 아니어도 됨(중요). 동일하지 않은 스키마 버전일지라도 호환성을 유지할 수 있도록 Schema Evolution을 지원

    - Backward: 상위 버전의 스키마를 쓰는 컨슈머가 하위 버전의 스키마를 쓰는 프로듀서와 호환 
    - Forward: 상위 버전의 스키마로 전달한 프로듀서의 데이터를 하위 버전의 컨슈머가 효환
    - Full: 둘다 지원
- Unclean.leader.election.enable을 true로 하면 동기화되지 않은 복제본이 리더가 될 수 있음. 메시지가 손실되는 경우가 발생 가능
- key 사용: 동일한 키를 공유하는 메시지에 대해 강력한 순서나 그룹화가 필요한 경우 키가 필요. 동일한 키를 가진 메시지를 항상 올바른 순서로 표시해야 하는 경우 메시지에 키를 첨부하면 동일한 키를 가진 메시지가 항상  topic의 동일한 파티션으로 이동. kafka는 파티션 내의 순서를 보장하지만 topic의 파티션 간에는 순서를 보장하지 않으므로 키를 제공하지 않으면 파티션간 라운드 로빈 배포가 발생하므로 순서가 유지되지 않음
- log.retention.hours는 작은 것 기준으로 적용
- 특정 topic에 key K와 value null인 메시지를 보내면 broker는 정리시에 K key가 있는 모든 메시지를 삭제함
- schema가 없는 일반 텍스트의 경우 converter의 key.converter.schemas.enable 및 value.converter.schemas.enable을 false로 설정
- log.cleanup.policy=compact 를 하면 cleanup후 키당 하나의 메시지만 최신 값으로 유지. 압축된 모든 로그 오프셋은 consumer가 다음으로 가장 높은 오프셋을 가지므로 오프셋의 레코드가 압축된 경우에도 유효
- auto.create.topics.enable이 true로 설명되어 있으면 다음 요청에 자동으로 topic을 생성함
    - 클라이언트가 topic에 대한 메테데이터를 요청
    - consumer가 topic에서 메시지를 읽음
    - producer가 topic에 메시지를 보냄