---  
share: true  
toc: true  
categories: [with Dan]  
tags: [redis, db, lock, transaction, bean, spring, kotlin]  
layout: post  
title: "Redis의 Cache 와 Lock"  
date: "2023-06-29"  
github_title: "2023-06-29-Redis의 Cache 와 Lock"  
image:  
  path: /assets/img/with_dan.png  
---  
  
## TL; DR  
  
여울과 댄의 Chain of Thoughts를 정리합니다. 매주 목요일 오전 8시 ~ 10시까지 기술, 경제, 사회 등의 분야를 막론한 이야기들을 주고 받습니다. 몇 가지 간단한 질문에서 시작해서 꼬리에 꼬리를 물고 시간이 찰 때까지 파헤친 기록들을 정리합니다. Y는 Yeoul, D는 Dan을 의미합니다. 그리고 괄호() 안에 있는 내용은 Y의 Chain of Thoughts를 위한 주관적인 생각이나 질문입니다.  
  
## Redis를 사용한 성능 개선  
   
Q. (Y)  
Redis를 메모리 캐시 정도로 알고 있다. 쓰면 성능에 좋다고는 하는데, Production에서 실제로 어떻게 쓰는지, 그리고 어떤 식으로 성능이 좋아지는지를 조금 구체적인 그림으로 알고 싶다.  
  
A. (D)  
- 사실상 콴다의 모든 백엔드 서비스가 이걸 사용한다  
- 사용하면 무조건 좋다.  
	- 다만 비용과 복잡도 측면에서 당연히 비용이 발생하지만, 이것을 사용했을 때 성능을 개선하는게 너무나 자명해서 "왜안씀?"의 정도로 생각한다  
	- 메시지 브로커로도 사용할 수 있다. 다만, Redis로 메시지 브로커를 사용할 바에는 다른 서비스 (Kafka, RabbitMQ)를 쓰는게 더 나은 경우가 많아서 메시지 브로커로는 잘 사용하지 않는다.  
	- 일반적으로는 캐시로 사용한다. Redis는 단순하게 이야기하면 엄청 빠른 key-value store라서 실제 DB를 조회하기 전에 특정 key로 검색해서 이미 캐시된 값이 있는지 확인한다.  
  
Q. (Y)  
예시를 좀 보고 싶다. 일부분만 좀 보여달라.  
  
A. (D)  
특정 서버와 인터페이스 일부분을 보여주겠다. (사내 코드이므로 전부 공개할 수는 없고 general 한 interface 일부분만 공개하기로 함.)  
  
- 여기 보이는 것처럼 엄청 단순하다. cache라는 인스턴스는 메서드가 정의된 서비스 (QuizService)가 초기화될때 Spring 으로부터 Bean으로 주입받는다.  
- 이건 Backend Common Library에서 관리하고 있어서 여기저기서 편리하게 가져다 쓰면 된다.   
  
```kotlin  
suspend fun listCurrentQuizzes(    
	locale: ServiceLocale,    
	userId: Long,    
	grade: Int?,    
	categories: List<QuizCategory>,    
): ListCurrentQuizzesResponse {    
	val gradeCategory = getQuizGradeCategory(locale, category = null, grade)    
	    
	val cacheKey = CacheKey.CURRENT_QUIZZES_WITH_CHOICES.build(locale, gradeCategory)    
	    
	val currentQuizzes: List<Quiz> = cache.get(cacheKey) {    
	quizRepository.findWithChoicesCurrent(locale)    
	.filter(quizGradeCategoryPredicate(gradeCategory)).associateBy { it.category }.values.toList()    
	} ?: emptyList()    
	    
...  
}  
```  
  
- 자세히 보면 Cache Key를 Build하는 부분이 있고, 이 Key를 가지고 가져오는 cache 인스턴스의 Interface 부분이 있는데 이 부분도 엄청 단순하다 Cache Key를 Build하는 부분은 그냥 kotlin enum으로 관리하고 있다.   
- cache interface는 내가 직접 만든거긴 한데 우리는 Spring MVC가 아니라 Spring Webflux를 사용하고 있어서 non-blocking으로 redis cache에 접근할 수 있도록 해야 한다. 그래서 이 부분이 조금 다르다.  
  
  
  
### Chain of Thoughts  
  
Q. (Y)  
- Redis의 사용사례에서 갑자기 Message Queue로 빠지는 것 같은데 갑자기 채팅 같은 서비스들이 어떻게 구동되는지 궁금해졌다. 이쪽 이야기를 좀 해보자  
  
A. (D)  
- 예전 콴다 chat server가 Redis 기반의 메시지 브로커를 사용하고 있었다.   
- 왜 메시지 브로커를 사용하냐면 한 사용자가 메시지를 보냈을 떄, 이 이벤트를 사용자한테 바로 보내는게 아니라 서버를 거쳐서 메세지와 메타데이터를 저장하는 과정을 거쳐서 사용자한테 메시지를 보내야 하기 때문이다. (그래야 재전송 같은 기능들 구현이 가능한 듯 하다).   
- 카카오톡 같은 대용량 메신저에서도 메시지 브로커를 사용하는지는 잘 모르겠다. 하지만 되게 자주 쓰는 패턴인 것 같다. (아마 그래서 최근에 Dan이 Kafka나 RabbitMQ같은 메세지 큐를 몇 번 언급했었던 것 같다.)  
- 즉, 어떤 유저가 어떤 메시지를 어떤 채팅방에 쐈다 라는 이벤트를 갑지할 방법이 필요하다  
  
