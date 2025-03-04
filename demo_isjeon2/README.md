**스프링배치 - 기도 세션2**

https://github.com/schooldevops/spring-batch-tutorials/blob/12.Listener/01.InitSpringBatch.md

<img src="/Users/1112231/Library/Application Support/typora-user-images/image-20250304195347479.png" alt="image-20250304195347479" style="zoom:50%;" />

**1. Spring Batch 기본 개념**

•	**Spring Batch 구성 요소**:

​	•	**Step**: 배치 처리의 단위

​	•	**Item Reader**: 데이터를 읽어오는 역할

​	•	**Item Processor**: 데이터를 가공하는 역할

​	•	**Item Writer**: 데이터를 저장하는 역할

​	•	**Job Repository**: 배치 실행 정보를 DB에 저장하는 역할

•	**배치 실행 정보 관리**

​	•	Step 실행 시 몇 건이 실행되었는지, 몇 건이 스킵되었는지, 완료 여부 등을 Job Repository에 저장

​	•	Spring Batch 실행 시 Job Instance가 생성되며, Job Execution 정보를 관리함



**2. Spring Batch** **실행** **흐름**

1. **Job Instance 생성**:

​	•	첫 실행 시, batch_job_instance 테이블에 배치 잡 키와 정보 저장

​	•	실행마다 새로운 job_execution 생성하여 관리 (여러 개의 execution 가능)

​	•	실행 시각(시작, 종료) 및 상태(COMPLETED, FAILED 등) 저장



2. **Step Execution 관리**:

​	•	Step Execution 정보도 별도 관리 (batch_step_execution 테이블)

​	•	실행된 데이터 개수(총 처리 건수, 스킵된 건수, 오류 발생 건수 등) 저장

​	•	Step Execution이 중단되면 저장된 정보를 기반으로 재시작 가능

3. **Execution Context 저장**:

​	•	배치가 실행된 환경(EC2, EBS 등)에 대한 정보를 직렬화하여 저장

​	•	중단 시 해당 정보를 로드하여 재실행 가능

4. **배치 실행 시 Parameter 처리**:

​	•	시작 시간, 종료 시간 등의 파라미터 저장

​	•	동일한 파라미터로 실행될 경우, 중복 실행 방지



**3. Spring Batch** **아키텍처**

•	**Spring Batch는 DI(Dependency Injection)와 AOP(Aspect-Oriented Programming)를 지원**

​	•	DI: Job, Step 등을 빈(Bean)으로 등록하여 관리

​	•	AOP: Step 실행 시 어노테이션을 활용해 파라미터 전달 가능

•	**Spring Batch 실행 모델**

​	•	**Tasklet Model**:

 	•	단순한 처리 방식

​		•	한 개의 Step 내에서 하나의 작업을 수행

​		•	트랜잭션이 길어지면 비효율적

​	•	**Chunk Model**:

​		•	데이터를 특정 크기(Chunk)로 나누어 처리

​		•	Chunk 단위로 트랜잭션이 수행되어 장애 발생 시 특정 Chunk만 재실행 가능

​		•	데이터 읽기 → 가공 → 저장의 흐름을 유지





**4. Spring Batch** **실행** **과정**

​			1.	**Job Scheduler 실행**:

​				•	젠킨스, Step Function, 크론(Cron) 등을 이용하여 스케줄링

​				•	Job Launcher가 Job을 실행

​			2.	**Job Launcher 실행**:

​				•	Job Instance 생성 및 Job Repository 저장

​				•	Step 실행 시 Step Execution 정보 저장

​			3.	**Step 실행**:

​				•	Step 내에서 Item Reader → Item Processor → Item Writer 순으로 실행

​				•	처리된 데이터 개수 및 실행 결과를 DB에 기록

​			4.	**Job 완료 및 종료**:

​				•	Job Execution 상태 업데이트

​				•	최종적으로 모든 데이터 처리가 완료되면 Job 종료



**5. Spring Batch** **프로젝트** **설정**

​			1.	**Spring Boot 프로젝트 생성**:

​				•	start.spring.io에서 Java 프로젝트 생성

​				•	Gradle을 사용하고, Spring Batch 관련 라이브러리 추가

​			2.	**Application 설정 (application.yml)**

```
spring:
  batch:
    job:
      names: myJob
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
    username: sa
    password: 
```





​			3.	**Spring Batch Job 구성**

​				•	@Configuration을 이용해 Job과 Step 설정

​				•	@Bean으로 Job, Step, Tasklet을 정의하여 Spring Context에 등록

​			4.	**Tasklet 구현 예시**

```
@Component
public class GreetingTasklet implements Tasklet {
    private static final Logger log = LoggerFactory.getLogger(GreetingTasklet.class);

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) {
        log.info("Hello, Spring Batch!");
        return RepeatStatus.FINISHED;
    }
}
```







​			5.	**Job Configuration**

```
@Configuration
public class BatchConfig {
    @Bean
    public Job myJob(JobRepository jobRepository, Step myStep) {
        return new JobBuilder("myJob", jobRepository)
            .start(myStep)
            .build();
    }

    @Bean
    public Step myStep(JobRepository jobRepository, PlatformTransactionManager transactionManager, GreetingTasklet tasklet) {
        return new StepBuilder("myStep", jobRepository)
            .tasklet(tasklet, transactionManager)
            .build();
    }
}
```









**6. TransactionManager의 역할**

​	•	**Spring 라이프사이클에 따라 트랜잭션을 관리**

​	•	**DB Isolation Level 관리**:

​		1.	READ_UNCOMMITTED: 커밋하지 않은 데이터도 읽을 수 있음

​		2.	READ_COMMITTED: 커밋된 데이터만 읽을 수 있음

​		3.	REPEATABLE_READ: 동일한 트랜잭션 내에서 동일한 데이터를 읽을 수 있음

​		4.	SERIALIZABLE: 모든 트랜잭션이 순차적으로 실행됨 (성능 저하 가능)

​	•	**DB Propagation 관리**:

​		•	트랜잭션을 새로운 트랜잭션으로 만들거나 기존 트랜잭션을 사용할지 설정

​		•	트랜잭션 매니저를 여러 개 두고 선택적으로 사용 가능







**7. 배치 실행 및 문제 해결**

​	•	**배치 실행 명령어**

```
./gradlew bootRun
```



​	•	**실행 중 발생한 문제 및 해결책**

​		•	**DB 설정 오류**: H2 라이브러리가 build.gradle에 추가되지 않음 → 라이브러리 추가 후 해결

​		•	**Tasklet 무한 루프**: RepeatStatus.CONTINUABLE로 설정 → RepeatStatus.FINISHED로 변경하여 해결



**8.** **다음** **세션** **계획**

​	•	현재는 메모리 기반(H2) DB를 사용했지만, 다음 세션에서는 실제 MySQL DB를 연동하여 데이터 저장 방식 학습 예정

​	•	직접 배치 실행 후 결과 확인할 예정

​	•	발표할 사람 모집하여 실습 공유 예정