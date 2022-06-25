### 상태관리

- 배치잡 실행 중에 오류가 발생하면 어떻게 복구할 수 있을까?
- 실행 중 오류가 발생했을 때 어떤 처리를 하고 있었는지, 잡이 다시 시작되면 어떻게 되는지 알 수 있을까?

스프링배치는 잡이 실행될 때 잡의 상태를 JobRepository에 저장해 관리한다.

그리고 잡의 재시작 또는 아이템 재처리 시 어떤 동작을 수행할지 이 정보를 사용해 결정한다.

### 모니터링

잡이 처리되는 데 걸리는 시간이나 오류로 인해 재시도된 아이템 수와 가은 배치 실행 추세뿐만 아니라, 잡이 어디까지 실행됐는지를 파악할 수 있는 기능을 스프링 배치가 관련 수치 값을 취합해서 제공해준다.

### JobRepository

스프링 배치에서 JobRepository란

- JobRepository Interface
- JobRepository 인터페이스를 구현해 데이터를 저장하는 데 사용되는 저장소

![meta-data-erd.png](https://github.com/lets-study-hard/spring_batch/blob/main/chapter5/images/meta-data-erd.png)

- 6개의 테이블이 존재
- 스키마의 시작점은 BATCH_JOB_INSTANCE

**BATCH_JOB_INSTANCE**

잡을 식별하는 고유 정보가 포함된 잡 파라미터로 잡을 처음 실행하면 단일 JobInstance 레코드가 테이블에 저장된다.

| JOB_EXECUTION_ID | 테이블의 기본 키 |
| --- | --- |
| VERSION | 낙관적인 락(optimistic locking)에 사용되는 레코드 버전 |
| JOB_NAME | 실행된 잡의 이름 |
| JOB_KEY | 잡 이름과 잡 파라미터의 해시 값으로, JobInstnace를 고유하게 식별하는데 사용되는 값 |

**BATCH_JOB_EXECUTION**

배치 잡의 실제 실행 기록을 나타낸다. 잡이 실행될 때마다 새 레코드가 해당 테이블에 생성되고, 잡이 진행되는 동안 주기적으로 업데이트 된다.

| JOB_EXECUTION_ID | 테이블의 기본 키 |
| --- | --- |
| VERSION | 낙관적인 락에 사용되는 레코드 버전 |
| JOB_INSTANCE_ID | BATCH_JOB_INSTANCE 테이블을 참조하는 외래 키 |
| CREATE_TIME | 레코드가 생성된 시간 |
| START_TIME | 잡 실행이 시작된 시간 |
| END_TIME | 잡 실행이 완료된 시간 |
| STATUS | 잡 실행의 배치 상태 |
| EXIT_CODE | 잡 실행의 종료 코드 |
| EXIT_MESSAGE | EXIT_CODE와 관련된 메시지나 스택 트레이스 |
| LAST_UPDATED | 레코드가 마지막으로 갱신된 시간 |

BATCH_JOB_EXECUTION 테이블과 연관이 있는 세 개의 테이블이 존재한다.

**BATCH_JOB_EXECUTION_CONTEXT**

재시작처럼 배치가 여러 번 실행되는 상황에서 관련 정보를 스프링 배치가 잘 보관해야하는데 해당 테이블이 JobExecution의 ExecutionContext를 저장하는 곳이다.

| JOB_EXECUTION_ID | 테이블의 기본 키 |
| --- | --- |
| SHORT_CONTEXT | 트림 처리된 SERIALIZED_CONTEXT |
| SERIALIZED_CONTEXT | 직렬화된 ExecutionContext |

**BATCH_JOB_EXECUTION_PARAMS**

잡이 매번 실행될 때마다 사용된 잡 파라미터를 저장한다.

| JOB_EXECUTION_ID | 테이블의 기본 키 |
| --- | --- |
| TYPE_CODE | 파라미터 값의 타입을 나타내는 문자열 |
| KEY_NAME | 파라미터의 이름 |
| STRING_VAL | 타입의 String인 경우 파라미터의 값 |
| DATE_VAL | 타입의 Date인 경우 파라미터의 값 |
| LONG_VAL | 타입이 Long인 경우 파라미터의 값 |
| DOUBLE_VAL | 타입의 Double인 경우 파라미터의 값 |
| IDENTIFYING | 파라미터가 식별되는지 여부를 나타내는 플래그 |

**BATCH_STEP_EXECUTION**

| JOB_EXECUTION_ID | 테이블의 기본 키 |
| --- | --- |
| VERSION | 낙관적인 락에 사용되는 레코드 버전 |
| STEP_NAME | 스텝의 이름 |
| JOB_INSTANCE_ID | BATCH_JOB_INSTANCE 테이블을 참조하는 외래 키 |
| START_TIME | 스텝 실행이 시작된 시간 |
| END_TIME | 스텝 실행이 완료된 시간 |
| STATUS | 스텝의 배치 상태 |
| COMMIT_COUNT | 스텝 실행 중에 커밋된 트랜잭션 수 |
| READ_COUNT | 읽은 아이템 수 |
| FILTER_COUNT | 아이템 프로세서가 null을 반환해 필터링된 아이템 수 |
| WRITE_COUNT | 기록된 아이템 수 |
| READ_SKIP_COUNT | ItemReader 내에서 예외가 발생해 건너뛴 아이템 수 |
| PROCESS_SKIP_COUNT | itemProcessor 내에서 예외가 발생해 건너뛴 아이템 수 |
| WRITE_SKIP_COUNT | itemWriter 내에서 예외가 발생해 건너뛴 아이템 수 |
| ROLLBACK_COUNT | 스텝 실행에서 롤백된 트랜잭션 수 |
| EXIT_CODE | 스텝의 종료 코드 |
| EXIT_MESSAGE | 스텝 실행에서 반환된 메시지나 스택 트레이스 |
| LAST_UPDATED | 레코드가 마지막으로 업데이트된 시간 |

**BATCH_STEP_EXECUTION_CONTEXT**

| JOB_EXECUTION_ID | 테이블의 기본 키 |
| --- | --- |
| SHORT_CONTEXT | 트림 처리된 SERIALIZED_CONTEXT |
| SERIALIZED_CONTEXT | 직렬화된 ExecutionContext |

ExecutionContext가 JobExecution내에 존재하는 것처럼, StepExecution에서도 동일한 목적으로 사용되는 ExecutionContext가 존재한다.

StepExecution의 ExecutionContext는 스텝 수준에서 컴포넌트 상태를 저장하는데 사용된다.

### In-Memory JobRepository

스프링 배치는 [java.util.Map](http://java.util.Map) 을 데이터 저장소로 사용하는 JobRepository 구현체를 제공한다.

### Batch Infrastructor

- @EnableBatchProcessing 어노테이션을 사용하면 추가적인 구성 없이 스프링 배치가 제공하는 JobRepository를 사용할 수 있다.
- JobRepository를 커스터마이징해야할 때 BatchConfigurer 인터페이스를 사용해 Repository를 비롯한 모든 스프링 배치 인프라스트럭처를 커스터마이징할 수 있다.

### BatchConfigurer

BatchConfigurer는 스프링 배치 인프라스트럭처 컴포넌트의 구성을 커스터마이징하는 데 사용되는 전략 인터페이스이다.

@EnableBatchProcessing 어노테이션을 적용하면 스프링 배치는 BatchConfigurer 인터페이스를 사용해 프레임워크에서 사용되는 각 인프라스트럭처 컴포넌트의 인스턴스를 얻는다.

1. BatchConfigurer 구현체에서 빈을 생성한다.
2. SimpleBatchConfiguration에서 스프링의 ApplicationContext에 생성한 빈을 등록한다.

대부분 SimpleBatchConfiguration을 직접 건드릴 필요가 없다.

```java
public interface BatchConfigurer {
	JobRepository getJobRepository() throws Exception;
	PlatformTransactionManager getTransactionManager() throws Exception;
	JobLauncher getJobLauncher() throws Exception;
	JobExplorer getJobExplorer() throws Exception;
}
```

- 각 메서드는 스프링 배치 인프라스트럭처의 주요 컴포넌트를 제공한다.
- PlatformTransactionManager는 프레임워크가 제공하는 모든 트랜잭션 관리 시에 스프링 배치가 사용하는 컴포넌트다.
- JobExplorer는 JobRepository의 데이터를 읽기전용으로 볼 수 있는 기능을 제공한다.

대부분 이 모든 인터페이스를 직접 구현할 필요는 없으며, 스프링 배치가 제공하는 DefaultBatchConfigurer를 사용하면 앞서 언급했던 컴포넌트에 대한 모든 기본 옵션이 제공된다. → DefaultBatchConfigurer를 상속해 필요한 메소드만 재정의하는것이 간편

### JobRepository Customizing

JobRepository는 JobRepositoryFactoryBean이라는 FactoryBean을 통해 생성된다.

이 FactoryBean은 각 attribute를 커스터마이징할 수 있는 기능을 제공한다.
