
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

11.4.4 에처 처리하기

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


  
