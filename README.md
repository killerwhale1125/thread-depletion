# Native Thread 고갈 유발 및 해결 과정

### 파일 압축 해제  
```tar -xvf assignment.tar```

### 디렉토리 구성
./assignment
-  apache-tomcat-8.5.93     ->    톰캣
-  docker-compose.yml       ->    초기 install docker-compose 설정 파일
-  docker-compose2.yml     ->    Tomcat, Nginx 재실행 docker-compose 설정 파일 [ Oracle 제외 - 오래걸려서 ]
-  init.sql                              ->    DB 유저 설정, 테이블 생성, 300만 DB 데이터 삽입
-  jvm                                   ->    java 8
-  k6-install.sh                      ->    k6 Linux install
-  k6-script.js                        ->    k6 부하 script
-  nginx_config                     ->    nginx conf 파일
-  install.sh                           ->    Docker image pull, docker-compose up, db init
   Tomcat, Nginx, Oracle 설치 [ 우선적으로 실행해주세요 ]
   sudo ./install.sh
1. 필요한 Docker Image 설치 ( Oracle 오래 걸림 - 1~2분 소요 예정 )
2. Docker Container 구동 ( Oracle 오래 걸림 - 10분 정도 소요 예정 )
    - 대기 문구 알림 있습니다. 오래 걸려도 모든 작업 Successful 뜰 때까지 기다려주시면 감사하겠습니다.
3. DB 테이블 생성 및 300만 건 데이터 삽입 ( 1분 소요 )

✅ **최종 완료 문구**

```Disconnected from Oracle Database 12c Standard Edition Release 12.1.0.2.0 - 64bit Production```

✅ **docker-compose version 문제로 실행 에러 발생 시 조치 [ 선택 사항 ]**

``` sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose```

```sudo chmod +x /usr/local/bin/docker-compose```

### K6 설치

```sudo ./k6-install.sh```



### API 구성

| URL                            | Description                  |
|--------------------------------|------------------------------|
| `http://127.0.0.1/slow`        | DB 병목 유발 API              |
| `http://127.0.0.1/depletion`   | Thread 과생성 유발 API        |
| `http://127.0.0.1/prevention`  | Thread Pool 전략 영구 조치 API |

### Install 후 Tomcat, Nginx 재실행 [ 선택 사항 ]  
``` sudo ./shutdown -> Nginx, Tomcat만 Stop [ Oracle DB 설치 작업이 오래걸려 제외 ] ```

``` sudo ./start.sh -> Nginx, Tomcat 재시작 ```

### K6 Script 실행
``` K6_WEB_DASHBOARD=true k6 run k6-script.js ```


✅ **참고 사항** 
- K6 부하 테스트 실행 전 별도 서버에서 k6 사용 시 API 주석 처리와 IP 반드시 설정 변경 후 테스트 진행
부탁드립니다. 
- 서버 스펙에 따라 실행한 결과와 보고서 내용이 상이할 수 있음을 양해 부탁드립니다. 
- 실제 부하 실행 전 서버 스펙에 맞춰 적절한 부하량 조절 부탁드립니다.


### K6 Script 수정
sudo vi k6-script.js
-  부하량 조절 - target : { 설정할 부하량 } ex 300 
-  url 변경 - const url = http://{ IP }:80/depletion 에서 IP 변경 ( 기본값 localhost )



### 문제 상황

```org.apache.tomcat.util.threads.ThreadPoolExecutor$Worker.run java.lang.OutOfMemoryError: unable to create new native thread```

HTTP 502 Bad Gateway 알람 대량 발생 문제로 인해 log 확인 결과 Tomcat이 더이상 Native Thread를 생성할 수  
없어 Tomcat 서버가 마비되어 502 response가 발생하였습니다.

**테스트 서버 스펙**
-  WAS, DB, Nginx EC2 [t3.medium] 
-  CPU 2core, RAM 4GB, EBS 30GB 단일 서버 구성 
-  K6 [Local 개인 PC] - CPU 12core, RAM 20GB

정확한 테스트 지표를 위해 K6는 별도의 서버(개인 PC) 에서 진행하였습니다.

### 프로세스 버전 및 환경설정 -  OS Ubuntu 24.04.2 LTS 
-  Tomcat 8.5.93 [Docker]
   -  maxThreads - 200
   -  accept-count - 10
   -  Xss4m
-  ulimit -u - 150
-  Nginx [Docker]
   -  proxy_connect_timeout - 5s
   -  proxy_send_timeout - 5s
   -  proxy_read_timeout - 5s
-  Oracle-12c [Docker]
   -  send_timeout - 5s 
   -  DBCP maxActive - 50

### 상황 재현 시도 1 [ DBCP 점유 ]

