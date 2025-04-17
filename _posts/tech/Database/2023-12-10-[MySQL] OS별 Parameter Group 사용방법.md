---
categories: [Tech, Database]
tags: [Parameter Group, Database, blog]
---

### lower_case_table_names

`lower_case_table_names` 은 MySQL의 테이블 이름의 대소문자를 구분하는 설정이다.

```sql
-- 테이블명에 대한 대소문자를 구분한다.
lower_case_table_names = 0

-- 테이블명을 모두 소문자로 변환하지만 대소문자를 구별하지 않는다. MYSQL이 storage나 lookup의 모든 테이블이름을 소문자로 변환한다. 또한 데이터베이스명이나 alias에서도 적용된다.
lower_case_table_names = 1

-- 테이블명이나 데이터베이스명을 모두 대문자로 저장하지만, lookup시 소문자로 변환한다. 명칭비교는 대소문자를 구별하지 않는다.
lower_case_table_names = 2
```

default로 Unix는 0, Window는 1, MacOS는 2이다.

운영서버로 리눅스를 사용하며, 리눅스에 MySQL을 설치한다면 `lower_case_table_names`의 기본설정은 `0`이다. 물론 MySQL의 설정을 `1`로 변경하여 사용해도 된다. AWS의 RDS를 사용시 인스턴스의 parameter group을 `0`으로 생성했다면 중간에 수정이 불가능하다. parameter group을 다시 생성하여 적용시켜야 lower_case_table_names가 적용된다.

출처

- [https://dev.mysql.com/doc/refman/8.0/en/identifier-case-sensitivity.html](https://dev.mysql.com/doc/refman/8.0/en/identifier-case-sensitivity.html)
