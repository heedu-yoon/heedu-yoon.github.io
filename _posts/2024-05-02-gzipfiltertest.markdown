---
layout: post
title: gzip Filter사용 시 성능 테스트
date: 2024-05-02 19:20:23 +0900
category: was
---

WAS와 통신하는 구조에서는 Gzip Filter가 많이 사용된다.

일부 WAS는  동작이 안하기도해서 우리는 내부에서 구현한 Filter도 가지고 있는데 실제로 어떤 환경에서 가장 효율적일지 생각하다가 간략하게 성능 테스트를 해보았다.


|압축전|속도|압축후|시간|
|:--:|:--:|:--:|:--:|
|3.09MB|3Mbps(Regular 4G)|122.39KB|1.13초|
|3.09MB|3Mbps(Regular 4G)|X|6.05초|
|9.27MB|3Mbps(Regular 4G)|361.21KB|2.13초|
|9.27MB|3Mbps(Regular 4G)|X|18.77초|
|18.53MB|3Mbps(Regular 4G)|719.09KB|4.03초|
|18.53MB|3Mbps(Regular 4G)|X|37.05초|
|27.79MB|3Mbps(Regular 4G)|1.05MB|6.32초|
|27.79MB|3Mbps(Regular 4G)|X|55.14초|
|37.06MB|3Mbps(Regular 4G)|1.4MB|8.68초|
|37.06MB|3Mbps(Regular 4G)|X|74.4초|
|3.09MB|1Mbps(Good 3G)|122.39KB|0.758초|
|3.09MB|1Mbps(Good 3G)|X|16.77초|

이미 예상하였지만 고속네트워크에서는 50메가정도까지는 크게 차이가 안보이고 저속으로 갈 수록 차이가 확연히 보이고 있다.

모바일 환경에서 데이터를 많이 내려받아야 하는 케이스에서는 확실한 효과를 볼 수 있을 것 같다.