---
layout: post
title: Chrome 94 버전 이후 localhost 요청 블럭 문제
date: 2024-02-10 11:20:23 +0900
category: client
---

Chrome 94버전 이후에서 localhost로의 요청이 블럭되는 문제가 발생했다.

보안정책인 것으로 확인되었고 가장 간단한 방법은 설정 옵션을 disable로 변경하는 것이다.

#### 설정 경로 - chrome://flags/#block-insecure-private-network-requests
![chrome](/public/img/localhostblock.png)

하지만 Client들에게 모두 저런 가이드를 하는것은 현재 상황에 알맞지 않다고 판단하여 우회법을 고민해 보았다.

확인해보니 현상이 재현될 때는 일반 HTTP요청을 사용하였는데 WebSocket을 사용하면 문제가 발생하지 않는다.

이를 적용하기 위해 Server제품에 WebSocketListener를 붙이는 작업을 진행하였다.