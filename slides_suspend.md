---
title: Coroutine-CPS
type: slide
slideOptions:
center: false
---

# Coroutine-CPS

---

# CPS
(Continuation Passing-Style)

---

## Continuation Passing-Style

- Continuation이 무엇이며 Passing을 한다는 어떤 Style일까?

---

#### Direct style

```
fun postItem(item: Item){
    val token = requestToken()
    val post = createPost(token, item)
    processPost(post)
}
```

- 요청 후에 리턴 값을 기대하며 대기를 한다.
- requestToken() 이후의 createPost(), processPost()는 continuation이다.
- Direct style에서는 순차적으로 작성하게 된다.

---

#### Continuation-Passing Style

```
fun postItem(item: Item) {
    requestToken { token ->
        val post = createPost(token, item)
        processPost(post)
    }
}






fun postItem(item: Item) {
    requestToken { token ->
        createPost(token, item) { post -> 
            processPost(post)
        }
    }
}
```

- Continuation-Passing에서는 return 값을 파라미터로 전달하여 다음 프로세스들을 호출한다.
- CPS == Callback

---

### Coroutines Direct Style

```
suspend fun postItem(item: Item) {
    val token = requestToken()
    val post = createPost(token, item)
    processPost(post)
}
```

- suspend를 사용하면 위의 동작이 동기처리된다.
- CPS를 할 필요가 없다.
- 직관적이라 이해하기 쉽다.

---

### 어떻게 동작하는가? - Suspend
```
suspend fun createPost(
    token: Token,
    item: Item): Post {...}
```
- suspend 키워드를 달면, 컴파일 시에 다음과 같이 바뀐다.
```
Object createPost(
    Token token,
    Item item,
    Continuation<Post> const){...}
```
- Continuation이란 파라미터가 하나 더 생겼다.
- Continuation은 callback이다.

---

#### Continuation 생김새

```
interface Continuation<in T> {
    val context: CoroutineContext
    fun resume(value: T)
    fun resumeWithException(exception: Throwable)
}
```

---

#### Continuations

```
suspend fun postItem(item: Item) {
    val token = requestToken()
    val post = createPost(token, item)
    processPost(post)
}
```
- 메서드 안의 호출되는 모든 것들은 initial continuation이라고 할 수 있다.
- requestToken() 이후 호출되는 createPost()와 processPost()는 Continuation이다.

---

#### 어떻게 동작하는가? - labeling

```
suspend fun postItem(item: Item) {
    switch(label){
        case 0:
            val token = requestToken()
        case 1:
            val post = createPost(token, item)
        case 2: 
            processPost(post)
    }
    
}
```

---

#### 어떻게 동작하는가? - Continuation

```
fun postItem(item: Item, cont: Continuation) {
    // State Machine
    val sm = object : CoroutineImpl { ... }
    
    switch(label){
        case 0:
            val token = requestToken(sm)
        case 1:
            val post = createPost(token, item, sm)
        case 2: 
            processPost(post)
    }
    
}
```

---

#### 어떻게 동작하는가? - save state

```
fun postItem(item: Item, cont: Continuation) {
    val sm = object : CoroutineImpl { ... }
    
    switch(label){
        case 0:
            sm.item = item // save data
            sm.lable = 1 // next label
            val token = requestToken(sm) // 끝나면 resume() 호출
        case 1:
            val post = createPost(token, item, sm)
        case 2: 
            processPost(post)
    }
}
```

---

#### 어떻게 동작하는가? - callback

```
fun postItem(item: Item, cont: Continuation) {
    // 자기 자신인지 확인
    val sm = cont as? ThisSM ?: object : ThisSM {
        fun resume(...){
            postItem(null, this) // suspend function 호출
        }
    }
    
    switch(label){
        case 0:
            sm.item = item
            sm.lable = 1
            // 끝나면 sm에서 resume() 호출됨
            val token = requestToken(sm)
        case 1:
            val item = sm.item
            val token = sm.result as Token
            sm.label = 2
            val post = createPost(token, item, sm)
        case 2: 
            processPost(post)
    }
}
```

---

#### 어떻게 동작하는가? - restore state

```
fun postItem(item: Item, cont: Continuation) {
    val sm = ...
    
    switch(label){
        case 0:
            sm.item = item
            sm.lable = 1
            val token = requestToken(sm)
        case 1:
            val item = sm.item
            val token = sm.result as Token
            sm.label = 2
            val post = createPost(token, item, sm)
        ...
    }
}
```

---

## The end

- 참고: [KotlinConf 2017 - Deep Dive into Coroutines on JVM by Roman Elizarov
  ](https://www.youtube.com/watch?v=YrrUCSi72E8)[(pdf)](https://resources.jetbrains.com/storage/products/kotlinconf2017/slides/2017+KotlinConf+-+Deep+dive+into+Coroutines+on+JVM.pdf)