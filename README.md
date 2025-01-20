# 쿠폰 발급 시스템(실습으로 배우는 선착순 이벤트 시스템)

## 요구사항 정의
선착순 100명에게 할인쿠폰을 제공하는 이벤트를 진행하고자 한다.

이 이벤트는 아래와 같은 조건을 만족하여야 한다.
- 선착순 100명에게만 지급되어야 한다.
- 101개 이상이 지급되면 안된다.
- 순간적으로 몰리는 트래픽을 버틸 수 있어야합니다.

**쿠폰 발급 기능(싱글 스레드)**
```java
@Test
public void 한번만응모() {
    applyService.apply(1L);
    
    long count = couponRepository.count();
    
    assertThat(count).isEqualTo(1L);
}
```

**쿠폰 발급 기능(멀티 스레드)**
```java
@Test
public void 여러명응모() throws InterruptedException {
    int threadCount = 1000;
    ExecutorService executorService = Executors.newFixedThreadPool(32);
    CountDownLatch latch = new CountDownLatch(threadCount);

    for(int i = 0 ; i < threadCount ; i++){
        long userId = i;
        executorService.submit(() -> {
            try {
                applyService.apply(userId);
            } finally {
                latch.countDown();
            }
        });
    }

    latch.await();

    long count = couponRepository.count();

    assertThat(count).isEqualTo(100L);
}
```

단순히 싱글 스레드로 동작하는 경우에는 문제가 없지만, 멀티 스레드로 동작하는 경우에는 문제가 발생한다.

## 해결방식 1. Redis를 이용한 해결
Redis의 incr 명령어를 활용하여 문제를 해결할 수 있다. 하지만 쿠폰 생성에 대한 요청이 무수히 많은 경우
전체적으로 성능이 안 좋아질 수 있다(특히 RDB). 

```java
@Repository
public class CouponCountRepository {
    private final RedisTemplate<String, String> redisTemplate;

    public CouponCountRepository(RedisTemplate<String, String> redisTemplate) {
        this.redisTemplate = redisTemplate;
    }

    public Long increment() {
        return redisTemplate
                .opsForValue()
                .increment("coupon_count");
    }
}

```

## 해결방식 2. Kafka를 이용한 해결

_참고 문서 및 링크_
- [실습으로 배우는 선착순 이벤트 시스템(최상용)](https://www.inflearn.com/course/%EC%84%A0%EC%B0%A9%EC%88%9C-%EC%9D%B4%EB%B2%A4%ED%8A%B8-%EC%8B%9C%EC%8A%A4%ED%85%9C-%EC%8B%A4%EC%8A%B5/dashboard)
