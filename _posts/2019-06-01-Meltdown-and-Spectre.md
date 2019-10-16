---
layout: post
title: "멜트다운과 스펙터(Meltdown and Spectre)"
comments: true
excerpt_separator: <!--more-->
tags:
  - Spectre
  - Meltdown
  - MyriaBreak
---

`Plaid CTF 2019`에 `Spectre`라는 666점짜리 pwn문제가 있었는데, `Spectre`를 자세히 몰라 건드려보지도 못했었다. 그러다 최근 `Meltdown`이랑 `Spectre`을 공부할 기회가 생겨서 공부한 내용을 정리하여 포스팅하게 되었다. 기회가 된다면 pctf에 나왔던 Spectre라는 문제도 한번 건드려볼 생각이다.

<!--more-->


# 멜트다운과 스펙터(Meltdown and Spectre)

최신 CPU에서 사용하는 최적화 기법인 비순차적 실행(out-of-order execution)과 추측 실행(speculative execution)를 악용한 `매우 심각한 보안 취약점`

실행중인 다른 프로그램 또는 시스템에 저장된 저장된 비밀정보를 얻을 수 있다. 저장된 암호, 개인사진, 전자메일, 인스턴트 메시지 및 업무에 중요한 문서까지 포함될 수 있다.

둘 다 `정보유출공격` 으로 읽기만 가능하고 원격코드 실행은 불가능하다.

1. 멜트다운은 Intel CPU에서 유저레벨과 커널레벨 간 발생하는 취약점
⇒ 사용자 공간에서 운영체제 권한 영역을 훔쳐본다.
2. 스펙터는 모든 CPU에서 에플리케이션 간 발생하는 취약점
⇒ 한 프로그램이 다른 프로그램의 메모리를 훔쳐본다.

## 멜트다운(Meltdown)

<center><img src="../images/myria/Meltdown-and-Spectre/meltdown.png" width="200"></center>

- CVE-2017-5754 (rogue data cache load, 불량 데이터캐시 적재)
- 인텔 CPU의 보안 결함을 이용해 CPU가 처리하고 있는 프로세스의 데이터를 낱낱이 훔쳐볼 수 있는 버그. (커널 메모리 읽기)
- 인텔 CPU와 ARM Cortex-A 시리즈 기종에서만 있는 것으로 확인됬다.
AMD 계열은 해당되지 않았으나, 2018년 말에 새로 발견된 변종 Meltdown인 Meltdown-BR (Bounds Check Bypass)가 있다. 그러나 기존 Meltdown과 다르게 매우 취약하진않다. (이건 뭐 찾아
- 1995년 이후 출시된 모든 인텔 CPU에서 해당된다.(사실상 거의 모두)
- 비순차적 명령어 처리(Out of oder exceution)와 추측 실행(speculative execution)에 의해서 발생한다.

### 추측실행(speculative execution)

`성능을 위해서 다음 실행될 명령어 예측하여 먼저 실행하는 기법`
```c
if (x > 10)    
    y = 3
```
위의 조건문에서 x > 10을 판단하기 위해서 x 값을 읽어오는데, 시간이 걸린다. 이 때 CPU를 대기시키는 것 보단 미리 y=3을 실행해둔다. 만약, 예측을 잘못한 경우 y =3의 실행결과는 폐기하고  y=3이 실행되지 않은 것처럼 된다. 예측이 맞았다면 그대로 실행하여 성능이 좋아진다.

### 비순차실행 (Out Of Order Execution , OOOE)

`성능을 위해서 순서상 뒤에 있는 명령어를 앞당겨서 실행하는 기법`

CPU가 프로그램을 실행할 때, 프로그램의 명령어들을 순서대로 실행한다. 그런데 순서상 앞에 있는 명령어를 실행하는데 시간이 오래 걸리는 경우, CPU가 노는 시간이 생길 수 있다.

    #1 A = B + C
    #2 D = A + C
    #3 G = E + F

예를 들어, 순서상 앞에 있는 명령어의 연산을 실행하기 위해서 필요한 값을 어딘가에서 읽어와야 하는 경우 그 값을 읽어 오는데 시간이 걸려 CPU가 대기하게 된다. 이런 경우에는 앞의 연산결과와 상관없는 뒤의 명령어를 먼저 실행함으로써 CPU의 유휴시간을 최소화할 수 있다.

위 코드에서 #1, #2와 상관없는 #3를 먼저 실행함으로써 CPU를 최대한 활용하는 것이다.

### Rogue data cache load, 불량 데이터캐시 적재

`CVE-2017-5754`

비순차실행의 취약점을 이용하여 읽을 권한이 없는 메모리 영역을 읽어내는 공격법이다. 이 공격은 유저 프로세스에 할당된 메모리 공간에도 커널영역이 매핑되어 있다는 점을 악용한다.

```
#1 raise_exception();
#2 // the line below is never reached
#3 access(probe_array[data * 4096]);
```

A. meltdown을 잘 설명해주는 코드1 [https://meltdownattack.com/meltdown.pdf](https://meltdownattack.com/meltdown.pdf)

1) 프로그램 실행시 먼저 #1의 raise_exception() 함수가 실행된다.

2) raise_exception() 함수는 예외(exception)를 발생시켜 프로그램은 종료된다.
(여기서는 권한없이 Kernel 메모리에 접근하여서)

