## Broker 설치 구성

- Broker는 분리된 각각의 전용 서버에 분리하여 설치 구성하는 것을 권장
    - N개의 Broker가 있으면, Replication Factor는 최대 N까지 사용하여 Topic 생성 가능
    - Mission Critical Topic에는 Replication Factor는 보통 3을 많이 사용
        - 3개의 Broker로 구성하고 하나의 Broker가 장애 상태시, RF 3인 Topic 생성 불가능
        - 따라서, Broker는 4개의 이상 하나의 Cluster로 구성하는 것을 권장
    - 데이터 유실 방지를 위해서, min.insync.replicas는 2를 많이 사용
        - 3개의 Broker로 구성하고 1개의 Broker가 장애 상태시, Topic에 Write는 가능
        - 1개의 Broker가 추가로 장애시, 데이터 유실의 가능성이 높아짐
        - 따라서, Broker는 4개 이상 하나의 Cluster로 구성하는 것을 권장


## Broker CPU
- Broker는 CPU를 많이 사용하지는 않으나, 처리량에 따라서 Thread 파라미터 튜닝이 필요하며, Thread 증가에 따라 CPU 사용량이 증가
    - 설정을 잘 못 하거나, 메모리가 충분하지 않거나, Bug로 인해 CPU 사용률이 높을 수도 있음
    - Reference Architecture에는 Dual 12-core Sockets를 권장
    - Broker 파라미터
        - num.io.threads(기본값: 8): Disk 개수보다 크게 설정
        - num.network.threads(기본값: 3): TLS를 사용할 경우 두 배 이상으로 설정
        - num.recovery.threads.per.dat.dir(기본값: 1): Broker 시작시 빠른 기동을 위해서, core수까지만 설정
        - num.replica.fetchers(기본값: 1): Source Broker에서 메시지를 복제하는데 사용되는 Thread 수. 빠르게 복제하기 위해 값을 증가(Latency를 만족하는). Broker의 CPU 사용률과 네트워크 사용률이 올라감
        - num.cleaner.threads(기본값: 1): Disk 개수 혹은 core 개수까지만 설정

## Open File Descriptors

- Broker는 많은 수의 Partition을 지원하므로 상대적으로 소규모 배포에서도 Open File Handle 수가 기본값을 초과할 수 있음
    - 최소 ulimit -n 100000

## Broker Memory
- Broker의 JVM Heap + OS Page Cache
- Broker는 JVM Heap을 많이 사용하지 않음
    - Broker의 Heap 메모리는 운영환경인 경우 대부분 6GB 까지 할당
    - 매우 큰 Cluster 혹은 매우 많은 Partition이 필요한 경우 12GB 이상 사용
    - Broker의 OS만을 위해선 보통 1GB 정도 할당
    - Broker는 OS Page Cache를 많이 사용
        - OS Page Cache를 통해서 Zero Copy 전송을 수행
        - 많은수록 성능에 유리
    - 운영환경용 Broker 메모리는 최소 32GB 이상 권장하며, 처리량에 따라서 64GB 이상 사용 권장

## Broker Java Heap Memory

- kafka-server-start스크립트에Java Heap 설정하는옵션이있음
    - $KAFKA_HEAP_OPTS
-Xms6g ‒Xmx6g
    - $KAFKA_JVM_PERFORMANCE_OPTS
-server -XX:MetaspaceSize=96m -XX:MinMetaspaceFreeRatio=50
-XX:MaxMetaspaceFreeRatio=80 -XX:+UseG1GC -XX:MaxGCPauseMillis=20
-XX:InitiatingHeapOccupancyPercent=35 -XX:G1HeapRegionSize=16M

## Broker Network
- 전체 처리메시지(복제 포함)가 네트워크를 통해 전달
- 처리량이 작은 Application일 경우, Broker의 NW는 1Gb로 충분
- 처리량이 높은 Application일 경우, Broker의 NW는 10Gb 이상 필요
- Producer에서 압축 옵션을 사용하면 네트워크를 보다 효율적으로 사용 가능
- Internal과 External 트래픽 간 분리 가능
    - KIP-103: Separation of Internal and External traffic
https://cwiki.apache.org/confluence/display/KAFKA/KIP-103%3A+Separation+of+Internal+and+External+traffic

## Broker Disk
- Kafka 성능에 큰 영향
- Kafka Broker의 data log용 Disk는 OS 용 Disk와 분리 권장
    - Broker의 data log용으로 여러 개의 Local Disk 사용을 권장(RAID10 권장, JBOD 사용 가능)
    - SSD Disk를 권장
    - XFS 파일시스템을 사용해야 함
    - mount시에 noatime 옵션 사용 - Linux가 각 파일에 마지막으로 엑시스한 시간을 기록하는 파일 시스템 메타데이터를 유지 관리하는 방식을 off
    - Broker의 파라미터 중 log.dirs에 콤마(,)로 구분한 디렉토리들로 정의
    - 하나의 Partition은 하나의 volume에서 생성됨
    - Partition들은 log.dirs의 디렉토리에 round-robin 방식으로 분배
    - NAS 사용 불가

## Virtual Memory
- Memory swapping 최소화
    - vm.swappiness=1 (기본값: 60)
- Blocking Flush (synchronous)빈도 감소
    - vm.dirty_ratio=80 (기본값: 20)
- Non-Blocking background flushes (asynchronous) 빈도 증가
    - vm.dirty_background_ratio=5 (기본값: 10)
- 이 파라미터들은 /etc/sysctl.conf에 설정하고 sysctl -p 명령어로 Load함

## Zookeeper 권장 사양
- 홀수로 구성
- 최소 2Core - 권장 4Core
- Memory
    - 8 GB
- Transaction log (dataLogDir)
    - 512 GB SSD
- Database snapshots (dataDir)
    - 2 TB SSD RAID 10

## Zookeeper Disk
- Auto Purge 옵션
- ZooKeeper의 server.properties내의 purge snapshots 파라미터 설정

    - autopurge.snapRetainCount : 보존할SnapShot개수(권장3)
    - autopurge.purgeInterval : Purge Interval(권장24)
    - 위의 권장 옵션은 24시간마다 3 개를 제외한 모든 스냅샷을 제거하는 설정
    - 미션 크리티컬 시스템의 경우에는 3-5개 Zookeeper 노드에 추가 스토리지를 사용하여 보존 할 SnapShot 수를 조정하는 경우도 있음