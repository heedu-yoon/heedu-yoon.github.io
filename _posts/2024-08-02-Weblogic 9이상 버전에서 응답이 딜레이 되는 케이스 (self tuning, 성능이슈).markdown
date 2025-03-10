---
layout: post
title: Weblogic 9이상 버전에서 응답이 딜레이 되는 케이스 (self tuning, 성능이슈)
date: 2024-08-02 19:20:23 +0900
category: was
---

## WebLogic 9이상 버전 사용기
weblogic을 사용하는 상황에서 urlconnection을 사용하여 요청을 주고받는 경우가 있었다.
이때 약 5초 정도 서버가 멈춘 것처럼 응답이 딜레이 되는 경우가 있어 분석을 시작했다.</br>
결론부터 말하자면 weblogic 9부터 추가된 self tuning기능이 문제였다.</br>
9버전에서 self-tuning을 온전히 사용하는 이상 이런 현상은 항상 발생할 것으로 예상된다.
이에 대한 대응은 두 가지 방법이 있다.

#### 1. Dweblogic.Use81StyleExecuteQueues=true 옵션을 사용하여 self tuning기능 자체를 사용하지 않도록 한다. 해당 옵션은 8.1 버전의 방식으로 thread가 동작하도록 한다.
#### 2. Dweblogic.threadpool.MinPoolSize=50 -Dweblogic.threadpool.MaxPoolSize=50 옵션을 사용하여 thread를 모두 ACTIVE상태로 생성하여 tuning을 위한 thread생성 및 상태 변환을 하지 않도록 한다. 해당 내용에 대해서는 아래 성능 이슈 분석 내용에서 함께 다루도록 한다. JMeter부하 테스트 결과 (60 user로 30초간 부하 전달)

### JMeter부하 테스트 결과 (60 user로 30초간 부하 전달)
| 옵션 | Average Time(ms) | Max Time(ms) |
|:---:|:---:|:---:|
| 기본(self-tuning사용) | 11 | 9160 |
| Use81StyleExecuteQueues | 3 | 969 |
| MinPoolSize, MaxPoolSize 50고정 | 2 | 892 |

- 1개씩 1000번 요청 시도에서 5초 이상 딜레이가 걸리는 케이스
| 옵션 | 횟수 |
|:---:|:---:|
| 기본(self-tuning사용) | 7회 발생 |
| Use81StyleExecuteQueues | 발생하지 않음. |
| MinPoolSize, MaxPoolSize 50고정 | 발생하지 않음. |

위 이슈를 대응하면서 weblogic자체에 성능 이슈가 존재하는 것이 아닌가 하는 의문을 가지게 되었고,</br>
이를 확인하기 위해 로컬에 weblogic을 설치하고 visualJVM으로 thread상태를 모니터링하였다.

- 첫 기동 후 아무 요청이 없을 때 thread상태
![img.png](img.png)

Weblogic의 self tuning기능의 통제를 받는 것으로 보이는 thread가 15개(default로 생각됨) 생성 되었고 0번 thread 1개를 제외하고는 모두 STANDBY라는 상태로 표시되어 있다.

- 요청을 1개씩 순차적으로 500번 넣은 시점 thread상태
![img_1.png](img_1.png)

응답을 받고 다음 요청을 보내는 방식으로 500번 요청한 시점에 thread상태에서는 7개의 thread가 Active상태로 전환되어 있다.



- 요청을 1000번을 넣은 시점 thread상태
![img_2.png](img_2.png)

같은 요청 계속 넣었지만 5번 thread를 제외하고는 모두 STANDBY상태로 되돌아 간 것을 확인할 수 있다.(ACTIVE가 7개에서 1개로 줄어듬).



- 요청을 멈추고 5분간 대기 후 thread상태
![img_3.png](img_3.png)

1개의 thread만 ACTIVE이지만 ACTIVE상태인 thread가 바뀐 것을 확인할 수 있다.(5번 에서 12번으로 ACTIVE thread가 변경됨)

- 딜레이가 걸리는 상황 thread상태
![img_4.png](img_4.png)
STANDBY상태의 1개 thread가 약 5초의 시간동안 delay중인 것으로 확인된다.


여기까지의 테스트 결과를 볼 때 아래와 같은 내용을 생각해 볼 수 있다.

