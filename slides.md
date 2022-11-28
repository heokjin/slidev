---
theme: apple-basic
class: 'text-center'
layout: intro
background: https://source.unsplash.com/collection/94734566/1920x1080
---
# Spring + Coroutine으로<br>서비스 개발하기
<div class="absolute bottom-10">
  <span class="font-700">
    Scott.hj
  </span>
</div>

---
layout: center
class: text-center
---

<p style="font-size: 28pt">
GW-Basic / Q-Basic <br><br>
COBOL / FORTRAN / PASCAL / C<br><br>
... <br><br>
Java / Android <br><br>
Python / Go <br><br>
Scala / Ruby / Kotlin
</p>


---

# Spring MVC
```text
- Synchronous And Blocking
- Tomcat의 기본 쓰레드 개수는 200
- Common Stack size is 1MB
- One Thread per request
- Thread 갯수가 많아 질수록 Tomcat의 thread 관리비용 증가, Application이 사용할수 있는 메모리 감소
- 이를 해결하기 위해 scale-out
```
<img src="/images/mvc1.png" class="absolute h-80 ">

---

# Spring WebFlux
- 리소스를 더 효울적으로 사용하자
```text
- Asynchronous And Non-blocking
- Event And CallBack Queue
```
<img src="/images/reactive.png" class="h-90">

---

# Kotlin-Coroutine
<a href="https://ko.wikipedia.org/wiki/%EC%BD%94%EB%A3%A8%ED%8B%B4" target="_blank" alt="GitHub"
class="text-xl icon-btn opacity-50 !border-none !hover:text-white">
코루틴 WIKI
</a>
- Coroutine are suspendable
- They can switch their context
- 개발과 유지보수가 쉬워진다

<div grid="~ cols-2 gap-3" m="t 3">
   <img src="/images/coroutine1.png" class="h-70">
   <img src="/images/suspend1.png" class="h-60">
</div>

---

# Finite State Machine (FSM) & CPS
<div grid="~ cols-2 gap-2" m="-t-2">

```kotlin
suspend fun postItem(item: Item) {
   val token = requestToken()
   val post = createPost(token, item)
   processPost(post)
}
```

```kotlin
fun postItem(item: Item, cont: Continuation) {
   val sm = cont as? ThisSM ?: object : ThisSM {
      fun resume(...) {
         postItem(null, this)
      }
   }
   when (sm.label) {
      0 -> {
         sm.item = item
         sm.label = 1
         requestToken(sm)
      }
      1 -> {
         val item = sm.item
         val token = sm.result as Token
         sm.label = 2
         createPost(token, item, sm)
      }
      2 -> { ... }
   }
}
```
</div>

---

# SuspendFunction Test
<div grid="~ cols-2 gap-1" m="t 0">
<div>

```kotlin
suspend fun threadTest(request: ServerRequest): ServerResponse {
    log.info("Main_" + request.exchange().request.id + Thread.currentThread())
    adService.getData() //100ms
    fun1(request)
    log.info("Main_" + request.exchange().request.id + Thread.currentThread())
    adService.getData() //100ms
    fun2(request)
    return ServerResponse.ok().bodyValueAndAwait("threadTest")
}
suspend fun fun1(request: ServerRequest): String {
    log.info("Fun1_Str" + request.exchange().request.id + Thread.currentThread())
        ...
    log.info("Fun1_End" + request.exchange().request.id + Thread.currentThread())
    return "ok"
}
suspend fun fun2(request: ServerRequest): String {
    log.info("Fun2_Str" + request.exchange().request.id + Thread.currentThread())
        ...
    log.info("Fun2_End" + request.exchange().request.id + Thread.currentThread())
    return "ok"
}
```

</div>
<div style="font-size:10px">