3) 여기서 일반적으로 #3은 실행되지않는다. 그러나 실제로는 `비순차실행` 에 의해 #3이 #1보다 먼저 실행될 수 있다.


```
#1  ; rcx= kernel address
#2  ; rbx= probe array
#3  retry;
#4  mov al, byte [rcx]
#5  shl rax, 0xc
#6  jz retry
#7  mov rbx, qword [rbx + rax]
```
B. meltdown을 잘 설명해주는 코드2 [https://meltdownattack.com/meltdown.pdf](https://meltdownattack.com/meltdown.pdf)

그럼 이제 해커가 어떻게 Kernel의 메모리값을 유출하는지 알아보자.

1. 먼저 해커가 캐시에 있는 값을 모두 날려준다. (또는 상관없는 값으로 채워준다.)
2. 그리고 #4가 실행된다. 이는 커널 메모리 영역(*byte[rcx]*)에 대한 읽기이므로 exception이 발생한다. 그러나 exception이 발생하기전에 `비순차실행`에 의해 #4의 메모리 읽기는 이미 수행되고, #5~#7까지 수행된다. (또한 `추측실행`도 적용되어 다른 #4가 실행될 때 #5~#7도 파이프라인에서 실행되고 있다.)
3. #7이 수행되면 probe array[`커널값(rax)` * 4096]이 rbx에 load된다.
4. 이제 해커는 Cache Side Channel Attack(또는 Cache Timing Attack)라 불리는 Flush+Reload기법을 이용해 `커널값`을 알아낼 수 있다.

( 3번에서4096을 곱해주는 이유는 페이지크기가 4KB이기 때문에, 4번의 공격에서 인접한 캐시에서 값을 가져오는 것을 방지하기 위해서이다.)

### Flush+Reload기법

`Cache Side Channel Attack 또는 Cache Timing Attack`

이제 위 공격에서 어떻게 커널에 있는 값을 알아낼 것인가? 하는 부분이다.
먼저 캐시에 대한 선행지식이 필요하다.

`캐시 메모리는 CPU안의 작고 빠른 메모리` 이다. CPU가 메인 메모리에서 데이터를 읽거나 쓸때마다 해당 데이터는 캐쉬 메모리 안에 임시로 저장된다. 모든 메모리 데이터의 접근 순서는 다음과 같다.

- CPU는 캐시 메모리에서 데이터를 먼저 찾는다. 데이터가 있다면 메인 메모리에 접근하지 않는다.
- 캐시 메모리에 데이터가 없으면 메인 메모리에서 데이터를 가져온다.
- 캐시 메모리가 메인 메모리보다 접근이 빠르다.

현재 위 공격을 수행하고 난 시점에서 캐시에는 `probe array[커널값(rax) * 4096]` 만 올라가있는 상태이다. 즉 캐시에는 `rax에 해당되는 페이지만` 올라가있는 것이다.
```c
access(probe_array[data * 4096]);
```
그렇다면 이제 probe_array의 0~255까지 모든 페이지를 랜덤하게 읽어오는 작업을 수행한다. 그리고 그 속도를 측정해보면 유난히 한 페이지만 불러오는 속도가 빠르고, 이 페이지번호가 바로 `커널값(rax)` 라는 것을 알 수 있다.

![]({{ site.baseurl }}/images/myria/Meltdown-and-Spectre/graph.jpg)

0~255 번째 페이지를 모두 Load했을 때의 Access 속도를 나타낸 그래프[https://meltdownattack.com/meltdown.pdf](https://meltdownattack.com/meltdown.pdf)

### Page Table

- `Page Table은 Virtual Address를 Physical Address로 변환하는데 사용`
- Kernel과 User Process들 각각 page table을 가지고 있다.


### 해결방법 / 현재 패치들

멜트다운은 이론 상 문제가 되는 비순차적 명령어 처리 기술이 적용된 모든 인텔 CPU에서 발생할 수 있다. 이는 커널 모드와 유저 모드를 구별하지 않도록 설계된 것이 발생 원인이다.



- 인텔은 아마존웹서비스(AWS), 마이크로소프트(MS), 구글, 레드햇 등과 협력해 멜트다운을 해결할 수 있는 패치를 개발
    - 비순차적 명령어 처리 기술을 비활성화하는 패치
    - 비활성화에 따라 CPU의 성능이 저하되는 문제가 발생(최대 30%까지 저하)

- KPTI(Kernel Page-Table Isolate)패치 = 커널페이지 테이블 격리 패치
    - 커널에서 사용하는 메모리 영역과 일반 프로그램이 사용하는 메모리 영역을 운영체제 차원에서 격리하여 사용하도록 커널 패치
    - KPTI를 사용하지 않던 기존에는 유저 프로세스가 사용하는 Page Table에도 Kernel Address가 매핑되어있어 인터럽트를 받으면 바로 해당되는 데이터를 꺼내 쓸 수 있기 때문에 부하(overhead)를 줄일 수 있다는 장점이 있었는데, 이를 분리하면서 유저 프로그램과 커널 간의 컨텍스트 전환에 더 많은 자원이 소모되어 성능에 영향을 미치게 되었다.

## 스펙터 (Spectre)

<center><img src="../images/myria/Meltdown-and-Spectre/spectre.png" width="200"></center>

- CVE-2017-5753
- CVE-2017-5715
- 한 유저 프로그램이 다른 유저 프로그램 메모리를 훔쳐보는 취약점
- 분기 예측을 적용한 최근 마이크로프로세서가 모두 영향권
- 멜트다운과 달리 x86과 ARM 아키텍처에 전부 적용된다.
- 멜트다운보다 덜 치명적이고 구현하기 어렵지만 막기도 어렵다.

분기예측(Branch Prediction) 기능에서 발생한 문제
CVE-2017-5753('bounds check bypass' 경계검사 우회)
CVE-2017-5715('branch target injection' 분기표적 주입)

### 분기 예측(Branch prediction)

`성능 최적화를 위해서 분기 위치 예측하는 기법`

원래는 분기문이 나올 때마다 분기 조건의 값을 확인하여 다음에 실행할 명령어의 위치 (분기 위치, Branch Target)를 계산하고 그 곳으로 이동한다. 분기 예측 방식에서는 각 분기문 별로 분기할 위치를 미리 예측하여 BTB(Branch Target Buffer)라는 곳에 저장해둔다. 그리고 분기문을 실행할 때 `추측실행` 이 가능한 경우에는 저장되어 있던 예측된 위치를 BTB에서 가져와 실행한다.

`추측실행` 과 상관없이 정확한 분기 위치는 매번 계산되므로 만약 추측실행이 잘못된 경우에는 실행결과는 폐기된다.

분기 위치의 예측은 직전에 그 분기문이 실행될 때의 분기 위치를 참고해서 그대로 정하는 경우가 많다.
⇒ 지난 분기에서 여기로 분기했으니 이번에도 같은 것이다.

### Bound Check Bypass, 경계검사 우회

`CVE-2017-5753`

`추측실행 기법` 의 취약점으로 배열(array)범위를 벗어난 값을 읽어내는 기법이다.

    #1  if (x < array1_size)
    #2      y = array2[array1[x] * 256];

`추측실행` 에 의해서 #2가 #1이 조건을 검사하기전에 실행될 수 있다. x의 값이 array1 배열의 범위를 넘어가는 값으로 조작되어 있어도 추측실행이 되어 그 값이 읽혀진다.

### Branch Target Injection, 분기표적 주입

`CVE-2017-5715`

이 공격은 `추측실행` 의 한 가지 유형이라고 할 수 있는 `분기 예측(Branch prediction)` 을 이용한다.
`분기 예측` 에서는 분기문을 만났을 때 다음으로 이동할 분기 위치를 계산하기 전에 예측된 위치를 사용하는데, 해커가 분기 위치의 예측과정을 조작하여 해커가 원하는 위치로 분기하도록 만들수 있다.

- 해커의 프로세스가 공격 코드가 있는 위치로 분기를 반복하면, 분기 예측 버퍼(BTB)에 그 위치가 들어가게 된다. 분기 예측 버퍼(BTB)는 여러 프로세스가 공유할 수 있기 때문에 희생자 프로세스가 분기문을 실행할 때 해커 프로세스에 의해서 BTB에 들어 있는 위치로 분기하게 된다.

### 스펙터(Spectre) 공격

    #1  if (x < array1_size)
    #2      y = array2[array1[x] * 256];

1. #1의 조건이 참이 되도록 x를 설정하여 코드를 여러번 반복시켜 분기예측버퍼(BTB)에 조건문이 참인 위치를 넣는다. (`Branch Target Injection`)
2. array1가 타겟 프로그램의 메모리 영역을 향하도록 x값을 설정한다. (`Bound Check Bypass`)
3. #1의 조건이 거짓이지만 `분기예측` 에 의해 #2가 실행된다.
4. 3번에서 명령실행 결과 array1의 값이 캐시에 올라간다.
5. Flush+Reload기법을 이용해 타겟 프로그램의 메모리값을 추출한다.

### 스펙터 공격의 어려움

- 기본적으로 공격이 되는 대상 프로그램에서 문제가 존재하는 코드의 추측 실행이 되어야 하기 때문에, 실제 버그를 이용해 공격하려면 공격하려는 대상 프로그램을 세세하게 분석해야 된다.
- 프로그램이 돌아가는 CPU 아키텍쳐나 운영체제에 대해서도 자세히 알아야된다. 그래서 실제로는 취약점을 공격하긴 매우 어렵다.

그래서 현재까지 실질적으로 해당 취약점의 영향을 받는 곳은 런타임에 JIT 컴파일러에 의존하는 자바스크립트 엔진과 그 웹브라우저들 뿐이다.
⇒ 그래도 현재 모든 CPU에서는 발생가능하다. 그러나 문제를 발견해 막기도 힘들다.

스펙터가 분기예측이라는 방법론 자체의 문제이기 때문에  멜트다운과 같이 소프트웨어 업데이트로는 사실상 완전히 막을 수 없다.


====================

`iOS`보안 공부를 하면서 `iOS`에서도 `Meltdown`과 `Spectre`때문에 보안패치를 하였다는 것을 알게되었다. 이에 이 내용을 추가하여 정리해두었다.


## iOS에서의 멜트다운, 스펙터 대처

iOS기기에 사용되는 ARM 기반 CPU 는 위 멜트다운과 스펙터에 영향을 받는다.

### iOS 11.2

- 대상: iPhone 5s 및 이후 모델, iPad Air 및 이후 모델, iPod touch (6th generation)
- 영향: 응용 프로그램에서 커널 메모리를 읽을 수 있음(Meltdown)
- 설명: 예측 실행과 간접 분기 예측 기술을 활용하는 마이크로프로세서를 탑재한 시스템에서 로컬 사용자 권한을 가진 공격자가 데이터 캐시의 사이드 채널 분석을 통해 무단으로 정보를 입수할 수 있습니다.

### iOS 11.2.2

- 대상: iPhone 5s 및 이후 모델, iPad Air 및 이후 모델, iPod touch (6th generation)
- 설명:  Safari 및 WebKit에 대해 Spectre 효과를 완화시키는 보안 개선(CVE-2017-5753 and CVE-2017-5715).

특히 Spectre는 Mac 시스템이나 iOS 기기의 로컬로 실행 앱으로는 악용하기가 매우 어려운 반면, 웹 브라우저(Safari 등)에서 실행되는 JavaScript를 통해서는 악용될 소지가 다분하다.