
11.4.1 리소스 얻기(GET)

* Mono로 응답받기
~~~
Mono<Ingredient> ingredient = WebClient.create()
    .get()
    .uri("http://localhost:8080/ingredients/{id}", ingredientId)
    .retrieve() //해당 요청을 실행한다.
    .bodyToMono(Ingredient.class) //응답몸체의 페이로드를 Mono<Ingredient>로 추출한다.
    
ingredient.subscribe(i -> {...}) //Mono에 추가로 오퍼레이션을 적용하려면 해당 요청이 전송되기 전에 구독을 해야한다.
~~~

* Flux로 응답받기
~~~
Flux<Ingredient> ingredients = WebClient.create()
    .get()
    .uri("http://localhost:8080/ingredients")
    .retrieve()
    .bodyToFlux(Ingredient.class);
    
ingredients.subscribe(i -> {...})
~~~

* 기본 URI로 요청하기기
기본 URI는 서로 다른 많은 요청에서 사용할 수 있다. 이 경우 기본 URI를 갖는 WebClilent 빈을 생성하고 어디서든 필요한 곳에서 주입하는 것이 유용할 수 있다.
~~~
@Bean
public WebClient webClient() {
  return WebClient.create("http://localhost:8080");
}
~~~

아래에서처럼 주입해서 사용하면 된다.  uri() 인자로 전달되는 URI에는 기본 URI에 대한 상대경로만 지정하면 된다.
~~~
@Autowired
WebClient webClient;

public Mono<Ingredient> getIngredientById(String ingredientId) {
  Mono<Ingredient> ingredient = webClient
      .get()
      .uri("/ingredients/{id}", ingredientId)
      .retrieve()
      .bodyToMono(Ingredient.class);
      
  ingredient.subscribe(i -> {...});
}
~~~

* 오래 실행되는 요청 타임아웃시키기
~~~
Flux<Ingredient> ingredients = WebClient.create()
    .get()
    .uri("http:localhost:8080/ingredients")
    .retrieve()
    .bodyToFlux(Ingredient.class);
    
ingredients
  .timeout(Duration.ofSeconds(1))
  .subscribe(
      i -> {...},
      e -> {
        //handle timeout error
        })
~~~

11.4.2 리소스 전송하기
* post 요청하기
~~~
Mono<Ingredient> ingredientMono = ...;
Mono<Ingredient> result = webClient
  .post()
  .uri("/ingredients")
  .body(ingredientMono, Ingredient.class)
  .retrieve()
  .bodyToMono(Ingredient.class)
  
result.subscribe(i -> {...});
~~~

* Mono, Flux 없이 syncBody() 사용하여 전송가능하다.
~~~
Ingredient ingredient = ...;
Mono<Ingredient> result = webClient
  .post()
  .uri("/ingredients")
  .syncBody(ingredient)
  .retrieve()
  .bodyToMono(Ingredient.class)
  
result.subscribe(i -> {...});
~~~

* POST 요청 대신 PUT 요청으로도 가능하다. 
~~~
Mono<Void> result = webClient
  .put()
  .uri("/ingredients/{id}", ingredient.getId())
  .syncBody(ingredient)
  .retrieve()
  .bodyToMono(Void.class)
  .subscribe();
~~~

11.4.3 리소스 삭제하기
PUT 요청처럼 DELETE 요청도 응답 페이로드를 갖지 않는다. 요청을 전송하려면 bodyToMono 에서 Mono void를 반환하고 subscribe()로 구독해야한다.

~~~
Mono<Void> result = webClient
  .delete()
  .uri("/ingredients/{id}", ingredientId)
  .retrieve()
  .bodyToMono(Void.class)
  .subscribe();
~~~

11.4.4 에러 처리하기

~~~
Mono<Ingredient> ingredientMono = webClient
  .get()
  .uri("http://localhost:8080/ingredients/{id}", ingredientId)
  .retrieve()
  .onStatus(HttpStatus::is4xxClientError, //처리를 원하는 상태코드를 지정한다.
            response -> Mono.just(new UnknownIngredientException())) //원하는 상태코드일 경우 처리 방식을 지정한다.
  // 처리하고 싶은 케이스가 더 있다면 onStatus 더 써서 지정하면 된다.
  .bodyToMono(Ingredient.class)

