# Batch

### 스텝

- 잡이 전체적인 처리를 정의한다면 스텝은 잡의 구성 요소를 담당
- 독립적이고 순차적으로 배치 처리를 수행(배치 프로세서) 모든 단위 작업의 조각
- 트랜잭션은 스텝 내에서 이루어지며 스텝은 서로 독립되도록 설계

### 유형

- Tasklet
    - Tasklet.execute 메서드가 RepeatStatus.FINISHED를 반환할때까지 반복적으로 실행
- Chunk
    - 2~3개의 주요 컴포넌트(ItemReader, ItemProcessor, ItemWriter)로 구성
    - 레코드를 그룹 단위로 처리
    - 각 청크는 자체 트랜잭션으로 처리되며 처리에 실패했다면 마지막으로 성공한 트랜잭션 이후부터 다시 시작가능
    

### Tasklet

- Tasklet Interface execute 메소드 구현
- MethodInvokingTaskletAdapter를 사용해 사용자의 코드를 스텝으로 정의 가능(POJO)
    
    

### Tasklet Interface

- RepeatStatus.CONTINUABLE → 해당 taklset을 다시 실행
    - Spring Batch에서 해당 tasklet의 실행 횟수 트랜잭션 등을 추적
- RepeatStatus.FINISHED → 해당 tasklet의 처리를 종료하고 다음처리를 실행

### 사용자 정의

- CallableTaskletAdapter
    - 새로운 스레드에서 실행
    - 값의 반환이 가능하고 CheckedException을 바깥으로 던질 수 있다
    
    ```java
    @EnableBatchProcessing
    @RequiredArgsConstructor
    @Configuration
    public class BatchConfiguration {
        private final JobBuilderFactory jobBuilderFactory;
        private final StepBuilderFactory stepBuilderFactory;
    
        @Bean
        public Job callableJob() {
            return this.jobBuilderFactory.get("callableJob")
                .start(callableStep())
                .build();
        }
    
        @Bean
        public Step callableStep() {
            return this.stepBuilderFactory.get("callableStep")
                .tasklet(tasklet())
                .build();
        }
    
        @Bean
        publicCallable<RepeatStatus> callableObject() {
            return () -> {
                System.out.println("This was executed in another Thread");
                returnRepeatStatus.FINISHED;
            };
        }
    
        @Bean
        public CallableTaskletAdapter tasklet() {
            final CallableTaskletAdapter callableTaskletAdapter = new CallableTaskletAdapter();
            callableTaskletAdapter.setCallable(callableObject());
    
            return callableTaskletAdapter;
        }
    }
    ```
    
    - 별개의 스레드에서 실행되지만 스텝과 병렬로 실행되는 것은 아님
    - RepeatStatus 객체를 반환하기 전에는 완료된 것으로 간주하지 않기 때문에 플로우 내의 다른 스텝은 실행되지 않는다.
- MethodInvokingTaskletAdapter
    - 이미 존재하는 다른 객체의 메소드를 호출하고 싶을 때 사용
    
    ```java
    @EnableBatchProcessing
    @RequiredArgsConstructor
    @Configuration
    public class BatchConfiguration {
        private final JobBuilderFactory jobBuilderFactory;
        private final StepBuilderFactory stepBuilderFactory;
    
        @Bean
        public Job job() {
            return this.jobBuilderFactory.get("job")
                .start(step())
                .build();
        }
    
        @Bean
        public Step step() {
            return this.stepBuilderFactory.get("step")
                .tasklet(tasklet())
                .build();
        }
    
        @Bean
        public MethodInvokingTaskletAdapter tasklet() {
            final MethodInvokingTaskletAdapter taskletAdapter = new MethodInvokingTaskletAdapter();
            taskletAdapter.setTargetObject(service());
            taskletAdapter.setTargetMethod("printHello");
    
            return taskletAdapter;
        }
    
        @Bean Service service() {
            return new Service();
        }
    }
    ```
    
