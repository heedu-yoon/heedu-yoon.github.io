---
layout: post
title: Access-Control-Allow-Origin header 여러개 설정하기
date: 2024-03-02 19:20:23 +0900
category: was
---

Cross Domain환경에서 필요한 Access-Control-Allow-Origin은 * 또는 1개의 명시적인 Domain만 설정이 가능하다.

종종 이를 Server에서 여러개 설정하고 싶은 경우가 있을 수 있다.

정책상으로는 불가능하지만 프로그래밍 적으러 어느정도 요구사항을 충족할 수있다.



간략한 순서는 아래와 같다.

1. Server에서 복수의 domain 설정.
1. Request의 origin Header를 확인하고 해당 헤더가 설정된 domain중에 존재하는지 확인.
1. 존재할 경우 Access-Control-Allow-Origin에 해당 domain을 설정함.

```java
String allowOrigin = "";

		if(allowOrigin.equals("*") || !allowOrigin.contains(","))
			return allowOrigin;
		
		String reqOrigin = request.getHeader("Origin");
		if(reqOrigin != null && !reqOrigin.equals("")) 	
		{
			String[] originArr = allowOrigin.split(",");
			
			for (String origin : originArr) 
			{
				if(reqOrigin.contains(origin))
				{
					allowOrigin = origin;
					break;
				}
			}
		}
		
		return allowOrigin;
```