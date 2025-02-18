---
id: 81kqcmlpjfys4fgbvrkss4b
title: Tips
layout: page
desc: ''
tags: spring-security
updated: 1739852742785
created: 1739816230926
---

## Authorization

Everyone wants to create their own API, and that makes sense. We want our app to look like our domain.

Historically, Spring Security has one way to do this: SpEL.
It's done by publishing a custom `MethodSecurityExpressionHandler`, which handler returns a `SecurityExpressionRoot` instance that has the application's custom expressions.
This allows the application to do:

```java
@PreAuthorize("isSubscribed() || isAdmin()")
```

While I appreciate the readability of this, I'd prefer something more like the following:

```java
@Authorize({ Subscribed.class, Admin.class })
```

This is more testable because I can have unit tests for `Subscribed.class` and for `Admin.class`.

So, how do we achieve that?

First, create the annotation:

```java
@PreAuthorize("permitAll()")
public @interface Authorize {
    Class<? extends AuthorizationManager<MethodInvocation>> any() default {};
}
```

Second, create an interceptor:

```java
@Component
public final class AuthorizeAuthorizationManager implements AuthorizationManager<MethodInvocation> {
    private final SecurityAnnotationScanner authorize = SecurityAnnotationScanners.fromAnnotation(Authorize.class);
    private final Map<Class<?>, AuthorizationManager<MethodInvocation>> cache = new HashMap<>();

    @Override
    public AuthorizationDecision check(Supplier<Authentication> authentication, MethodInvocation invocation) {
        Authorize authorize = this.authorize.scan(invocation, invocation.getDeclaringClass());
        if (authorize == null) {
            return null;
        }
        return AuthorizationManagers.anyOf(Stream.of(authorize.any()).map(this::manager).toList());
    }

    private AuthorizationManager<MethodInvocation> manager(Class<?> clazz) {
        return this.cache.computeIfAbsent(clazz, this::newInstance);
    }

    private <T> T newInstance(Class<?> clazz) {
        try {
            return this.cache.computeIfAbsentclazz.newInstance();
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }
    }
}
```

And then finally, publish an interceptor:

```java
@Bean(ROLE_INFRASTRUCTURE)
static MethodInterceptor authorizeInterceptor(AuthorizeAuthorizationManager authorize) {
    var interceptor = AuthorizationManagerBeforeMethodInterceptor.preAuthorize(authorize);
    interceptor.setOrder(AuthorizeInterceptorOrder.PRE_AUTHORIZE - 1);
    return interceptor;
}
```

This allows you to unit-test the authorization classes and get compile-time feedback on authorization expressions.