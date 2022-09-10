## 확장과 튜닝

cpu는 더이상 빨라지지 않지만, 컴퓨팅 비용은 감소
- 처리속도를 높이는 대신 단일 칩에서 더 많은 코어를 사용하거나 분산 시스템에서 더 많은 칩을 사용하여 컴퓨팅 선능을 얻을 수 있다.


1. 잡 프로파일링 : 최적화 관련 의사결정을 통해 성능향상
2. 잡 확장성 

## 잡 프로파일링


## 잡 확장하기

### 1. 다중 스레드 스텝
- (각 스텝 내) 청크마다 새 스레드를 생성해 각 청크를 병렬로 실행

#### ✔︎ 적용방법 (563p)
- TaskExecutor 추상화를 사용해 각 청크가 자체 스레드에서 실행되게 함
- ItemReader의 saveState=false로 설정 -> 잡을 재시작 할 수 없게함
  - read된 item에 여러 스레드가 접근 가능한 상태(동기화X), 재시작시 서로 다른 스레드로 덮어쓰기 발생가능성이 존재함

#### ✔︎ 사용케이스
- 입력 메커니즘이 네트워크, 티스크 버스와 같은 자원을 모두 소모 하는 경우 적절하지 않다.

### 2. 병렬 스텝
- 모든 스텝 or 플로우 를 병렬로 실행
- 보다 높은 처리량을 가짐

#### ✔︎ 적용방법 (566p)
- FlowBuilder의 split메서드 사용 
  - TaskExecutor 를 아규먼트로 받아 SplitBuilder를 반환
  - SplitBuilder를 통해 원하는 만큼의 플로우 객체 추가 가능
  - 각 플로우는 기본 TaskExecutor의 규약에 따라 자체 스레드에서 실행됨

✔︎ 잡의실행순서
- 잡의 실행순서는 동일함 (= 내부 스탭의 순서)
- 모든 플로우가 완료되어야 다음 스탭 실행

#### ✔︎ 사용케이스
- 스탭이 독립적인 경루 (A파일을 다읽지 않았다고, B파일 읽는걸 미룰 필요X)


### 2. AsyncItemProcessor & AsyncItemWriter
- 단일 JVM내 스레딩에만 의존
- 새 스레드에서 스텝의 ItemProcessor 부분만 실행

#### ✔︎ 적용방법 (572p)
- AsyncItemProcessor = 사용할 ItemProcessor구현체를 래핑하는 데코레이터이다. 
  - 어떤 아이템이 데코레이터에 전달될때, delegate(위임자)의 호출은 새 스레드에서 실행된다.
  - 이후 ItermProcessor의 실행 결과로 반환된 Future가 AsyncItemWriter로 전달 된다.

- AsyncItemWriter = 사용할 ItemWriter구현체를 래핑하는 데코레이터이다.
  - AsyncItemWriter는 Future를 처리해 그 결과를 위임 ItemWriter에 전달

✔︎ 주의점    
AsyncItemProcessor와 AsyncItemWriter를 함께사용하지 않으면,  
반환된 Future를 사용자가 직접 처리해 결과를 얻어내야 한다.    

chunk메서드의 반환타입도 Future -> <Transaction, Future<Transaction>>

✔︎ 의존성 추가   
spring-batch-integration

#### ✔︎ 사용케이스
ItemProcessor에서 병목이 존재하는 경우