ingredientMono.subscribe(
  ingredient -> {
    // 식자재 데이터를 처리한다.
  },
  error -> {
    //에러를 처리한다.
  }
)
~~~

11.4.5 요청 교환하기
retrieve 메소드는 ResponseSpec 타입의 객체를 반환하였다. 허나 ResponseSpec은 간단한 상황에서는 쓰기 좋으나 몇가지 경우에서는 제한이 된다. 예를 들어, 응답의 헤더나 쿠키 값을
사용할 필요가 있을 때는 사용할 수가 없다.
ResponseSpec이 기대에 미치지 못할때는 retrieve 메소드 대신 exchange를 호출할 수 있다. exchange 메소드는 ClientResponse 타입의 Mono를 반환한다. ClientResponse 타입은 리액티브
오퍼레이션을 적용할 수 있고 응답의 모든 부분(페이롣, 헤더, 쿠키 등)에서 데이터를 사용할 수 있다.

아래는 exchange와 retrieve 비교 코드이다.

~~~
Mono<Ingredient> ingredientMono = webClient
        .get()
        .uri("http://localhost:8080/ingredients/{id}", ingredientId)
        .exchange()
        .flatMap(cr -> cr.bodyToMono(Ingredient.class));
~~~

~~~
Mono<Ingredient> ingredientMono = webClient
        .get()
        .uri("http://localhost:8080/ingredients/{id}", ingredientId)
        .retrieve()
        .bodyToMono(Ingredient.class);
~~~

exchange에 대해 더 알아볼 수 있는 예제이다. 요청의 응답에 X_UNAVAILABLE의 헤더가 포함될 수 있으며 헤더가 존재한다면 Mono는 빈것이어 한다고 가정을 해보고 코드를 구현한다면 아래와 같다.
~~~
Mono<Ingredient> ingredientMono = webClient
        .get()
        .uri("http://localhost:8080/ingredients/{id}", ingredientId)
        .exchange()
        .flatMap(cr -> {
                if(cr.headers().header("X_UNAVAILABLE").contains("true") {
                    return Mono.empty();
                }
                return Mono.just(cr);
            })
        .flatMap(cr -> cr.bodyToMono(Ingredient.class));
~~~

11.5 리액티브 웹 API 보안
- 기존 spring security는 서블릿 필터를 중심으로 만들어졌다.
- 그러나 WebFlux에서는 서블릿이 개입된다는 보장이 없다. 실제로 non-servilet 서버에 구축될 가능성이 많다.
- 스프링 5.0 버전부터 스프링 시큐리티는 서블릿 기반의 스프링 MVC와 리액티브 스프링 WebFlux 애플리케이션 모두의 보안에 사용될 수 있다. 스프링의 WebFliter가 이 일을 해준다.

11.5.1 리액티브 웹 보안 구성하기
기존 스프링 MVC 어플리케이션 보안 구성과 WebFlux 어플리케이션 보안 구성을 비교해보자
~~~
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/design", "/orders").hasAuthority("USER")
                .antMatchers("/**").permitAll();
    }

}
~~~
~~~
@Configuration
@EnableWebFluxSecurity //@EnableWebSecurity가 아니다.
public class SecurityConfig  //기존과 달리 상속을 받지 않는다. 
{
    
    @Bean
    public SecurityWebFilterChain securityWebFilterChain(ServerHttpSecurity http) { //HttpSecurity 대신 ServerHttpSecurity 사용한다.
            return http
                .authorizeExchange() //요청 수준의 보안을 선언하는 authorizeRequests와 동일한 역할을 한다.
                .pathMatchers("/design", "/orders").hasAuthority("USER")
                .anyExchange() // "/**" 와 동일한 역할한다.
                .permitAll()
                .and()
                .build();
    }
}
~~~

11.5.2 리액티브 사용자 명세 서비스 구성하기
~~~
@Service
public ReactvieUserDetailService userDetailService(UserRepository userRepo) {
    return new ReactiveUserDetailService() {
        @Override
        public Mono<UserDetails> findByUsername(String username) {
            return userRepo.findByUsername(username)
                            .map(user -> {
                                   return user.toUserDetail();
                            });
        }
    }
}
~~~

