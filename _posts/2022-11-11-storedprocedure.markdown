---
layout: post
title: Oracle 19c에서 stored procedure 조회가 비정상적으로 오래걸리는 문제
date: 2024-2-10 17:20:23 +0900
category: java
---

기존에 잘 사용하고 있던 java.sql.DatabaseMetaData클래스의 getProcedures메서드가 oracle 19c에서 굉장히 오래걸리는 문제가 발생하였다.

확인은 19c와 11g를 하였고 같은 파라미터를 주고 실행하였을 대 속도 차이는 아래와 같다.

- oracle 19c : 26,711개 약 19분

- oracle 11g : 25,208개 약 11초


getProcedures는 인자로 catalog, schema, produreName을 받는데 catalog를 null로 주면 현상이 발생한다.(null은 전체를 의미하는 값으로 사용되어왔다.)

다른데서 이슈가 있었는지 조사하는 과정에서 oracle의 아래 설명을 확인하였다.

```
catalog
  Oracle does not have multiple catalogs, but it does have packages. Consequently, the catalog parameter is treated as the package name.
  This applies both on input, which is the catalog parameter, and the output, which is the catalog column in the returned ResultSet.
  On input, the construct " ", which is an empty string, retrieves procedures and arguments without a package, that is, standalone
  objects.
  A null value means to drop from the selection criteria, that is, return information about both standalone and packaged objects.
  That is, it has the same effect as passing in the percent sign (%).
  Otherwise, the catalog parameter should be a package name pattern, with SQL wild cards, if desired.
  ```

설명을 보니 null과 %가 비슷한 효과로 사용되는 것 같지만 완전히 동일한 의미는 아닌 것으로 보인다.

테스트 결과 %를 사용하면 속도가 다시 빨라지는데 조회된 Procedures의 갯수가 줄어든다.

추가 조사결과 null을 넣었을 때 여러유저의 Procedures를 조회하는데 일부 유저의 Procedures를 조회하는 과정에서 시간이 오래걸리는 것을 확인하였다.(이 중 SYS가 가장 오래걸림)

이 이슈는 많은 고민을 하였지만 근본적인 해결은 하지 못하고 기본으로는 SYS의 Procedures는 조회하지 않고 필요할 경우 조회가가능하도록 옵션을 제공하였다.