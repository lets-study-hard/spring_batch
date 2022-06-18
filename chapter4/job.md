## Job & Step
- 잡과 스텝을 실행했을 때 진행되는 과정 (실행에 필요한 구성 읽기 + 구성의 유효성 검증 + 잡 스텝의 종료)
- 다양한 구성 방법
- 여러 배치 컴포너트가 다양한 스코프간 데이터 전달하는 방법

---

### 1. Job 이란?

"처음부터 끝까지 독립적으로 실행가능한, 고유하며 순서가 있는 **스텝의 목록**이다."

- 유일하다.
  - 스프링 프레임워크의 빈 구성처럼 자바나 XML으로 구성
  - 구성한 내용을 재사용 가능 (여러번 정의할 필요 없음)
- 순서를 가진 여러 스텝의 목록이다.
  - 어플리케이션의 논리 흐름과 같다
- 처음부터 끝까지 실행가능하다.
  - 외부 의존성 없이 실행하는 일련의 스텝
- 독립적이다.
  - 외부 의존성 없이 실행가능 != 외부 의존성을 가질수 없다
  - 의존성을 관리할 수 있다.
  - ex. 파일이 없을때의 오류 처리, 파일이 전달될때 까지 기다리지 않고 스스로 제어 
  - 배치 실행중 사용자가 처리에 관여하는 부분은 없다.

### 2. 잡의 생명주기

잡의 정의 = 잡의 인스턴스 생성을 위한 청사진  
자바 클래스 코드 작성 = JVM에서 인스턴스 생성을 위한 청사진

#### 잡 러너(job runner)

잡의 실행은 잡 러너(job runner)에서 시작된다.

① 스프링 배치에서 제공   
_패키지 : org.springframework.batch.core.launch.support_

- CommandLineJobRunner
  - 스크립트나 명령행에서 직접 잡을 실행
  - **스프링을 부트스트랩**하고 전달받은 파라미터로 요청잡을 실행
- JobRegistryBackgroundJobRunner
  - 실행 중인 자바 프로세스내에서 **스케줄러를 사용해 잡을 실행**할때,
  - 스프링 부트스트랩시 실행가능한 잡을 가지는 **JobRegistry를 생성하는데 사용**된다.

⓶ 스프링 부트에서 제공

- JobLauncherCommandLineRunner
  - 구현체의 별도 구성이 없다면, ApplicationContext에 정의된 Job타입의 모든 빈을 기동시에 실행

> **잡 러너는 프레임워크의 표준 모듈이 아니다.**  
시나리오마다 서로 다른 구현체가 필요함에 따라 프레임워크는 JobRunner라는 인터페이스를 제공하지 않는다.   
> (위의 잡 러너들도 공통기능으론 main메서드 밖에 없다.)

**SimpleJobLauncher**
- 스프링 배치의 단일 JobLauncher인터페이스의 구현체
- 프레임워크 실행시, 실제 진입점
- 잡 러너 내부의 클래스
- 요청된 잡을 실행시 코어 스프링의 TaskExector 인터페이스 사용 (다양한 TaskExector구성 방법있음)
  - SyncTaskExector사용시 잡은 JobLauncher와 동일한 스레드에서 실행 (별도의 스레드에서 실행도 가능)

**배치잡 실행 → JobInstance생성(잡이름+파라미터) → JobExecution**
  - JobInstance의 완료 = 완료된 JobExecution가 존재
  - 완료된 JobInstance는 재실행 될 수 없다.

JobInstance
- JobRepository의 DB로 상태를 식별 (4장에서 자세히...)
- BATCH_JOB_INSTANCE (JOB_KEY = 잡 이름+파라미터의 해시값)
- BATCH_JOB_EXECUTION_PARAMS


JobExecution
- 잡 실행의 실제시도
- BATCH_JOB_EXECUTION
- BATCH_JOB_EXECUTION_CONTEXT (상태식별)

### 3. 잡 구성하기

```java
@EnableBatchProcessing // 어플리케이션에서 한번만 적용, 배치잡 수행시 필요한 인프라스트럭처를 제공
@SpringBootApplication // 스프링 부트를 부트스트랩, @Configuration를 포함 (생략가능)
public class HelloWorldJob {

    @Autowired
    private JobBuilderFactory jobBuilderFactory; // JobBuilder 인스턴스 생성
    
    @Autowired
    private StepBuilderFactory stepBuilderFactory; // StepBuiler 인스턴스 생성
    
    @Bean
    public Job job() { // job생성
      return this.jobBuilderFactory.get("basicJob")
              .start(step1())
              .build();
    }
    
    @Bean
    public Step step1() { // step생성
      return this.stepBuilderFactory.get("step1")
              .tasklet((contribution, chunkContext) -> {\
                              System.out.println(String.format("Hello, %s!", name));
                              return RepeatStatus.FINISHED;
                          })
              .build();
    }
    
    public static void main(String[] args) {
              SpringApplication.run(HelloWorldJob.class, args);
    }
	
}
```

### 3. 잡 파라미터

JobInstanceAlreadyCompleteException
- 동일한 파라미터로 완료된 잡을 재실행시 오류 발생

잡에 파라미터 전달방법?
- 사용자가 잡을 어떻게 호출하는지에 따라 달라짐
- 잡 러너의 기능중 하나
  - 잡 실행에 필요한 JobParameters 객체를 생성해 JobInstance에 전달

① commandLine (JobLauncherCommandLineRunner)  
```shell
java -jar demo.jar name=Michael
```
- 잡 러너가 JobParameters 인스턴스 생성, 잡이 전달받는 모든 파라미터의 컨테이너 역할을 한다.


⓶ JobRegistryBackgroundJobRunner