<span style="color:orange">[   XNIO-1 I/O-4]</span> : START  <br>
[4 @coroutine#57] : 4921c35b-57_Main1_Thread<span style="color:orange">[XNIO-1 I/O-4 @coroutine#57,5,main]</span> <br>
[3 @coroutine#57] : Fun1_Str_4921c35b-57_Thread<span style="color:orange">[reactor-http-nio-13 @coroutine#57,5,main]</span> <br>
[3 @coroutine#57] : Fun1_End_4921c35b-57_Thread<span style="color:orange">[reactor-http-nio-13 @coroutine#57,5,main]</span> <br>
[3 @coroutine#57] : 4921c35b-57_Main2_Thread<span style="color:orange">[reactor-http-nio-13 @coroutine#57,5,main]</span> <br>
[6 @coroutine#57] : Fun2_Str_4921c35b-57_Thread<span style="color:orange">[reactor-http-nio-36 @coroutine#57,5,main]</span> <br>
[6 @coroutine#57] : Fun2_End_4921c35b-57_Thread<span style="color:orange">[reactor-http-nio-36 @coroutine#57,5,main]</span> <br>
<span style="color:orange">[   XNIO-1 I/O-4]</span> : {"thread":"4921c35b-57","datetime":1668931801937,"method... <br>
</div>
</div>

---

# 특이점
<div grid="~ cols-2 gap-1" m="t 0">
<div>

```kotlin {1,4,5,9,14-19}
package org.springframework.web.reactive.function.server

@Bean
fun router() = coRouter {
    GET("/test2", adHandler::test2)
    GET("/test3", adHandler::rateLimitTest)
}
private fun asHandlerFunction(init: suspend (ServerRequest) -> ServerResponse) = HandlerFunction {
    mono(Dispatchers.Unconfined) {
        init(it)
    }
}

suspend fun delayTest() {
    delay(10)
    for (i in 1..20000) {
        map[i] = sin(cos(sin(cos(sin(cos(i.toDouble()))))))
    } ...
}
```
</div>

<div>
<img src="/images/dispatcher_unconfined_delay.jpg" class="h-80">
</div>

</div>

---

# Dispatchers
Thread 혹은 Thread pool을 결정짓는다.

### Dispatchers.Unconfined
```text
해당 코루틴을 호출한 쓰레드에서 쭉... 실행
코틀린 공식 사이트에선 일반적인 경우에 사용하지 말라고 권고
https://kotlinlang.org/docs/coroutine-context-and-dispatchers.html#debugging-coroutines-and-threads
```

### Dispatchers.IO
```text
It defaults to the limit of 64 threads
cpu core에 비례하여 높아질수 있으며, 갯수를 설정할 수 있다.
Network or Disk
```

### Dispatchers.Default
```text
By default, the maximal level of parallelism used by this dispatcher is equal to the number of CPU cores
Cpu를 많이 사용하는 작업 
```


withContext(Dispatchers.IO / Default)
---

# Dispatchers.Unconfined VS Dispatchers.IO
<div grid="~ cols-2 gap-1" m="t 0">
<div>
<img src="/images/dispatcher_unconfined.png" class="h-100">
</div>

<div>
<img src="/images/dispatcher_io.png" class="h-100">
</div>

</div>

---

# 왜 Kotlin-Coroutine 인가? - sequential code
- I don't care reactor function
- Avoid Callback Hell

Java - webflux
```java
public Mono<AuthenticatedUser> authenticate(String accessToken) {
        return redisTemplate.opsForValue().get(createUserRedisKey(accessToken))
        .map(getAuthenticatedUser())
        .switchIfEmpty(setAuthenticatedUser(accessToken))
        .flatMap(setTUserId());
        }
```

Kotlin - coroutine
```kotlin
suspend fun authenticate(accessToken: String): AuthenticatedUser {
   val redisKey = createUserRedisKey(accessToken)
   val authUser = redisClient.findByKeyWithT(redisKey, AuthenticatedUser::class.java) ?: setAuthenticatedUser(accessToken)
   return setTUserId(authUser)
}
```

---

# 왜 Kotlin-Coroutine 인가? - error handling
Java - webflux
```java {6-8}
 private Mono<Message> translateMessage(Channel1on1SubtypeCall channel, ...) {
        return messageRepository.save(message)
                .flatMap(savedMessage -> {
                    ...
                    ); })
                //onErrorMap
                //onErrorReturn
                .onErrorResume(throwable -> {
                    return Mono.just(MessageTranslationResult.builder().build());
                })
                .flatMap(translateResult -> {
```
Kotlin - coroutine
```kotlin {1,5,8}
try {
   return tAccountWebClient.mutate()
       .baseUrl(host)
      ...
} catch (e: Exception) {
   log.error("TAccountClient provider: $provider, uid: $uid")
   throw e
}
```

---

# Extensions
- Class or Interface에 새로운 함수를 쉽게 추가
- 같은 method라면 원래 클래스의 멤버 함수가 우선
- private, protected 멤버를 가져올수 없다.
```kotlin {1-10,16,17}
package org.springframework.web.reactive.function.server.ServerRequest
 
companion object {
    fun ServerRequest.tUser(): TUser {
        val u = attribute(T_USER).orElseThrow { GatewayException(HttpStatus.NOT_FOUND, 404, "Not found tUser") } as TUser
        when (u.state) {
            UserState.SLEEPING -> throw UnicornException.Dormant
            UserState.BANNED -> throw UnicornException.UnicornBanned(u.banned_reason ?: "")
            else -> {
                ...
            }
        }
    }
}

suspend fun saveHyundaiYachtInApp(request: ServerRequest): ServerResponse {
    val user = request.tUser()
    val form = request.formData().awaitSingle().toSingleValueMap()
}
```

---

# Resilience4j
```kotlin {1-10,13,15,19,20,22,23}
companion object {
   val circuitBreaker = circuitBreaker("commonCircuit")
   val rateLimiter = rateLimiter("commonRateLimiter")
   val retryer = retryer("commonRetryer")
}
private fun customRateLimitConfig(): RateLimiterConfig = RateLimiterConfig.custom()
   .limitForPeriod(1000)
   .limitRefreshPeriod(Duration.ofSeconds(1))
   .timeoutDuration(Duration.ofSeconds(5))
   .build()
----------------------------------------------------------------------------------------
suspend fun rateLimitTest(request: ServerRequest): ServerResponse {
     rateLimiter.executeSuspendFunction {
         showNotice()
     }
 }

 suspend fun complexResilienceTest(request: ServerRequest): ServerResponse {
     retry.executeSuspendFunction {
         rateLimiter.executeSuspendFunction {
             showNotice()
         }
     }
 }
```

---

# Swagger
Spring MVC
```java
@Operation(summary = "특정 상품코드정보로 약관 정보 목록", description = "유효한 모든 약관 정보를 필수/선택별로 목록을 내려준다.")
@GetMapping("/agreements/code/{productCode}")
public List<AgreementDto> getAgreementByCode(
    @Parameter(description = "상품 코드") @PathVariable String productCode,
    @Parameter(description = "locale 정보") @RequestParam(value = "locale", defaultValue = "en") Locale locale,
    @Parameter(description = "국가코드 정보") @RequestParam(value = "country", defaultValue = "kr") String countryCode) {
```

Kotlin - couroutine
```java
@RouterOperations(
    RouterOperation(path = "/promotion/out-app/event", method = [RequestMethod.POST], beanClass = AdService::class, beanMethod = "hyundaiYacht"),
    RouterOperation(path = "/promotion/fun/event1", method = [RequestMethod.GET], beanClass = AdService::class, beanMethod = "funtion1"),
    RouterOperation(path = "/promotion/fun/event2", method = [RequestMethod.GET], beanClass = AdService::class, beanMethod = "funtion2"),
    RouterOperation(path = "/promotion/fun/event3", method = [RequestMethod.GET], beanClass = AdService::class, beanMethod = "funtion3")
)
```

---

# Async

```kotlin
 suspend fun threadTest(request: ServerRequest): ServerResponse {
    val locations = listOf("Seoul", "Jeju", "Pusan", "Chuncheon", "Gwangju", "YangYang", "NamYang") //7번 호출 
    val res = CoroutineScope(Dispatchers.Unconfined).async {
        var plus = ""
        val datas = mutableListOf<Deferred<Any>>()
        for (data in locations) {
            datas += async { locationService.getData(request, data) } //300ms
        }

        for (data in datas) {
            plus += data.await()
        }

        plus
    }
    return ServerResponse.ok().bodyValueAndAwait(res.await())
}

```

---

# Async

<div grid="~ cols-2 gap-1" m="t 0">
<div>
<span style="color:darkseagreen">Dispatchers.Unconfined</span>
<div style="font-size:10px">

[4 @coroutine#10]: 7a1be981-2_Main1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#10,5,main]</span> <br>
[4 @coroutine#12]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#12,5,main]</span>_Seoul <br>
[4 @coroutine#13]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#13,5,main]</span>_Jeju<br>
[4 @coroutine#14]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#14,5,main]</span>_Pusan<br>
[4 @coroutine#15]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#15,5,main]</span>_Chuncheon<br>
[4 @coroutine#16]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#16,5,main]</span>_Gwangju<br>
[4 @coroutine#17]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#17,5,main]</span>_YangYang<br>
[4 @coroutine#18]: 7a1be981-2_Sub1<span style="color:orange">[XNIO-1 I/O-4 @coroutine#18,5,main]</span>_NamYang<br>
[   XNIO-1 I/O-4]: {"thread":"7a1be981-2","datet ...

</div>
</div>


<div>
<span style="color:darkseagreen">Dispatchers.IO</span>
<div style="font-size:10px">

[2 @coroutine#10]: -2_Main1<span style="color:orange">[XNIO-1 I/O-2 @coroutine#10,5,main]</span><br>
[5 @coroutine#12]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-5 @coroutine#12,5,main]</span>_Seoul<br>
[0 @coroutine#13]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-10 @coroutine#13,5,main]</span>_Jeju<br>
[6 @coroutine#14]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-6 @coroutine#14,5,main]</span>_Pusan<br>
[4 @coroutine#15]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-14 @coroutine#15,5,main]</span>_Chuncheon<br>
[7 @coroutine#18]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-7 @coroutine#18,5,main]</span>_NamYang<br>
[4 @coroutine#17]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-4 @coroutine#17,5,main]</span>_YangYang<br>
[8 @coroutine#16]: -2_Sub1<span style="color:orange">[DefaultDispatcher-worker-8 @coroutine#16,5,main]</span>_Gwangju<br>
[   XNIO-1 I/O-2]: {"thread":"-2","datetime":1669203681615,"method":"GET","path":"/test6" ...
</div>
</div>

</div>

---

# Async

<div grid="~ cols-2 gap-1" m="t 0">
<div>
Users: 200 <br>
<span style="color:red">Dispatchers.Unconfined</span>
<img src="/images/async_unfined.png" class="h-90">

```text
상대적으로 적은 thread를 활용함
```
</div>

<div>
Users: 200 <br>
<span style="color:red">Dispatchers.IO</span>
<img src="/images/async_io.png" class="h-90">

```text
상대적으로 많은 thread를 활용함
```
</div>
</div>

---

# 하지만 runBlocking이 나타난다면?

<div grid="~ cols-2 gap-1" m="t 0">
<div>

```kotlin
suspend fun threadTest(request: ServerRequest): ServerResponse {
    val locations = listOf("Seoul", "Jeju", "Pusan", "Chu...")
    var plus = ""
    runBlocking {  //Dispatchers.IO or Unconfined
        val datas = mutableListOf<Deferred<Any>>()
        for(data in locations) {
            datas += async {adService.getData(request, data)}
        }
    
        for(data in datas) {
            plus += data.await()
        }
    }
    return ServerResponse.ok().bodyValueAndAwait(plus)
}
```

```text
runBlocking은 코루틴이 모두 끝날때 까지 기다려주는
편리한 기능이지만 Thread를 block 한다.
```

</div>

<div>
<img src="/images/runblocking.png" class="h-90">
</div>
</div>

---

# Performance Test
```text
Pod: 1
Cpu: 3000m
Memory: 1024Mi

WebClient maxConnection: 1000

JVM_MEMORY: -Xms512m -Xmx512m
WARMUP OK...
```
<hr>
```kotlin
suspend fun test(request: ServerRequest): ServerResponse {
    val map = HashMap<Int, Double>()
    for (i in 1..100) {
        map[i] = sin(cos(sin(cos(sin(cos(i.toDouble()))))))
    }
    val data = adService.getData()
    val r = "$data, ${map.size}"
    return ServerResponse.ok().bodyValueAndAwait(r)
}
```



---

# Test(Low cpu) - MVC
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Users - 100<br>
Thread-200, Loop-100, <span style="color:red">Connection-100ms</span>
<img src="/images/test-mvc_100ms_100loop_200thread_100user.png" class="h-80">
</div>

<div>
Users - 1000<br>
Thread-200, Loop-100, <span style="color:red">Connection-100ms</span>
<img src="/images/test_mvc_100ms_100loop_200thread_1000user.png" class="h-80">
</div>

</div>

---

# Test(Low cpu) - MVC
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Thread-8102, Loop-100, <span style="color:red">Connection-100ms</span>
<img src="/images/test_mvc_8102thread_1000users.png" class="h-80">
</div>

<div>
<span style="color:red">OOM</span>
<img src="/images/oom.png" class="h-10"> <br>
```text
Too much Thread
```
</div>
</div>

---

# Test(Low cpu) - Coroutine {Undertow}
<div grid="~ cols-2 gap-1" m="t 0">

<div>
Users - 1000<br>
Thread-200, Loop-100, <span style="color:red">Connection-100ms</span>
<img src="/images/test_gw_100ms_100loop_200thread.png" class="h-100">
</div>

<div>
Users - 1000<br>
Thread-6, Loop-100, <span style="color:red">Connection-100ms</span>
<img src="/images/test_gw_100ms_100loop_6thread.png" class="h-100">
</div>

</div>

---

# Test(High cpu) MVC vs Coroutine - 800users
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Thread-200, <span style="color:red">Loop-10000</span>, Connection-5ms
<img src="/images/test_mvc_5ms_10000loop_800users.png" class="h-80">
<img src="/images/test_mvc_5ms_10000loop_800users_err.png" class="h-30">
</div>

<div>
Thread-4, <span style="color:red">Loop-10000</span>, Connection-5ms, <span style="color:red">TOMCAT</span>
<img src="/images/test_gw_5ms_10000loop_800users.png" class="h-80">
<img src="/images/test_gw_5ms_10000loop_800users_err.png" class="h-30">
</div>
</div>

---

# Test(High cpu) Coroutine - 800users {Undertow}
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Thread-1, <span style="color:red">Loop-10000</span>, Connection-5ms
<img src="/images/test_gw_5ms_10000loop_800users_1thread.png" class="h-80">
<img src="/images/test_gw_5ms_10000loop_800users_1thread_err.png" class="h-30">
</div>

<div>
Thread-16, <span style="color:red">Loop-10000</span>, Connection-5ms
<img src="/images/test_gw_5ms_10000loop_800users_16thread.png" class="h-80">
<img src="/images/test_gw_5ms_10000loop_800users_16thread_err.png" class="h-30">
</div>
</div>

---

# Test(High cpu) Coroutine - 1000users {Undertow}
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Thread-6, <span style="color:red">Loop-10000</span>, Connection-5ms
<img src="/images/test_gw_5ms_10000loop_1000users_6thread.png" class="h-80">
<img src="/images/test_gw_5ms_10000loop_1000users_6thread_err.png" class="h-30">
</div>

<div>
```text
was에 따라 트래픽이 처리되는 양상이 달랐지만...
io-thread는 cpu core 의 두배정도가 적당해 보인다.
트래픽이 높은 서비스에서는 로깅도 주의하자.
```
</div>
</div>

---

# Test(Normal) MVC vs Coroutine
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Thread-200, Loop-100, Connection-5ms, <span style="color:red">Users-2000</span>
<img src="/images/test_mvc_5ms_100loop_2000users.png" class="h-80">
<img src="/images/test_mvc_5ms_100loop_2000users_err.png" class="h-30">
</div>

<div>
Thread-6, Loop-100, Connection-5ms, <span style="color:red">Users-2000</span>
<img src="/images/test_gw_5ms_100loop_2000users.png" class="h-80">
<img src="/images/test_gw_5ms_100loop_2000users_err.png" class="h-30">
</div>

</div>

---

# Test(Normal) MVC vs Coroutine
<div grid="~ cols-2 gap-1" m="t 0">
<div>
Thread-200, Loop-100, Connection-5ms, <span style="color:red">Users-2500</span>
<img src="/images/test_mvc_5ms_100loop_2500users.png" class="h-80">
<img src="/images/test_mvc_5ms_100loop_2500users_err.png" class="h-30">
</div>

<div>
Thread-6, Loop-100, Connection-5ms, <span style="color:red">Users-5000</span>
<img src="/images/test_gw_5ms_100loop_5000users.png" class="h-80">
<img src="/images/test_gw_5ms_100loop_5000users_err.png" class="h-30">
</div>
</div>

---
layout: center
class: text-center
---
<p style="font-size: 30pt">
Ktor?
</p>

---

# 추가테스트 Ktor
<div grid="~ cols-2 gap-1" m="t 0">
<div>

```kotlin
fun main() {
    embeddedServer(Netty, port = 8080, host = "0.0.0.0", module = Application::module, configure = {
        connectionGroupSize = 3
        workerGroupSize = 20
        callGroupSize = 6
    }).start(wait = true)
}
val commonModule = module {
    single {
        HttpClient(CIO) {
            install(ContentNegotiation) {
                jackson()
            }
            engine {
                endpoint {
                    maxConnectionsCount = 3000
                    maxConnectionsPerRoute = 500
                    connectTimeout = 3000
                }
            }
        }
    }
}
```
</div>

<div>
Thread-6, Loop-100, Connection-5ms, Users-2000
<img src="/images/test_gw2_ktor.png" class="h-100">
</div>
</div>

---
theme: apple-basic
class: 'text-center'
layout: intro
background: https://source.unsplash.com/collection/94734566/1920x1080
---
# END

결론 kotlin coroutine 쓸만하다.

---