Q. (Y)  
- 아 생각해보니까, A가 B에게 메세지를 보내는게 Peer to Peer가 아니구나. 서버를 거쳐야 하고, 그래서 웹소켓으로 연결하더라도 A와 서버, 서버와 B 이렇게 해야될 것 같긴 하다. (그리고 그게 효율적인지는 잘 모르겠다.)  
- 이런 의미에서 Message Queue를 사용하는 방식은 되게 괜찮은 방법이라는 생각이 든다.  
  
  
### Additional Discussions  
  
- Redis에서 Cache할 때, 해당 key-value에 대해 유효 기간을 설정한다. 이 유효기간이 지나면 이 값을 주기적으로 감지하고 있다가 날리는가? 아니면 그냥 값은 그대로 놔두고, 해당 key로 read 요청이 들어왔을 때 그제서야 이 값은 유효기간이 지났으니 cache fail을 주는가?  
- Kotlin enum에 정리 (build를 사용해 본적이 없었음.)  
- Kotlin inline function  
- Kafka, RabbitMQ 예제를 좀 만들어보자  
  
  
## Redis Lock  
  
Q. (Y)  
- 앞서 Redis를 자주 사용하는 패턴 중에 캐시를 언급했고, 그 다음으로 Lock을 언급했다. 사실 Lock이라는거는 되게 DB에서는 기본적인 개념이라서 Lock이 무엇인가에 대해서라기보다는 주로 어떤 경우에 Redis Lock을 거는지가 궁금했다.  
- 왜냐하면 앞에서 Dan이 말해준 Redis는 key-value store이고, RDB같이 복잡한 테이블간 조인이나 트랜잭션 관리가 필요 없을 것 같아서 그 사용처도 굉장히 단순할거 같아서.  
  
A. (D)  
- 대표적인 경우가 클라이언트 "따닥"이슈이다.   
- 보통 DB락을 쓸때는 순차적으로 동시성 이슈 없이 모든 요청이 잘 반영되기 위해서 쓰는거고, Redis Lock에서는 클라이언트에서 미처 처리하지 못했거나, 구현상의 이슈로 한번 버튼을 누르더라도 요청이 두번 오는 경우들이 있는데, 이때 요청을 한번만 처리하기 위해서 사용한다  
  
Q. (Y)  
- 아 그러면 프론트엔드에서 Throttle과 같은 기능을 서버에서 Redis Lock으로 구현하는건가?  
- Redis에서 value의 uniqueness는 key에 의해 결정되고, 여기서 lock을 잡는다는건 key에 대해 잡는다는거 같은데 그러면 lock을 잡은 key에 대해 read 요청이 오면 그냥 무시하는건가?  
  
A. (D)  
- 비슷한 느낌이다  
- 코드를 보면 이해가 쉬울거 같다.  
  
```kotlin  
LockKey.LOCK_USER_QUIZ_HISTORY.build(userId, quiz.id).let {    
	if (!lock.acquireLock(it)) return    
}  
```  
  
- 함수가 호출되어서 lock을 얻으려고 하는데, 이미 lock을 잡고 있는게 있으면 기다리고 자시고 하지 않고 그냥 return 한다. (무시한다는 뜻)  
  
  
### Chain of Thoughts  
  
Q. (Y)  
-  아까 보여준 코드에서 (SNUTT Backend 인듯) `SELECT FOR UPDATE` 라는 문법을 본 거 같은데 사실 다는 DB 쿼리를 짤 일이 잘 없어서 저 문법이 생소하다. 조금 자세한 설명이 가능한가?  
  
A. (D)  
- SQL 표준 문법이다. Exclusive Lock을 잡는 문법이고 `SELECT FOR UPDATE`와 `SELECT FOR SHARE`가 있다. 학교에서 Lock을 배워서 알겠지만 Lock에는 Exclusive Lock과 Shared Lock이 있다.  
- 근데 여기서 재밌는 부분은 MYSQL innoDB의 구현인데 `SELECT FOR UPDATE`로 Exclusive Lock을 잡아도 "잠금 없는 읽기"가 가능하다. 이 부분을 잘 모르는 개발자들이 꽤 많을 거다.  
  
Q. (Y)  
- 내가 이해한건 `SELECT FOR UPDATE`로 Lock을 잡더라도 innoDB에서는 `SELECT`로 lock 없이 읽으려는 요청에 대해서는 잠금 없는 읽기를 허용한다는 뜻으로 이해했다.  
- 왜 이런 구현이 있는지는 잘 모르겠지만, 뭔가 브라우저도 구현체마다 동작 방식이 조금씩 다른 경우가 있는데 (이를테면 requestAnimationFrame callback을 처리하는 task queue와 promise를 처리하는 task queue의 처리 우선순위는 브라우저마다 조금씩 다르다) 뭔가 그런거 같다는 느낌이 든다  
- 이걸 이해하고 있지 못한 주니어 개발자가 실수를 할 수도 있겠다는 생각이 들긴 한다.  
  
A. (D)  
- 여기에 더해서 재밌는 개념이 비관적 Lock과 낙관적 Lock 이라는 게 있다. 실제 JPA에서 Annotation도 지원할만큼 De Facto로 알려진 개념이라서 한번 봐도 좋을것 같다. (`@OptimisticLock`)  
  