- SystemCommandTasklet
    - 시스템 명령을 실행할때 사용하며 지정한 시스템 명령은 비동기로 실행된다.
    
    ```java
    @EnableBatchProcessing
    @RequiredArgsConstructor
    @Configuration
    public class BatchConfiguration {
        private final JobBuilderFactory jobBuilderFactory;
        private final StepBuilderFactory stepBuilderFactory;
    
        @Bean
        public Job Job() {
            return this.jobBuilderFactory.get("job")
                .start(step())
                .build();
        }
    
        @Bean
        public Step step() {
            return this.stepBuilderFactory.get("step")
                .tasklet(tasklet())
                .build();
        }
    
        @Bean
        public SystemCommandTasklet tasklet() {
            final SystemCommandTasklet systemCommandTasklet = new SystemCommandTasklet();
            systemCommandTasklet.setCommand("rm -rf /temp.txt");
            systemCommandTasklet.setTimeout(5000);
            systemCommandTasklet.setInterruptOnCancel(true);
    
            return systemCommandTasklet;
        }
    }
    ```
    
    - setInterruptOnCancel → 잡이 비정상적으로 종료될 때 시스템 프로세스와 관련된 스레드 종료 여부
    - workingDirectory → 작업하고자하는 디렉토리
    - systemProcessExitCodeMapper → 시스템 반환 코드를 스프링 배치 상태값으로 매핑하는 구현체 적용
    - terminateCheckInterval → 시스템 명령은 비동기 방식으로 실행되므로, 명령 실행 이후에 해당 명령의 완료 여부를 주기적으로 확인 default는 1초

### 청크 기반 스텝

```java
@EnableBatchProcessing
@RequiredArgsConstructor
@Configuration
public class BatchConfiguration {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;

    @Bean
    public Job job() {
        return this.jobBuilderFactory.get("job")
            .start(step())
            .build();
    }

    @Bean
    public Step step() {
        return this.stepBuilderFactory.get("step")
            .<String, String>chunk(10)
            .reader(itemReader(null))
            .writer(itemWriter(null))
            .build();
    }

    @Bean
    @StepScope
    public FlatFileItemReader<String> itemReader(
        @Value("#{jobParameters['inputFile']}")ResourceinputFile
) {
        return new FlatFileItemReaderBuilder<String>()
            .name("itemReader")
            .resource(inputFile)
            .lineMapper(new PassThroughLineMapper())
            .build();
    }

    @Bean
    @StepScope
    public FlatFileItemWriter<String> itemWriter(
        @Value("#{jobParameters['outputFile']}")ResourceoutputFile
) {
        return new FlatFileItemWriterBuilder<String>()
            .name("itemWriter")
            .resource(outputFile)
            .lineAggregator(new PassThroughLineAggregator<>())
            .build();
    }
}
```

![스크린샷 2022-06-18 오전 11.10.45.png](https://github.com/lets-study-hard/spring_batch/blob/main/chapter4/images/chunk.png)

### 청크 크기 구성

- 정적 개수 설정
- CompletionPolicy
    - CompletionPolicy 인터페이스의 구현체를 통해 청크의 완료 여부를 결정할 수 있는 결정 로직을 구현할 수 있도록 해준다.
    - SimpleCompletionPolicy → 처리된 아이템의 개수를 통해 결정
    - TimeoutTerminationCompletionPolicy →  청크 내 처리시간이 해당 시간을 넘을 때 정상처리 간주
    - CompositCompletionPolicy → 여러 정책 중 하나라도 완료라고 판단되면 정상처리 간주
    

### 스텝 리스너

- 개별 스텝, 청크의 시작과 끝에서 특정 로직을 처리할 수 있도록 해준다.
- StepExecutionListner, ChunkListner
- beforeStep(), afterStep(), beforeChunk(), afterChunk()
- afterStep()을 제외한 다른 모든 메서드는 void, aterStep()은 ExitStatus를 반환한다.
- 스프링 배치에서 리스너 구현을 간단히 할 수 있도록 @AfterStep, @BeforeStep, @AfterChunk, @BeforeChunk 어노테이션 제공
