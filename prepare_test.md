## KSQL
- KSQL에서 특정 토픽을 메시지를 모두 읽고자 할 때: KSQL CLI에서 auto.offset.reset을 earliest로 설정을 한다.
- KSQL은 ANSI SQL을 준수하지 않음

## KStream
-  KStream-KTable join의 결과는 KStream
-  KStream에서 exactly_once가 보장되는 데이터 흐름은 kafka to kafka 

## Consumer
- Consumer Group에서 Consumer 갯수는 Partition 갯수에 따라 결정(1:1이 베스트)
- RoundRobinAssignor - Consumer1,2 / Partition3 / Topic1,2 --> C1은 t1의 p0,p2와 t2의 p1을 할당 받고, C2는 t1의 p1과 t2의 p0,p2를 할당받음
- at-most once에서는 데이터 처리 전에 offset을 commit하는 것을 추천
- 특정 topic에 대해 메시지를 이미 읽었다면 offsets이 committed되어서 auto.offset.reset 옵션은 무시됨

## Producer
- max.in.flight.requests.per.connection을 증가시키면 메시지의 순서가 보장되지 않음
- producer가 매번 batch가 full되는 상황에서 throughput을 증가 시킬려면 batch.size를 증가시키고 compression을 가능케 하면 됨(압축을 활성화하면 보다 작은 배치를 만들 수 있음 / batch가 가득차 있으므로 Linger.ms(producer가 메시지를 보내기 전에 기다리는 최대 시간)는 의미가 없음)
- producer가 key가 없는 로그 메시지를 전송하는 경우 bootstrap.server, acks, key.serializer, value.serializer를 필수적으로 구성해야함
- 재시도 가능한 오류: not_enough_replicas, not_leader_for_partition
- producer의 batching chance를 증가시키려면 batch.size가 아닌 linger.ms가 중요함

## Stateless & Stateful
- stateful: Aggregate, Count, Reduce, Joining
- stateless: Flatmap, Peek, Fliter, Foreach, GroupByKey, GroupBy, SelectKey
- stateless는 메세지의 처리가 메시지에 의존함을 의미. ex) JSON에서 Avro로 변환하거나 스트림을 필터링하는 행위

## Connector
- max.tasks parameter보다 테이블이 적으면 테이블 갯수만큼 task가 진행됨
- import data from external databases --> source connector / export data from Kafka to external databases --> sink connector

## Schema Registry
- Schema Registry는 호환되지 않는 스키마 변경에 대한 보호 장치
- 클라이언트는 HTTPS, HTTP 인터페이스를 통해서 Schema Registry와 상호작용

## Broker
- leader와 replicas에 대해 데이터를 확실하게 전달하기 위해서는 asks=all, replication factor=3, min.insync.replicas=2가 좋음(min.insync.replicas는 topic에 대한 설정)

- replicas 3인 클러스터에서 하나의 브로커를 종료시키고 재시작하면 모든 데이터가 다른 리더 브로커에서 복제될 때까지 온라인 상태가 되지 않음

- kafka는 zero-copy를 통해 페이지 캐시에서 직접 데이터 전송, 메시지 변환 x 통해서 producer, consumer 사이에서 뛰어난 성능을 보임

## Partition
- assign()은 특정 partition을 consumer에게 수동으로 할당 / subscribe()는 topic 이름을 지정하여 partition을 자동으로 할당

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