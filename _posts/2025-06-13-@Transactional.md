---
title: @Transactional 
categories:
  - Spring
feature_image: "https://picsum.photos/2560/600?image=872"
---

## @Transactional
@Transactional 은 Spring Framework에서 트랜잭션을 관리하는 어노테이션이다.
```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default -1;

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}

```

## 동작 방식
트랜잭션 어노테이션을 사용하면 프록시 객체 생성한다. 단계별로 파훼해보자.

### 기본적인 어노테이션의 동작 방식
1. 컴파일하면 어노테이션 정책에 의해 런타임까지 유지 된다.
    - @Retention(RetentionPolicy.RUNTIME) 이므로 런타임시까지 유지된다.
    - 스프링은 리플랙션을 통해 해당 어노테이션이 붙은 클래스/메서드를 탐색한다.
2. 빈 등록 단계에서 동장
    - 컴포넌트 스캔에 의해 Bean으로 ApplicationContext에 등록된다.
3. 프록시를 통한 기능 확장
    - 프록시 객체를 사옹하려 원래 객체 대신 등록
    - 이 프록시는 호출을 가로채, 트랜잭션을 관리

### 그렇다면 트랜잭션은?
1. 프록시 객체를 생성한다.
  - @EnableTransactionManagement , TransactionInterceptor AOP 어드바이스가 등록된다.
  - 대상 클래스가 인터페이스가 있다면 JDK dynamic proxy , 없다면 CGLIB 프록시를 사용한다.
    - JdkDynamicAopProxy
    ```java
    //JdkDynamicAopProxy 안에서 Proxy 객체를 얻는 부분
    public Object getProxy() {
        return this.getProxy(ClassUtils.getDefaultClassLoader());
    }
    ```
    - CGLIB
    
2. 메서드 호출 시 트랜잭션 시작
- 실제 비지니스 메서드 호출 전, TransactionInterceptor 실행된다.
```java
    //transaction이 적용된 메서드와 타겟 클래스를 가져온다.
    public Object invoke(final MethodInvocation invocation) throws Throwable {
        Class<?> targetClass = invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null;
        return this.invokeWithinTransaction(invocation.getMethod(), targetClass, new TransactionAspectSupport.InvocationCallback() {
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();
            }
        });
    }
```

- 내부 PlatformTransactionManager를 사용해 트랜잭션이 시작된다.
```java
TransactionStatus status = txManager.getTransaction(txDefinition);
```
3. 메서드 실행
- 트랜잭션이 시작된 상태에서 실제 비지니스 로직 실행한다.

4. 정상 종료시 트랜잭션 커밋
- 메서드 실행 후 예외가 없다면 txManager를 통해 commit 실행한다.
```java
    //TransactionAspectSupport 안 transactionManager에 의한 commit
    protected void commitTransactionAfterReturning(TransactionInfo txInfo) {
        if (txInfo != null && txInfo.hasTransaction()) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() + "]");
            }

            txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
        }

    }
```
5. 예외 발생 시 롤백
- RuntimeException 발생시, 스프링 트랜잭션은 txManger가 rollback을 한다.
```java
//내부 invokeWithinTransaction을 보면 rollbackOn을 통해 롤백하는걸 볼 수 있다.
protected Object invokeWithinTransaction(Method method, Class targetClass, final InvocationCallback invocation) throws Throwable {
        final TransactionAttribute txAttr = this.getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
        final PlatformTransactionManager tm = this.determineTransactionManager(txAttr);
        final String joinpointIdentification = this.methodIdentification(method, targetClass);
        if (txAttr != null && tm instanceof CallbackPreferringPlatformTransactionManager) {
            try {
                Object result = ((CallbackPreferringPlatformTransactionManager)tm).execute(txAttr, new TransactionCallback<Object>() {
                    public Object doInTransaction(TransactionStatus status) {
                        TransactionInfo txInfo = TransactionAspectSupport.this.prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);

                        ThrowableHolder var4;
                        try {
                            try {
                                Object var3 = invocation.proceedWithInvocation();
                                return var3;
                            } catch (Throwable ex) {
                                if (txAttr.rollbackOn(ex)) {
                                    if (ex instanceof RuntimeException) {
                                        throw (RuntimeException)ex;
                                    }

                                    throw new ThrowableHolderException(ex);
                                }
                            }
                            .....
                            //DefaultTransactionAttribute 기본 전략은 RumtimeException 혹은 Error를 상속하는 Throwable
                            public boolean rollbackOn(Throwable ex) {
                                return ex instanceof RuntimeException || ex instanceof Error;
                            }
```

## Transaction Option
- propagation (전파수준)
  - REQUIRED (default) : 현재 트랜잭션이 존재하면 참여, 가장 일반적
  - REQUIRES_NEW : 현재 트랜잭션이 있어도 새롭게 시작
  - MANDATORY : 현재 트랜잭션이 존재하면 중첩 트랜잭션 시작
  - SUPPORTS : 현재 트랜잭션이 있으면 참여, 없으면 트랜잭션 없이 시작
  - NOT_SUPPORTED : 현재 트랜잭션이 있으면 중단 후 트랜잭션 없이 시작
  - NEVER : 현재 트랜잭션이 없다면 예외 발생
- isolation (고립수준)
  - default 값은 데이터베이스의 기본 고립 수준을 사용한다. (MySQL은 Repeatable Read, Oracle은 Read Committed)
  - READ_UNCOMMITTED
  - READ_COMMITTED
  - REPEATABLE_READ
  - SERIALIZABLE
- timeout
  - 트랜잭션이 지정된 시간안에 완료되지 않는다면 rollback 처리
  - 기본값은 무제한이다.
- readOnly
  - 읽기 전용 트랜잭션임을 명시, 주로 조회 쿼리 성능 최적화 용도로 사용한다.
  - readOnly로 설정되면 데이터베이스 slave DB를 조회 
- rollbackFor
  - 이 옵션을 사용한다면 지정된 checked 예외 발생하면 rollback
- noRollbackFor
  - 지정한 예외가 발생해도 rollback 하지 않는 것을 의미한다.