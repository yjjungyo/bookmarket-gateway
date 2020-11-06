

# 신규배송상태알람

## 이벤트 스토밍 
![image](https://user-images.githubusercontent.com/70673830/98245081-bcad1980-1fb3-11eb-8f63-fdda45462788.png)

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

### CI/CD 점검
![image](https://user-images.githubusercontent.com/70673830/98233337-405e0a80-1fa2-11eb-977b-577920b03a21.png)
![image](https://user-images.githubusercontent.com/70673830/98233383-51a71700-1fa2-11eb-9af9-630338761118.png)

### 배송 상태 Book Message 확인 Monitoring
![image](https://user-images.githubusercontent.com/70673830/98305857-fad32900-2005-11eb-96b8-42b33f33ecb5.png)


## 오토스케일 아웃
- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 20프로를 넘어서면 replica 를 20개까지 늘려준다:
![image](https://user-images.githubusercontent.com/70673830/98315204-0e3cbf00-201b-11eb-9f9d-d76a9214743a.png)

- Circuite Breaker 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
```
siege -c100 -t120S -v --content-type "application/json" 'http://20.196.153.152:8080/orders POST {"bookId": "10", "qty": "1", "customerId":"1002"}'
```
![image](https://user-images.githubusercontent.com/70673830/98315297-3debc700-201b-11eb-8a6e-b74395eade27.png)

- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
```
kubectl get deploy payment -w
```
![image](https://user-images.githubusercontent.com/70673830/98315692-1fd29680-201c-11eb-840b-f97ea6e8238d.png)

- 어느 정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인.


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
