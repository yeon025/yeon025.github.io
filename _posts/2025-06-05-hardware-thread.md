---
title: "[컴퓨터 구조] 하드웨어 스레드"
author: 
date: 2025-06-05 16:35:00 +0900
categories: [Computer Science]
---
![danawa](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FUP2Cn%2FbtsOolvQmrh%2FNmhCjqWuZZKVk2r5bbCFcK%2Fimg.png)
[[컴퓨터 구조]CPU 코어](https://yeon025.github.io/posts/cpu-core/) 에서는 `8Core 16Thread`의 Core에 대해 설명했다.
이번 글은 Thread에 대해 설명해보려고 한다.

<br>
<br>

## 하드웨어 스레드
하드웨어 스레드는 CPU 코어가 실제로 명령어를 실행하는 물리적인 경로이다.<br>
CPU 코어 하나는 하이퍼스레딩(Hyper-Threading)과 같은 기술을 통해 두 개의 하드웨어 스레드를 가질 수 있다. 이를 통해 두 개의 작업 흐름을 번갈아가며 실행할 수 있다.<br>
따라서 하드웨어 스레드는 코어가 처리해야 할 작업 단위이며, 운영체제 입장에서는 각 하드웨어 스레드가 하나의 독립된 실행 유닛처럼 동작한다.

<br>
<br>

## 하이퍼스레딩
하이퍼스레딩은 하나의 물리적인 CPU 코어가 두 개의 논리적인 코어(스레드)처럼 동작하게 만들어 병렬 처리 성능을 향상시키는 기술이다.<br>
명령어 실행 중 CPU의 일부 자원이 대기 상태에 놓이는 일이 많다. 
하이퍼스레딩은 이 유휴 시간을 줄이고 다른 스레드를 동시에 실행시켜 CPU의 효율을 높이는 방식이다.

<br>

![task_manager_cpu](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fblog.kakaocdn.net%2Fdn%2FzVxXx%2FbtsOtf09qDz%2FJkXA1oYB9qTw2lcmCipmS1%2Fimg.png)
사진을 보면 4개의 코어가 있고, 각 코어당 2개의 하드웨어 스레드가 동작하여 총 8개의 하드웨어 스레드가 있는 것을 확인할 수 있다. 작업 관리자에서는 CPU 그래프가 8개로 표시되어, 하이퍼스레딩을 시각적으로 보여준다.