+++

authors = [
    "Lena"
]
title = "동시성 프로그래밍(Concurrency programming): 스레드와 큐(Thread and Queue)"
date = 2020-11-26T22:05:34+09:00
description = "Concurrency programming: introduce thread and queue in iOS"
tags = [
    "iOS", "Concurrency programming", "GCD", "Concurrent Queue", "Serial Queue", "Sync", "Async"
]
categories = [
     "iOS", "Concurrency programming", "GCD"
]
series = ["Concurrency programming"]
images = [
  "/images/async+concurrent.jpg"
]
draft = false

+++

[동시성 프로그래밍 시리즈] **Sync** vs **Async** / **Serial** vs **Concurrent**에 대해 자세히 알아봅니다 🙌🏻<br>

<br>

<!--more-->

> **비동기(Async)와 동시(Concurrent)는 같은 말일까요?** 

이 질문에 대한 답을 찾아가봅시다.

이 질문에 정확히 답변을 하기 위해서는 비동기란 무엇인지, 동시란 무엇인지 이해하고 있어야 하겠죠? 아래 본문에서는 이런 내용을 다룰 것입니다.



##    <  📑 목차  >

1. 알아두기
2. 큐(Queue)(대기열/대기행렬)
   2.1. 큐 소개
   2.2. 큐의 종류
3. Synchronous(동기) vs Asynchronous(비동기)
4. Serial(직렬) vs Concurrent(동시)
5. 스레드의 작업 처리 방식과 큐의 작업 분산 방식 조합
6. 정리
7. 함께보면 좋을 자료들
8. 참고



<br>

## <span style="color: #6666FF">알아두기</span>