1. 7번의 이미지상에서는 STANDBY지만 실제로 running이 되었을 때는 ACTIVE 되어 있을 것으로 추정된다. 이는 visualVM에서 thread name을 실시간으로 최신화시켜주지 않기 때문인 것으로 보이고, 실제로 visualVM을 다시 시작하면 딜레이가 발생했을 때, running이었던 thread가 ACTIVE상태가 되어있는 것을 확인할 수 있었다.
2. Weblogic의 self tuning기능은 만들어 놓은 Execute Thread의 상태를 STANDBY나 ACTIVE로 전환하는 방식으로 thread pool을 관리하는 기능으로 추정된다.(최소 1개의 thread는 ACTIVE상태로 두는 것으로 확인됨)
3. Weblogic의 self tuning기능이 어떤 알고리즘으로 thread를 관리하는지 까지는 알 수 없지만 thread들의 ACTIVE, STANDBY 상태가 수시로 변경되고 있고, 요청을 실행하기 위해서는 ACTIVE 상태로 전환되어야 하는 것으로 생각된다.
4. 이러한 상태 전환 과정에서 STANDBY -> ACTIVE로 변경될 때, 즉 5초간 딜레이가 발생하는 순간에  Thread Dump를 떠서 확인한 결과 5초간 running을 수행한 해당 thread가 항상 “sun.nio.ch.WindowsSelectorImpl$SubSelector.poll0(Native Method)” 부분에서 멈춰 있는 것을 확인할 수 있었다.
JMeter로 부하 테스트 (60 유저로 30초간 부하) (WebLogic이 기본 설정인 상태)

Jmeter 동작 중 thread상태(요청이 많아져 thread를 추가로 생성하고 있는 모습)
요청을 1개씩 넣을 때는 약 5초~6초의 딜레이가 발생하였는데, JMeter로 부하를 주었을 경우에는 최대 10초 정도의 응답 시간도 발견되었다.


weblogic thread monitoring화면
그래프의 진행을 보면 대기 중인 thread가 늘어나는 시점에 보류 중인 사용자 요청도 많이 늘어난 걸 확인할 수 있다.

만약 저 때 self tuning기능으로 새로운 대기 중인 thread가 생성되었다면 그 순간 성능 저하가 발생하여 요청이 쌓이게 되고 그 결과로 10초의 딜레이가 발생한 것으로 추정된다.

대기 중인 스레드가 생성된 시점 이후에는 다시 안정적으로 돌아가지만 저 순간의 요청에 대한 응답이 오래 걸리는 것이 아닌가 하는 생각이 든다.



Jmeter동작을 멈추고 5분간 대기후 thread 상태

Thread가 없어지지는 않았지만 1개를 제외하고 모두 STANDBY로 전환된 모습.


Jmeter로 부하를 5분간 주었을 때 response time

weblogic의 기본 설정(self tuning사용)

Response time이 튀는 것을 볼 수 있다. 그래프 상에서 가장 오래 걸린 Response는 약 8초까지 확인된다.



WebLogic의 thread 설정을 MinPoolSize=50, MaxPoolSize=50으로 설정

Response time이 안정되어 있는 것을 확인할 수 있다. 최초 시작은 약 8ms이지만, 안정된 이후로는 1ms이하의 Response time을 보여준다.



WebLogic의 thread 설정을 Use81 StyleExecuteQueues로 설정

Response time이 안정되어 있는 것을 볼 수 있다. 최초 시작은 약 1.5ms이지만, 안정된 이후로는 1ms이하의 Response time을 보여준다.



위 설정은 실행 시 환경 변수 이외에 config.xml을 수정하여 적용할 수 도 있다.

MinPoolSize=50, MaxPoolSize=50로 설정하는 경우
<self-tuning-thread-pool-size-min>50</self-tuning-thread-pool-size-min>
<self-tuning-thread-pool-size-max>50</self-tuning-thread-pool-size-max>
Use81StyleExecuteQueues로 설정하는 경우(반드시 <listen-port>위에 추가)
<execute-queue>
<name>default</name>
<thread-count>150</thread-count>
<threads-maximum>150</threads-maximum>
<threads-minimum>150</threads-minimum>
</execute-queue>
<execute-queue>
<name>weblogic.kernel.System</name>
<thread-count>100</thread-count>
<threads-maximum>100</threads-maximum>
<threads-minimum>100</threads-minimum>
</execute-queue>
<use81-style-execute-queues>true</use81-style-execute-queues>
<listen-port>60001</listen-port>