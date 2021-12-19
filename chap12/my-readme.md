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
- 도메인 타입을 매핑하는데 유용한 어노테이션들이 있다.
  * @Id : 이것이 지정된 속성을 문서 ID로 지정한다.
  * @Document : 이것이 지정된 도메인 타입을 몽고DB에 저장되는 문서로 선언한다.
  * @Field : 몽고DB의 문서에 속성을 저장하기 위해 필드 이름(과 선택적으로 순서)을 지정한다.
- 이러한 세개의 어노테이션 중에 @Id와 @Docuemnt는 필수다. @Field가 지정되지 않은 도메인 타입의 속성들은 필드 이름과 속성 이름을 같은 것으로 간주한다.
~~~
@Data
@RequiredArgsConstructor
@NoArgsConstructor(access=AccessLevel.PIVATE, force=true)
@Document(collection="ingredients") //여기서 collection는 RDB의 테이블명이랑 같다고 봐야하는듯?
public class Ingredient {
  
  @Id
  private final String id;
  private final String name;
  private final Type type;
  
  public static enum Type {
    WRAP, PROTEIN, VEGGIES, CHEESE, SAUCE
  }
}
~~~

~~~
@Data
@RestResource(rel="tacos", path="tacos")
@Document
public class Taco {
  @Id
  private String id;
  
  @NotNull
  @Size(min=5, message="Name must be at least 5 characters long")
  private String name;
  
  private Date createdAt = new Date();
  
  @Size(min=1, message="You must choose at least 1 characters ingredient")
  private List<Ingredient> ingredients;
}
~~~

~~~
@Data
@Document
public class Order implements Serializable {
  private static final long serialVersionUID = 1L;
  
  @Id
  private String id;
  
  private Date placedAt = new Date();
  
  @Field("customer") //customer 열을 문서에 저장한다는 것을 나타내기 위해서... 라는데 뭔말인지 모르겠다.
  private User user;
  
  //다른 속성들은 생략
  
  private List<Taco> tacos = new ArrayList<>();
  
  public void addDesign(Taco design) {
    this.tacos.add(design);
  }
  
}
~~~

~~~
@Data
@NoArgsConstructor(access=AccessLevel.PRIVATE, force=true)
@RequiredArgsConstructor
@Document
public class User implements UserDetails {
  private static final long serialVersionUID = 1L;
  
  @Id
  private String id;
  
  private final String username;
  
  private final String password;
  private final String fullname;
  //나머지는 생략...
}
~~~

12.3.3 리액티브 몽고DB 리퍼지터리 인터페이스 작성하기
- 몽고DB의 리액티브 리퍼지터리를 작성할 때는 ReactiveCrudRespository 혹은 ReactiveMongoRepository 둘 중에 선택해서 사용할 수 있다.
- 둘의 차이점은 ReactiveCrudRespository가 새로운 문서나 기존 문서의 save() 메소드에 의존하는 반면, ReactiveMongoRepository는 새로운 문서의 저장에 최적화된 소수의 특별한 insert() 메소드를 제공한다.
- Ingredient의 경우 데이터베이스 추가하는 게 많이 없으므로 ReactiveCrudRespository 사용하도록 한다.
~~~
@CrossOrigin(origins="*")
public interface IngredientRepository extends ReactiveCrudRepository<Ingredient, String> {
}
~~~
- Taco 객체의 경우 생성하는 경우가 많으므로 ReactiveMongoRepository의 최적화된 insert() 메소드가 유용할 수 있으므로 ReactiveMongoRepository를 사용해본다.
- 그러나 ReactiveMongoRepository의 단점은 몽고DB에 매우 특화되어 있으므로 다른 데이터베이스에는 사용이 불가하다는 점이다. 만약 중간에 다른 데이터베이스로 전환할 경우에는 곤란할 수 있으므로 신중하게 결정할 필요가 있다.
~~~
public interface TacoRepository extends ReactiveMongoRepository<Taco, String> {
  Flux<Taco> findByOrderByCreatedAcDesc();
}
~~~
- 위 코드에서 findByOrderByCreatedAcDesc()는 Flux를 반환하므로 결과의 페이징을 신경쓰지 않아도 된다. 대신에 take() 오퍼레이션을 적용하여 Flux에서 발행되는 처음 12개의 Taco 객체만 반환할 수 있다. 예를 들어, 최근 생성된 타코들을 보여주는 컨트롤러에서는 다음과 같이 호출할 수 있다.
~~~
Flux<Taco> recents = repo.findByOrderCreatedAtDesc().take(12);
~~~

- 아래는 OrderRepository 인터페이스이다. Order의 경우도 잦은 생성메소드가 호출되므로 ReactiveMongoRepository를 사용해본다.
~~~
public interface OrderRepository extends ReactiveMongoRepository<Order, String> {

}
~~~

- 마지막으로 User 객체를 문서로 저장하는 리퍼지터리 인터페이스이다.
~~~
public interface UserRepository extends ReactvieMongoRepository<User, String> {
  Mono<User> findByUsername(String username);
}
~~~

요약
=============
1. 스프링 데이터는 카산드라, 몽고DB, 카우치베이스, 레디스 데이터베이스의 리액티브 리퍼지터리를 지원한다.
2. 스프링 데이터의 리액티브 리퍼지터리는 리액티브가 아닌 리퍼지터리와 동일한 프로그래밍 모델을 따른다. 단, Flux나 Mono와 같은 리액티브 타입을 사용한다.
3. JPA 리퍼지터리와 같은 리액티브가 아닌 리퍼지터리는 Mono나 Flux를 사용하도록 조정할 수 있다. 그러나 데이터를 가져오거나 저장할 때 여전히 블로킹이 발생한다.
4. 관계형이 아닌 데이터베이스를 사용하려면 해당 데이터베이스에서 데이터를 저장하는 방법에 맞게 데이터를 모델링하는 방법을 알아야 한다.
5. 
