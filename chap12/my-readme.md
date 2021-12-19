12. 리액티브 데이터 퍼시스턴스

12.1.1 스프링 데이터 리액티브 개요
- 리액티브 리파지터리는 도메인 타입이나 컬렉션 대신 Mono나 Flux를 인자로 받거나 반환하는 메소드를 갖는다는 것이 핵심이다.
~~~
Flux<Ingredient> findByType(Ingredient.Type type)
Flux<Taco> saveAll(Publisher<Taco> tacoPublisher);
~~~

12.1.2 리액티브와 리액티브가 아닌 타입 간의 변환
- 리액티브를 지원하지 않는 관계형 데이터베이스로부터 데이터를 가져온 뒤 가능한 빨리 리액티브 타입으로 변환하여 사용할 수 있다.
~~~
List<Order> orders = repo.findByUser(someUser);
Flux<Order> orderFlux = Flux.fromIterable(orders); //fromIterable 사용하여 변환할 수 있다.
~~~
~~~
Order order = repo.findById(Long id);
Mono<Order> orderMono = Mono.just(order);
~~~
- 반대로 리액티브를 지원하지 않는 데이터베이스로 저장하기 위해서는 도메인타입이나 Iterable 타입으로 변환 가능하다.
~~~
Taco taco = tacoMono.block();
tacoRepo.save(taco);
~~~
~~~
Iterable<Taco> tacos = tacoFlux.toIterable();
tacoRepo.saveAll(tacos)
~~~

- 그러나 위의 .block() 혹은 toIterable()은 모두 추출작업을 할 때 블로킹이 되므로 좋은 방법은 아닐것이다. 이처럼 블로킹 되는 추출 오퍼레이션을 피하는 더 리액티브한 방법이 있다.
. 즉, Mono나 Flux를 구독하면서 발행되는 요소 각각에 대해 원하는 오퍼레이션을 수행하는 것이다.
- 여기서 리파지터리의 save() 메소드는 여전히 블로킹 오퍼레이션이다. 그러나 Flux나 Mono가 발행하는 데이터를 소비하고 처리하는 리액티브 방식의 subscribe()를 사용하므로 블로킹 방식의 일괄
처리보다는 더 바람직한다.
~~~
tacoFlux.subscribe(taco -> {
  tacoRepo.save(taco);
});
~~~

(카산드라 관련된 거는 생략함. 필요할 경우 책을 보자.)

12.3.1 스프링 데이터 몽고DB 활성화하기
- 리액티브가 아닌 몽고DB를 사용하고자 할 때는 다음 의존성을 빌드에 추가한다.
~~~
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>
    spring-boot-starter-data-mongodb
  </artifactId>
</dependency>
~~~
- 리액티브 리퍼지터리를 작성할 때는 아래 의존성을 추가해야한다.
~~~
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>
    spring-boot-starter-data-mongodb-reactive
  </artifactId>
</dependency>
~~~
- 몽고DB 내장 DB를 사용하기 위해서는 아래 의존성을 추가하면 개발 시 편리하게 사용할 수 있다.
~~~
<dependency>
  <groupId>de.falpdoodle.dmbed</groupId>
  <artifactId>
    de.flapdoodle.embed.mongo
  </artifactId>
</dependency>
~~~
- 몽고DB 속성 설정의 항목들은 아래와 같다.
~~~
data:
  mongodb:
    host: mongodb.tacocloud.com
    port: 27018
    username: tacocloud
    password: s3cr3tp455w0rd
    database: reactive-mongodb //데이터베이스 이름이며, 기본값은 test이다.
~~~

12.3.2 도메인 타입을 문서로 매핑하기
- 도메인 타입을 매피앟는데 유용한 어노테이션들이 있다.