![image](https://github.com/user-attachments/assets/08c23252-83e5-4b67-bbe0-8e007c04e076)

Tomcat maxThreads 200 설정으로 Thread 고갈이 가능할까?
-  max 200 설정은 즉 Thread Pool 갯수가 200개로 제한되는 것
-  201번째부터는 max-connections 값 까지 TPC 연결 요청을 수립하여 TaskQueue에 Thread-Pool을 얻을 때 까지 요청  Channel 을 대기 시킴
-  요청이 max-connections 값 초과 시 OS backLog Queue에 accept-Count 수 만큼 요청을 대기

즉, Tomcat은 maxThreads 만큼만 추가 Thread를 생성하고 초과로 Thread를 생성되지 않아 native thread 고갈이 안된다고 판단


**DBCP maxTotal=50 설정으로 병목을 유발한다면 가능할까?**
-  동시 요청 중 DB 연결이 필요한 건 maxThreads 갯수 중 50개까지만 가능
-  초과 요청은 DBCP 를 얻기 위해 대기

즉 DB 커넥션 풀 부족이라도 Thread 생성은 Tomcat maxThreads 제한 영향을 받기 때문에 추가 스레드 생성이 유도되지
않는다고 판단

✅ **테스트 코드**
```java
/**
 DB 300만 데이터에서 1만개의 데이터 조회를 통해 DBCP를 오래 점유하도록 구성
 */
@GetMapping("/slow")
public ResponseEntity<String> slow() throws InterruptedException, JsonProcessingException {
   System.out.println(Thread.currentThread().getId());
   String query = "SELECT o.id, o.user_id, o.product_name, o.amount, o.ordered_at, u.username,   
   u.email " +
   "FROM mcc.orders o " +
           "JOIN mcc.users u ON o.user_id = u.id " +
           "FETCH FIRST 10000 ROWS ONLY";

   // DTO 매핑 메모리 낭비 유발 
   List<OrderDTO> orders = jdbcTemplate.queryForList(query).stream()
           .map(row -> new OrderDTO(
                   ((Long) row.get("id")).longValue(),
                   ((Long) row.get("user_id")).longValue(),
                   (String) row.get("product_name"),
                   ((Integer) row.get("amount")).intValue(),
                   (Timestamp) row.get("ordered_at"),
                   (String) row.get("username"),
                   (String) row.get("email")
           ))
           .collect(Collectors.toList());

   return ResponseEntity.ok("ok");
}
```


### 테스트 결과

✅ **DB 병목 상황 maxThreads 제한으로 시간차 간격 정상 response**

![image](https://github.com/user-attachments/assets/6e988784-867d-497b-baa0-e44e9febf8dd)

✅ **결과**

단순히 maxThreads = 200, DBCP maxAvtive - 50  설정으로 native Thread 고갈이 발생하지 않았으며, Tomcat과 커넥션 풀
설계 상 제한으로 Thread 생성이 통제되기 때문이라 판단하였습니다.

### 상황 재현 시도 2 [ Thread 수동 생성 및 Thread Pool 점유 ]

✅ **테스트 코드** 

```java
/* 캐시에 비동기 처리 한다고 가정 */
@GetMapping("/depletion")
public ResponseEntity<String> depletion() throws InterruptedException {
   Thread t = new Thread(() -> {
      try {
          Thread.sleep(60000); // 1분 동안 스레드 점유 ( 비동기 캐싱 처리 가정 )
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
   });
   t.start();
   return ResponseEntity.ok("ok");
}
```

극단적인 방법이지만 Native Thread 유발을 위해 Thread를 수동 생성하는 방법을 선택하였습니다.


✅ **Thread 급격히 증가**

![image](https://github.com/user-attachments/assets/de7a8bed-9dc4-422d-a616-97a56efe7ebc)

✅ **Tomcat, OS Thread 생성 실패**

![image](https://github.com/user-attachments/assets/d9cfa80c-23d8-4694-a63f-a669a822dc20)
-  Tomcat log 확인 시 과도한 요청 및 메모리 소모로 OS Level Native Thread 생성 불가능 확인


✅ **Thread dump 내용**

![image](https://github.com/user-attachments/assets/0b06669a-a397-4be2-8034-c8f45ab3810a)
-  Thread Pool 이 꽉차 TaskQueue에 WAITING 중인 Threads 
-  Thread #4497 번호를 보아 현재 수많은 스레드가 생성되고 있음을 유추 가능



✅ **Nginx 응답 로그**
-  초기에는 HTTP 500 에러가 반환되다가, 이후에는 톰캣 응답이 지연되면서 HTTP 504 Gateway Timeout 발생 
-  proxy_read_timeout 등의 설정을 통해 타임아웃을 조절 가능하지만, 스레드 고갈 시에는 톰캣이 응답을 돌려주지 못함

✅ **OS가 Native Thread 생성을 거부한 이유**

테스트 당시 ulimit -u, vm.max_map_count, Docker 메모리 제한 모두 충분했음에도 native thread 생성 오류가 발생한
이유는, 제가 설정한 스레드 stack 크기 -Xss4m에 따라 4MB의 stack을 native memory에서 할당하려 했기 때문입니다.
시스템 전체 물리 메모리가 4GB에 불과하고 ( process 할당 후 available - 1.3GB ) swap도 없는 상황에서는 4500개 이상의
스레드가 생성되면서 native memory 부족이 발생했고, 결국 pthread_create() 실패로 이어졌습니다.

### 🛠️ 긴급 조치 (Hot-Fix)

| 조치 항목             | 기대 효과 |
|----------------------|-----------|
| OS 설정 `ulimit -u`  | 스레드 생성 제한 값 증가<br>(※ 해당 테스트에는 적합하지 않음) |
| JVM 옵션 `-Xss`      | 스레드 Stack 크기 감소 → Thread stack 메모리 절감 → 더 많은 Thread 생성 가능 |
| Swap 메모리          | 부족한 native memory 보완 |

### 🔧 영구 조치 (Permanent Fix)

| 항목                        | 조치안 |
|---------------------------|--------|
| ThreadPool 전략            | `ExecutorService` 로 Thread 생성 제어 및 `BlockingQueue` 사용 시 갯수 제한 |
| RejectedExecutionHandler  | 예외 정책 필수 설정 |
| 비동기 처리 성능 개선      | 병목 유발 코드 성능 최적화 |

✅ **영구 조치 코드 [ ThreadPool 고정 풀 전략 ]**

```java
private ExecutorService executor; // LinkedBlockingQueue 사용 
 
@GetMapping("/prevention") 
public ResponseEntity<String> prevention() { 
    try { 
        executor.submit(() -> { 
            try { 
                Thread.sleep(60000); // 1분 점유 
            } catch (InterruptedException e) { 
                Thread.currentThread().interrupt(); 
            } 
        }); 
        return ResponseEntity.ok("ok"); 
    } catch (RejectedExecutionException e) { 
        // 큐가 가득 찼거나, maxPoolSize 초과 등으로 작업 거부 
        return ResponseEntity.status(503).body("Server Busy - Please retry later."); 
    } 
} 
 
static class CustomRejectionHandler implements RejectedExecutionHandler { 
   @Override 
   public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) { 
       System.err.println("Queue Size max!! - Now Queue Size : " + 
executor.getQueue().size()); 
       throw new RejectedExecutionException("Queue size max!"); 
   } 
}
```


✅ **LinkedBlockingQueue 설정 정보**
-  corePoolSize : 10 
-  maxPoolSize : 100 
-  keepAliveTime : 10L 
-  queueCapacity : 500

최대 작업 가능 Thread를 100개로 지정하였으며 Queue Size 500 지정함으로서 대량 요청에도 500개까지 대기하며 나머지
요청은 RejectedExecutionException 발생시켜 응답하도록 설정하였습니다.

✅ **정상적으로 스레드 생성 제한 로그 출력**

![image](https://github.com/user-attachments/assets/d4a5f515-677d-4481-ac66-4a19bb3d31e7)
-  Queue Size 초과하는 대량 요청 발생 시 정상적으로 RejectedExecutionException 예외 발생 응답

✅ **max - 100 까지 생성 후 나머지 생성 요청은 BlockingQueue에서 대기 확인**

![image](https://github.com/user-attachments/assets/e190d0da-6d9a-46f1-b4f4-f2c4210670a6)
 -  내부 코드 설정인 Thread.sleep(...)으로 인해 TIMED_WAITING 상태 -  pool-1-thread-100 인 것을 보아 max 값으로 설정한 100 까지 생성된 것을 확인

✅ **스레드 생성 속도 확인**

![image](https://github.com/user-attachments/assets/8c1c024a-effb-4ef3-a1fd-3c7f247480c8)
-  2초 간격으로 현재 스레드 사용 갯수 조회 시 스레드 생성이 안정적으로 유지됨

✅ **Nginx Log**

![image](https://github.com/user-attachments/assets/b2dcbcd4-c895-47bc-ab0e-0cc125870fff)
-  200 과 의도한 503 Status 가 번갈아가며 응답되어 정상적으로 제한하고 있음

### 부작용 및 롤백 계획

### ⚠ 예상 위험 및 부작용

| 항목             | 위험 요소               | 설명 |
|------------------|------------------------|------|
| `ulimit -u` 증가 | 무제한 Thread 생성 가능성 | ThreadPool 외에서 Thread가 무제한 생성되면 OOM(OutOfMemoryError) 발생 위험 |
| `-Xss` 감소      | StackOverflow 가능성 증가 | 호출 스택이 깊은 서비스에서는 StackOverflowError 예외 발생 우려 |
| Queue Size 과도 설정 | 메모리 과점유 및 지연 | Queue가 너무 크면 대기 요청이 많아지고, 처리 지연 및 메모리 사용량 증가 |


### 🔁 롤백 계획

| 대상 조치 항목      | 롤백 시나리오                             | 롤백 방법 |
|--------------------|------------------------------------------|-----------|
| `ulimit -u` 증가   | ulimit 값을 높인 상태에서 Thread 고갈 발생 시 | `/etc/security/limits.conf`에서 기존 soft/hard `nproc` 값으로 복원 후 서비스 재시작 |
| `-Xss` 감소        | StackOverflowError 발생 시               | JVM 옵션을 원래 값으로 복원 후 서비스 재시작 |
| Queue Size 과도 설정 | Queue size가 너무 작아 대량 503 응답 발생 시 | `@Profile` 또는 `@Conditional`로 staging ↔ prod 분리 구조 설정 후, `application.yml` 값으로 제어 |
