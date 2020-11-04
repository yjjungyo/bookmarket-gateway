Skip to content
Search or jump to…

Pull requests
Issues
Marketplace
Explore
 
@cha6471 
jaesunyoon
/
bookmarket-gateway
1
03
Code
Issues
7
Pull requests
Actions
Projects
Wiki
Security
Insights
You’re making changes in a project you don’t have write access to. Submitting a change will write it to a new branch in your fork cha6471/bookmarket-gateway, so you can send a pull request.
bookmarket-gateway
/
README.md
 

Spaces

2

Soft wrap
525
- 결제서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 15프로를 넘어서면 replica 를 10개까지 늘려준다:
526
```
527
kubectl autoscale deploy pay --min=1 --max=10 --cpu-percent=15
528
```
529
- CB 에서 했던 방식대로 워크로드를 2분 동안 걸어준다.
530
```
531
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"bookId": "10", "qty": "1", "customerId": "1002"}'
532
```
533
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다:
534
```
535
kubectl get deploy pay -w
536
```
537
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
538
```
539
NAME    DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
540
pay     1         1         1            1           17s
541
pay     1         2         1            1           45s
542
pay     1         4         1            1           1m
543
:
544
```
545
- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
546
```
547
Transactions:           5078 hits
548
Availability:          92.45 %
549
Elapsed time:          120 secs
550
Data transferred:         0.34 MB
551
Response time:            5.60 secs
552
Transaction rate:        17.15 trans/sec
553
Throughput:           0.01 MB/sec
554
Concurrency:           96.02
555
```
556
​
557
​
558
## 무정지 재배포
559
​
560
* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함
561
​
562
- seige 로 배포작업 직전에 워크로드를 모니터링 함.
563
```
564
siege -c100 -t120S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"bookId": "10", "qty": "1", "customerId": "1002"}'
565
​
566
** SIEGE 4.0.5
567
** Preparing 100 concurrent users for battle.
568
The server is now under siege...
569
​
570
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
571
HTTP/1.1 201     0.68 secs:     207 bytes ==> POST http://localhost:8081/orders
572
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
573
HTTP/1.1 201     0.70 secs:     207 bytes ==> POST http://localhost:8081/orders
574
:
575
​
576
```
577
​
578
- 새버전으로의 배포 시작
579
```
580
kubectl set image ...
@cha6471
Propose changes
Commit summary
Update README.md
Optional extended description
Add an optional extended description…
 
© 2020 GitHub, Inc.
Terms
Privacy
Security
Status
Help
Contact GitHub
Pricing
API
Training
Blog
About
