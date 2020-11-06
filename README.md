* 참고

jungwookim37

https://github.com/tkinma/Kakao

jaesunyoon


# 신규배송상태알람

## 이벤트 스토밍 
![image](https://user-images.githubusercontent.com/24926691/98333718-b450ef00-2044-11eb-8ed4-cc15ff1d3953.png)

## 신규 배송 알림
    - KPI: 배송 상태 및 취소 상태 실시간 체크
    - 팀 과제의 Bookmarket에 주문ID 및 Status를 통한 각 사용자 별 Message 수신
    
## 헥사고날 아키텍처 변화 
![image](https://user-images.githubusercontent.com/70673830/98327470-e9564500-2036-11eb-8ece-5f7d1d74d67f.png)

##CQRS
customerview(mypage)에 추가로 통해 구현하였다.

![image](https://user-images.githubusercontent.com/70673830/98318185-6f679100-2021-11eb-8ca6-f3990f870b7c.png)

##gateway
gateway 프로젝트 내 application.yml 에 추가로 구성

![image](https://user-images.githubusercontent.com/70673830/98318472-046a8a00-2022-11eb-8a6d-ddb46d76af0d.png)

## 폴리글랏 퍼시스턴스
bookMessage 서비스에는 H2 DB 대신 HSQLDB를 사용하기로 하였다. 이를 위해 메이븐 설정(pom.xml)상 DB 정보를 HSQLDB를 사용하도록 변경하였다.

![image](https://user-images.githubusercontent.com/70673830/98321420-6201d500-2028-11eb-9321-ea74966a4d84.png)


## Kafka 기동 및 모니터링 용 Consumer 연결
```
kubectl -n kafka exec -ti my-kafka-0 -- /usr/bin/kafka-console-consumer --bootstrap-server my-kafka:9092 --topic bookmarket --from-beginning
```

### 책 주문 생성
![image](https://user-images.githubusercontent.com/70673830/98305636-9a43ec00-2005-11eb-8af2-9061a422f24b.png)

### 결재 상태 확인
![image](https://user-images.githubusercontent.com/70673830/98305714-c0698c00-2005-11eb-82c4-7a744b9fd96b.png)

### 배송 상태 확인
![image](https://user-images.githubusercontent.com/70673830/98305776-db3c0080-2005-11eb-9675-871d7960faa8.png)

### CI/CD Pipelines 구성
![image](https://user-images.githubusercontent.com/24926691/98331086-258da380-203f-11eb-879e-cdcf7bb81958.png)

![image](https://user-images.githubusercontent.com/24926691/98331168-5a015f80-203f-11eb-97e9-f46b0b1c1d56.png)

### 배송 상태 Book Message 확인 Monitoring
![image](https://user-images.githubusercontent.com/70673830/98305857-fad32900-2005-11eb-96b8-42b33f33ecb5.png)


## 오토스케일 아웃
- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 20개까지 늘려준다:
![image](https://user-images.githubusercontent.com/70673830/98315204-0e3cbf00-201b-11eb-9f9d-d76a9214743a.png)

* autoscale 을 위해 deployment.yml 에  resource 설정
![image](https://user-images.githubusercontent.com/24926691/98332936-085ad400-2043-11eb-98f8-22ebfe467249.png)


* autoscale 테스트를 위해 bookAlarm 에 외부 노출 Mapping
![image](https://user-images.githubusercontent.com/24926691/98332911-fc6f1200-2042-11eb-9204-0ec9dc255c3b.png)


* hpa  설정하기
-----------------------
kubectl autoscale deploy bookalarm --cpu-percent=20 --min=1 --max=20 -n books

* 테스트 중...꼬임....


## Circuit Breaker 점검
```
Hystrix Command
	5000ms 이상 Timeout 발생 시 CircuitBearker 발동

CircuitBeaker 발생
	http http://bookAlarm:8080/selectBookAlarmInfo?id=0
		- 잘못된 쿼리 수행 시 CircuitBeaker
		- 10000ms(10sec) Sleep 수행
		- 5000ms Timeout으로 CircuitBeaker 발동
		- 10000ms(10sec) 
```
### 실행 결과
```
HTTP/1.1 200
Content-Length: 17
Content-Type: text/plain;charset=UTF-8
Date: Thu, 05 Nov 2020 04:22:29 GMT

CircuitBreaker!!!
```
### 소스코드
```
  @GetMapping("/selectBookAlarmInfo")
  @HystrixCommand(fallbackMethod = "fallback", commandProperties = {
          @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000"),
          @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "10000")
  })
  public String selectBookAlarmInfo(@RequestParam long id) throws InterruptedException {

   if (id <= 0) {
    System.out.println("@@@ CircuitBreaker!!!");
    Thread.sleep(10000);
    //throw new RuntimeException("CircuitBreaker!!!");
   } else {
    Optional<BookAlarm> bookAlarm = bookAlarmRepository.findById(id);
    return bookAlarm.get().getBookMessage();
   }

   System.out.println("$$$ SUCCESS!!!");
   return " SUCCESS!!!";
  }

  private String fallback(long id) {
   System.out.println("### fallback!!!");
   return "CircuitBreaker!!!";
  }
```
