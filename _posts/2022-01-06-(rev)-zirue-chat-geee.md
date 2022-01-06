---
title: (REV) zirue chat geee
categories: [REV]
tags : [mineswipper]
date : 2022-01-06 20:03:44 +0900
author:
    name: 
    link: 
toc: false
comments: false
mermaid: false
math: false
---

## 목차

- 찾은 개수 카운팅 변수 찾고 어디서 access 하는지 확인
- 지뢰 그려주는 함수 찾기
- 코드인젝션

---

---

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled.png)

일단, 치트엔진으로 분석을 시작했다.

찾은 개수 카운팅하는 변수를 찾고, 어디서 access 하는지 검색했다.

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%201.png)

칸은 16*16 = 256 개고, 지뢰는 40개 이므로, 

현재 찾은 개수와 찾아야 하는 개수를 비교해 

같으면 mine.exe+347C 에 1을 넣어 호출해준다.

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%202.png)

EIP 조작해서 1넣고 호출해보면, 깃발로 모든 지뢰를 표시해준다.

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%203.png)

0을 넣어서 호출하면 지뢰가 표시되고 게임이 끝난다.

0 or 1 으로 게임승리와 종료를 판별하는 것 같다.

다음으로, 웃는얼굴을 클릭했을 때 핸들러를 찾기 위해,

지뢰를 초기화해주는 initialize (임의로 명명) 함수의 xref 를 찾아가 봤더니

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%204.png)

이렇게 생긴 함수에서 지뢰를 초기화해줫다.

코드인젝션을 통해 correct(0) 을 호출하게 해보겠다.

—> correct(0) 호출하면, 게임이 끝나는것과 마찬가지라서 얼굴은 죽은모양을 하고있고, 클릭이 안된다. 그래서 지뢰를 그려주는 함수만 호출해야한다.

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%205.png)

동적으로 확인 결과, `sub_1002F80`  함수에서 지뢰를 그려준다.

a1 이 0이되어야 하므로, 0xA 를 push 하고 호출하면 된다 (스택은 저 함수에서 정리해줌)

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%206.png)

Cheat engine 의 auto assemble 기능을 이용해 쉽게 코드인젝션 할 수 있다.

+367A 는 initialize 함수고, +2F80 이 지뢰를 그려주는 함수다.

![Untitled](/assets/img/2022-01-06-(rev)-zirue-chat-geee/Untitled%207.png)

코드 인젝션의 결과는 위와 같다. 이제 지뢰찾기 고수가 될 수 있다.