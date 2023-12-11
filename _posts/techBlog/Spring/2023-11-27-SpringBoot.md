---
layout: post
tags: [SpringBoot, blog]
---

### 스프링 부트

- 스프링을 편리하게 사용할 수 있도록 지원, 최근에는 기본으로 사용
- 단독으로 실행할 수 있는 스프링 애플리케이션을 쉽게 생성
- Tomcat 같은 웹 서버를 내장해서 별도의 웹서버를 설치하지 않아도 됨
- 손쉬운 빌드 구성을 위한 starter 종속성 제공
- 스프링과 3rd party(외부) 라이브러리 자동 구성
- 메트릭(모니터링), 상태확인, 외부 구성 같은 프로덕션 준비 기능 제공
- 관례에 의한 간결한 설정

<br>

### 스프링의 진짜 핵심

- 스프링은 자바 언어 기반의 프레임워크
- 자바 언어의 가장 큰 특징 - 객체 지향 언어
- 스프링은 객체 지향 언어가 가진 강력한 특징을 살려내는 프레임워크
- 스프링은 좋은 객체 지향 애플리케이션을 개발할 수 있게 도와주는 프레임워크

<br>

### 좋은 객체지향 프로그래밍이란

객체지향 프로그래밍

- 객체 지향 프로그래밍은 컴퓨터 프로그램을 명령어의 목록으로 보는 시각에서 벗어나 여러개의 독립된 단위, 즉 "객체"들의 모임으로 파악하고자 하는것
- 각각의 객체는 메시지를 주고받고, 데이터를 처리할 수 있다.(협력)
- 객체 지향 프로그래밍은 프로그램을 유연하고 변경이 용이하게 만들기 때문에 대규모 소프트웨어 개발에 많이 사용됨

역할과 구현의 분리

- 역할과 구현으로 구분하면 단순해지고, 유연해지며 변경도 편리해진다.
- 장점
  - 클라이언트는 대상의 역할(인터페이스)만 알면 된다.
  - 클라이언트는 구현 대상의 내부 구조를 몰라도 된다.
  - 클라이언트는 구현 대상의 내부 구조가 변경되어도 영향을 받지 않는다.
  - 클라이언트는 구현 대상 자체를 변경해도 영향을 받지 않는다.

자바 언어에서는

- 자바 언어의 다형성을 활용
  - 역할 = 인터페이스
  - 구현 = 인터페이스를 구현할 클래스, 구현 객체
- 객체를 설계할 때 역할과 구현을 명확히 분리
- 객체 설계시 역할(인터페이스)을 먼저 부여하고, 그 역할을 수행하는 구현 객체 만들기

스프링과 객체 지향

- 스프링은 다형성을 극대화해서 이용할 수 있게 도와준다.
- 스프링에서 이야기하는 제어의 역전(IoC), 의존관계 주입(DI)은 다형성을 활용해서 역할과 구현을 편리하게 다룰 수 있도록 지원한다.

<br>

### 좋은 객체 지향 설계의 5가지 원칙 (SOLID)

- SRP : 단일 책임 원칙
- OCP : 개방-폐쇄 원칙
- LSP : 리스코프 치환 원칙
- ISP : 인터페이스 분리 원칙
- DIP : 의존관계 역전 원칙

#### SRP 단일 책임 원칙

- 한 클래스는 하나의 책임만 가져야 한다.
- 하나의 책임이라는 것은 모호하다.
- 중요한 기준은 변경, 변경이 있을 때 파급 효과가 적으면 단일 책임 원칙을 잘 따른것

#### OCP 개방-폐쇄 원칙

- 소프트웨어 요소는 확장에는 열려 있으나 변경에는 닫혀있다.
- 다형성을 활용하자
- 인터페이스를 구현한 새로운 클래스를 하나 만들어서 새로운 기능을 구현
- 역할과 분리를 생각하자
- 문제점
  - MemberService 클라이언트가 구현 클래스를 직접 선택해야 한다.
  ```
  MemberRepository m = new MemoryMemberRepository() // 기존코드
  MemberRepository m = new JdbcMemberRepository()   // 변경코드
  ```
  - 구현 객체를 변경하려면 클라이언트 코드를 변경하애 한다.
  - 분명 다형성을 사용했지만 OCP 원칙을 지킬수 없다.
  - 이 문제는 어떻게 해결하나?
    - 객체를 생성하고, 연관관계를 맺어주는 별도의 조립, 설정자가 필요 (스프링이 이 과정을 도와준다.)

#### LSP 리스코프 치환 원칙

- 프로그램의 객체는 프로그램의 정확성을 깨뜨리지 않으면서 하위 타입의 인스턴스로 바꿀수 있어야 한다.
- 다형상에서 하위 클래스는 인터페이스 규약을 다 지켜야 한다는 것, 다형성을 지원하기 위한 원칙, 인터페이스를 구현한 구현체는 믿고 사용하려면 이 원칙이 필요하다.
- ex) 차량의 엑셀은 앞으로 가도록 만들어야한다. 뒤로가면 안된다

#### ISP 인터페이스 분리 원칙

- 특정 클라이언트를 위한 인터페이스 여러 개가 범용 인터페이스 하나보다 낫다.
- 자동차 인터페이스 -> 운전 인터페이스, 정비 인터페이스로 분리한다.
- 분리하면 정비 인터페이스 자체가 변해도 운정자 클라이언트에 영향을 주지 않음

#### DIP 의존관계 역전 원칙

- 프로그래머는 추상화에 의존해야지, 구체화에 의존하면 안된다. 의존성 주입은 이 원칙을 따르는 방법 중 하나다.
- 역할에 의존해야 된다는 것

스프링은 다음 기술로 다형성 + OCP, DIP를 가능하게 지원

- DI(Dependency Injection): 의존관계, 의존성 주입
- DI 컨테이너 제공

클라이언트 코드의 변경 없이 기능 확장