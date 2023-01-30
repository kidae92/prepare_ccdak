## Transaction
- Transaction을 구현하기 위해, 몇 가지 새로운 개념들이 도입
    - Transaction Coordinator: Consumer Group Coordiantor와 비슷하게, 각 Producer에게는 Transaction Coordinator가 할당되며, PID 할당 및 Transaction 관리의 모든 로직을 수행
    - Transaction Log: 새로운 Internal Kafka Topic으로써, Consumer Offset Topic과 유사하게, 모든 Transaction의 영구적이고 복제된 Record를 저장하는 Transaction Coordinator의 상태 저장소
    - TransactionalId: Producer를 고유하게 식별하기 위해 사용되며, 동일한 TransactionId를 가진 Producer의 다른 인스턴스들은 이전 인스턴스에 의해 만들어진 모든 Transaction을 재개(또는 중단) 할 수 있음

## Transaction 관련 파라미터
- Broker Configs

![](./images/52.PNG)

- Producer Configs

![](./images/53.PNG)

- Consumer Configs

![](./images/54.PNG)