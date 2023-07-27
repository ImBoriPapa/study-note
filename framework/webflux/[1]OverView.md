# 1.Over View

## WebFlux란
- WebFlux는 Spring Framework의 일부인 reactive-stack web framework이다.
- Web Application을 구축하기 위한 non-blocking 및 Reactive Programming Model을 제공하기 위해 Spring Framework 버전 5.0에 도입되었습니다.
- spring-webmvc와 spring-webflux 두 웹 프레임워크는 source module의 이름을 미러링하고 스프링 프레임워크에 나란히 존재
- 각 모듈은 선택사항이며 Application은 하나 또는 다른 모듈을 사용하거나 경우에 따라서 둘 다 사용할 수 있다 예를 들어 reactive WebClient와 함께 Spring MVC Controller를 사용할 수 있다.
- WebFlux는 Reactive Streams API를 기반으로 하여, 역압으로 비동기 데이터 스트림을 처리할 수 있습니다.
- WebFlux는 Netty, Underow 및 Servlet container와 같은 서버에서 실행이 된다.
- WebFlux는 완전한 non-blocking이며 Reactive Streams를 지원합니다.
- WebFlux는 높은 동시성과 많은 수의 동시 연결을 효율적으로 처리하도록 설계되었습니다.
- 이를 통해 프레임워크는 데이터 스트림을 효율적으로 관리 및 처리할 수 있으므로 실시간 데이터, 이벤트 중심 애플리케이션 및 반응형 프로그래밍 시나리오를 처리하는 데 적합합니다.

## 생성 배경
- 적은 수의 스레드로 동시성을 처리하고 더 적은 하드웨어로 리소스로 확장할 수 있는 non-blocking web stack의 필요성
- functional programming으로 Java8의 추가된 Lamda 표현식과 비동기 로직의 선언적 구성을 허용하는 non-blocking Application 및 continuation-style APIs에 이점을 활용

## 특징
- Non-Blocking: WebFlux 응용 프로그램은 I/O 작업 중에 스레드를 차단하지 않고 비동기 프로그래밍을 사용하여 Non-Blocking I/O를 수행하므로 응용 프로그램은 요청당 별도의 스레드를 필요로 하지 않고 여러 요청을 동시에 처리할 수 있습니다.

- Reactive Programming: WebFlux는 반응형 프로그래밍의 원리를 활용하여 개발자가 복잡한 비동기 작업을 선언적이고 기능적인 스타일로 구성할 수 있습니다. 이 접근 방식은 비동기 이벤트와 데이터 스트림의 처리를 단순화합니다.

- Reactive Stremes: WebFlux는 Back Pressure을 사용한 비동기 스트림 처리 표준을 정의하는 반응형 스트림 사양을 기반으로 구축되었습니다. Back Pressure을 통해 데이터 스트림 소비자는 데이터 생성 속도를 제어하여 과부하 시나리오를 방지할 수 있습니다.

- 다중 서버 지원: 웹플럭스는 Netty, Underow 및 기존 서블릿 컨테이너를 포함한 다양한 서버에서 실행할 수 있습니다. 이러한 유연성을 통해 개발자는 특정 요구에 가장 적합한 서버를 선택할 수 있습니다.

- 기능적 및 주석이 달린 프로그래밍 모델: WebFlux는 기능적 및 주석이 달린 두 가지 프로그래밍 모델을 제공합니다. 기능적 모델은 라우팅 및 처리 기능을 명시적으로 정의하는 것을 포함하고 주석이 달린 모델은 주석을 사용하여 요청 매핑 및 핸들러를 정의합니다.

- 반응형 라이브러리와의 통합: WebFlux는 다른 반응형 라이브러리와 원활하게 통합되므로 개발자는 웹 애플리케이션을 구축할 때 반응형 생태계의 잠재력을 최대한 활용할 수 있습니다.

WebFlux는 애플리케이션이 많은 동시 요청을 효율적으로 처리해야 하거나 실시간 분석, IoT 애플리케이션, 채팅 애플리케이션 등과 같은 실시간 데이터 스트림으로 작업해야 하는 시나리오에 특히 유용합니다. 이는 Spring Web MVC의 기존 서블릿 기반 접근 방식에 대한 대안을 제공하며 현대적인 고성능 웹 애플리케이션을 구축하는 데 적합합니다.

## "Reactive"의 정의
- "Reactive(반응적)"이라는 용어는 네트워크 구성 요소가 I/O 이벤트에 반응하는 것
- UI Controller가 마우스 이벤트에 반응하는 것 등 변화에 반응하는 것을 중심으로 구축된 프로그래밍 모델을 의미.
- "non-blocking"은 반응적이다 왜냐하면 blocking이 되는 대신 작업이 완료되거나 데이터가 사용 가능해지면 알림에 반응하는 방식이 되기 때문이다.
- non-blocking back pressure는 Reactive와 연관 짓는 중요한 메커니즘이다.
- synchronous, imperative한 코드가 요청을 차단하는 것은 요청자가 기다리도록 강요하는 자연스러운 형태의 back pressure의 역활을 한다.
- non-blocking 코드에서, 빠른 Producer가 목적지를 압도하지 않도록 event의 속도를 제어하는 것이 중요해진다.
- Reactive Streams은 Java9의 추가된 비동기 요소 간의 상호 작용을 정의하는 작은 spec이다
- Reactive Streams의 주요 목적은 subscriber가 얼마나 빨리 또는 얼마나 느리게 데이터를 생성하는지 제어할 수 있도록 하는 것
  
## Reactive API

# 출처
> https://docs.spring.io/spring-framework/reference/web/webflux/new-framework.html