* 먼저 스레드에 대한 개념을 알고 읽으면 좋을 것같습니다.(사실 반드시 스레드에 대해서 알고 있는 상태에서 이후 내용을 읽으셨으면 좋겠습니다 👻) - 위키피디아 링크: [스레드(Thread)](https://en.wikipedia.org/wiki/Thread_(computing))

* 동시성 프로그래밍의 시작은 "수행해야 할 작업(Task)들을 한 개의 스레드에서만이 아니라 다른 스레드에서도 어떻게 동시에 일을 시킬 수 있을까?" 라는 아이디어에서 시작합니다.

* 큐와 스레드를 이해하기에 앞서 [공식문서](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html#//apple_ref/doc/uid/TP40008091-CH100-SW1)에 있는 **GCD**(*Grand Central Dispatch*)에 대한 개념을 알고 갑시다.<br>
  (추가로 [이 자료](https://hcn1519.github.io/articles/2018-05/concurrent_programming)도 한 번 읽어보시면 좋을 것 같습니다 :) )

  > One of the technologies for starting tasks asynchronously is ***Grand Central Dispatch (GCD)***. This technology takes the thread management code you would normally write in your own applications and moves that code down to the system level. All you have to do is define the tasks you want to execute and add them to an appropriate _**dispatch queue**._ **GCD takes care of creating the needed threads and of scheduling your tasks to run on those threads**._ Because _the thread management is now part of the system_, **GCD provides a holistic approach to task management and execution, providing better efficiency than traditional threads.**<br>
  > **출처**: [Concurrency and Application Design](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html#//apple_ref/doc/uid/TP40008091-CH100-SW1)

  >   In the past, if an asynchronous function did not exist for what you want to do, you would have to write your own asynchronous function and create your own threads. But now, OS X and iOS provide technologies to allow you to perform any task asynchronously **_without having to manage the threads yourself._**<br>
  > **출처**: [Concurrency and Application Design](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html#//apple_ref/doc/uid/TP40008091-CH100-SW1)

  

  **직접적으로 스레드를 관리하지 않고 큐(Queue) 개념을 사용하여 작업을 분산 처리합니다. iOS에서는 작업을 큐(Queue)에 보내면 OS가 알아서 다른 스레드로 분산처리를 해줍니다. 즉, 개발자가 스레드를 직접적으로 만들어서 큐에 작업을 넣어주지 않아도 됩니다. 개발자는 큐를 만들어서 큐에 작업을 넣어주고, 그러면 GCD가 스레드 생성과 스케줄링 관련된 일을 모두 담당하여 스레드에 작업을 배치합니다. GCD가 스레드보다 더 높은 레벨/차원에서 일을 한다고 이해하면 좋을 것 같네요.**

<br>

## <span style="color: #6666FF">큐(Queue)(대기열/대기행렬)</span>

<br>

GCD는 스레드 관리를 해주면서 개발자가 코드로 작성한 작업을 시스템 레벨에서 동작하도록 해주고 개발자는 수행할 작업을 큐에 등록 하면 됩니다. GCD가 쓰레드 생성과 스케줄링 관련된 일을 모두 담당합니다.

그럼 작업을 등록할 큐에는 어떤게 있을까요? <br>

큐는 크게 `Dispatch Queue`, `Dispatch Sources`, `Operation Queue`가 있습니다. (이번 글에서는 `Dispatch Queue` 에 대한 내용만 다룹니다.)

#### <span style="color:orange">1. 큐 소개</span>

* 큐는 항상 선입선출(FIFO: First In First Out)로 동작합니다. 
* 먼저 들어온 작업이 다른 스레드에 먼저 배치가 되지만 그렇다고 먼저 끝나는 것이 보장하는 것은 아닙니다.  

 <br>

### _즉, 2개 이상의 스레드를 사용하여 작업을 처리하고 싶을 때 큐에 작업을 보내면 됩니다!_

<br>

그렇다면 작업을 어떻게 큐에 보낼까요?

**_DispatchQueue_ 를 사용하는 방법이 있습니다.** 

```swift
DispatchQueue.global().async{
  // 작업1
  // 이 클로저 안에 있는 코드들이 작업의 한 단위라고 볼 수 있습니다.
} // 작업1을 글로벌큐에 비동기적으로 보낼꺼야.라는 의미입니다.

// 위의 코드를 아래와 같이 표현할 수도 있습니다.
let queue = DispatchQueue.global() 
queue.async { 
  
} 
```

**Dispatch**의 사전적인 의미는 ***보내다, 파견하다***입니다. 그러니까 **DispatchQueue**는 *큐에 보내다* 정도로 이해할 수 있겠네요.

> Dispatch queues are a C-based mechanism for executing custom tasks.<br>
> **출처**: [Concurrency and Application Design](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html#//apple_ref/doc/uid/TP40008091-CH100-SW1)

<br>

#### <span style="color:orange">2. 큐의 종류</span>

큐는 여러 종류가 있습니다. 그리고 그 큐마다 특성이 다릅니다. <br><span style="color:orange">원하는 큐로 작업을 보내면 시스템에서 큐의 특성에 따라 **스레드를 생성하고 일을 처리합니다.** </span>



| 메인큐                                                       | Global Queue(글로벌큐)                                       | Private Queue(프라이빗큐)                                    |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1. 한 개<br />2. **직렬(Serial)**<br />3. 메인 스레드<br />4. `DispatchQueue.main.async{}` | 1. 종류 여러 개<br />2. 기본 설정 동시(Concurrent)<br />3.중요도에 따라 서비스 품질(QoS)을 설정한 큐 생성 가능. <br />4. `DispatchQueue.global()` 또는 `DispatchQueue.global(qos:) ` | 1. 커스텀으로 만듦.<br />2. 기본 설정 직렬(Serial)(동시(Concurrent) 설정 가능)<br />3. QoS 설정도 가능<br />4. `DispatchQueue(label:)` |

각 큐에 대한 내용은 다음에 자세히 다루겠습니다. 여기서는 메인큐에 대한 내용을 알아두면 될 것 같습니다. ([다음 글](https://lena-chamna.netlify.app/post/concurrency_programming_caution_when_using_dispatchQueue/)에 관련 내용이 나옵니다)

<br><br>

## <span style="color: #6666FF">Synchronous(동기) vs Asynchronous(비동기)</span>

작업을 시키는 스레드에서 작업들을 다른 스레드로 분산처리 시키고 그 작업이 끝날때 까지 기다릴지 기다리지 않을지에 대한 개념입니다.

| <span style="color:orange">비동기(Async)</span>              | <span style="color:orange">동기(Sync)</span>                 |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 작업을 작업을 시키는 스레드가 아닌 다른 스레드에서 하도록 시킨 후,<br /> 그 작업이 끝나길 **기다리지 않고** 다음 작업을 진행합니다. <br>(기다리지 않아도 다음 작업을 생성할 수 있습니다.) | 작업을 작업을 시키는 스레드가 아닌 다른 스레드에서 하도록 시킨 후, <br />그 작업이 끝나길 **기다렸다가** 작업이 완료되면 다음 작업을 진행합니다. <br/>(기다렸다가 다음 작업을 생성할 수 있습니다.) |

<br>

* **비동기(Async)**

  ```swift
  DispatchQueue.global().async{
  
  }
  // 원래의 작업이 진행되고 있던 곳에서 디스패치 글로벌 큐로 작업을 보낸 후 
  // 그 작업이 끝날때 까지 기다리지 않습니다.
  ```

  참고로 iOS에서 네트워크 통신을 위해 사용하는 `URLSession(configuration:).dataTask(with:)` 은 내부적으로 비동기 처리가 되어있습니다. 그래서 이 메서드를 사용하면 내부적으로 알아서 다른 스레드를 생성해서 작업을 처리하고 있다고 이해하면 됩니다.

  ```swift
  URLSession(configuration: .ephemeral).dataTask(with: url){
    data, response, error in 
    ....
  }
  ```

* **동기(Sync)**

  ```swift
  DispatchQueue.global().sync{
  
  }
  // 원래의 작업이 진행되고 있던 곳에서 디스패치 글로벌 큐로 작업을 보낸 후 
  // 그 작업이 끝날 때까지 동기적으로 기다립니다.
  ```


<br><br>

## <span style="color: #6666FF">Serial(직렬) vs Concurrent(동시)</span>

이 두 개념은 **큐의 특성**에 대한 개념입니다. 

* **Serial**

시리얼큐는 큐가 받아들인 작업들을 **하나의 스레드로만 보내 처리하는 큐**입니다. 시리얼큐는 _순서가 중요한 작업_ 을 처리할 때 사용합니다.

* **Concurrent**

컨커런트큐는 받아들인 작업을 **여러 개의 스레드로 나눠서 보내 처리하는 큐**입니다. 이 때 몇 개의 쓰레드로 분산할지는 위에서 언급했듯시스템(OS)이 알아서 결정합니다. 컨커런트큐는 각자 _중요도나 작업 성격이 독립적이지만 유사한 여러 개의 작업을 처리할 때_ 사용합니다. (예를 들면 테이블 뷰 각 셀에 들어가는 이미지나 텍스트 데이터를 API에서 가지고 올 때가 있습니다.)

<br><br>

## <span style="color: #6666FF">스레드의 작업 처리 방식과 큐의 작업 분산 방식 조합</span>

음.. 그럼 4개의 개념은 각각 정리가 되었네요. 그러면 위 개념들을 조합해서는 어떨까요?

|                                                        | <span style="color:orange">Sync(동기)</span>                 | <span style="color:orange">Async(비동기)</span>              |
| ------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| <span style="color:orange">**Serial(직렬)**</span>     | 작업을 다른 스레드에서 하도록 분산처리한 후 그 작업이 끝나길 <span style="color:brown">**기다렸다가**</span> 다음 작업을 큐에 넣습니다. <br>이 때 다른 스레드에 작업을 분산하는 큐의 특성은 Serial입니다. 즉, 작업을 시키는 스레드(A)에서 분산처리 시킨 작업을 해당 스레드(A)가 아닌 <span style="color:brown">**다른 한 개의 스레드(B)에서**</span> 처리하도록 합니다(분배합니다.) | 작업을 다른 스레드에서 하도록 분산처리한 후 그 작업이 끝나길 <span style="color:brown">**기다리지 않고**</span> 다음 작업을 큐에 넣습니다.<br/>이 때 작업을 다른 스레드에 분산하는 큐의 특성은 Serial입니다. 즉, 작업을 시키는 스레드(A)에서 분산처리 시킨 작업을 해당 스레드(A)가 아닌 <span style="color:brown">**다른 한 개의 스레드(B)에서**</span> 처리하도록 합니다(분배합니다.) |
| <span style="color:orange">**Concurrent(동시)**</span> | 작업을 다른 스레드에서 하도록 분산처리한 후 그 작업이 끝나길 <span style="color:brown">**기다렸다가**</span> 다음 작업을 큐에 넣습니다. <br/>이 때 작업을 다른 스레드에 작업을 분산하는 큐의 특성은 Concurrent 입니다. 즉, 작업을 시키는 스레드(A)에서 분산처리 시킨 작업을 해당 스레드(A)가 아닌 <span style="color:brown">**다른 여러 개의 쓰레드(B,C,D...)에서**</span> 처리하도록 합니다(분배합니다.) | 작업을 다른 쓰레드에서 하도록 분산처리한 후 그 작업이 끝나길 <span style="color:brown">**기다리지 않고**</span> 다음 작업을 큐에 넣습니다. <br/>이 때 다른 스레드에 작업을 분산하는 큐의 특성은  Concurrent입니다. 즉, 작업을 시키는 스레드(A)에서 분산처리 시킨 작업을 해당 스레드(A)가 아닌 <span style="color:brown">**다른 여러 개의 쓰레드(B,C,D...)에서**</span> 처리하도록 합니다.(분배합니다.) |

그림으로 보면 이렇습니다.

|                | Sync                                                         | Async                                                        |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **Serial**     | <img src="https://i.imgur.com/hRuqzM1.png" style="zoom:60%;" /> | <img src="https://i.imgur.com/qt4qV7K.png" style="zoom:60%;" /> |
| **Concurrent** | <img src="https://i.imgur.com/ou8BToH.png" style="zoom:60%;" /> | <img src="https://i.imgur.com/Mro0GTL.png" style="zoom:60%;" /> |



## <span style="color: #6666FF">정리</span>

그럼 다시 글 초반에 언급했던 **비동기(Async)와 동시(Concurrent)는 같은 말일까요?** 이 질문에 대해 생각해봅시다.

저도 공부하면서 이렇게 정리하기 전에는 항상 정확히 누구에게 설명하기에는 아리송하면서 헷갈렸는데요. 이번에 정리하면서 확립이 많이 됐습니다.

정리하자면 이렇습니다.

<span style="color:orange">***동기 / 비동기 개념은 작업을 보내는 시작점에서 작업이 끝날때 까지 기다릴지 말지에 대해 다루는 것입니다.*** <br>***한편 직렬 / 동시 개념은 큐로 보낸 작업들이 여러 개의 스레드로 갈 것인지 아니면 하나의 스레드로만 갈 것인지에 대해 다루는 것입니다.***</span>

그러니까 둘은 다른 개념이고, 둘이 같은 말인지 묻는다면 다르다고 답해야 하는거죠 :)

<br>

## <span style="color: #6666FF">함께 보면 좋을 자료들</span>



#### 👉🏻 [동시성 프로그래밍(Concurrency programming): dispatchQueue 사용시 주의할 점들](https://lena-chamna.netlify.app/post/concurrency_programming_caution_when_using_dispatchQueue/)



(기록해 놨다가 저도 나중에 보려고 씁니다 🐝)

1. [Concurrency Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/Introduction/Introduction.html)

2. [Mastering Grand Central Dispatch - WWDC2011](https://developer.apple.com/videos/play/wwdc2011/210/)

3. [Concurrent Programming With GCD in Swift 3 - WWDC2016](https://developer.apple.com/videos/play/wwdc2016/720)

4. [Modernizing Grand Central Dispatch Usage - WWDC2017](https://developer.apple.com/videos/play/wwdc2017/706/)

   

<br>

## <span style="color: #6666FF">참고</span>

1. [DispatchQueue](https://developer.apple.com/documentation/dispatch/dispatchqueue)
2. [iOS Concurrency(동시성) 프로그래밍, 동기 비동기 처리 그리고 GCD/Operation](https://www.inflearn.com/course/iOS-Concurrency-GCD-Operation)
3. [Concurrency Programming Guide - Concurrency and Application Design](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/ConcurrencyandApplicationDesign/ConcurrencyandApplicationDesign.html#//apple_ref/doc/uid/TP40008091-CH100-SW1)

<br>

**피드백 주신 JK 감사합니다 🙏🏻**

