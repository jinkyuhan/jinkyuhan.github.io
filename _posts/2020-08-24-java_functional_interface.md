---
layout: post
title: 자바의 람다 & 함수형 인터페이스
date: 2020-08-24
summary: 자바가 함수를 값처럼 다루는 법
categories: study java
---

- 들어가기전 네줄 요약
  - Java8에서는 함수형 프로그래밍을 지원하기 위해 함수를 값처럼 다룰수 있어야 했다.
  - 이를 위해 함수를 인터페이스로 감싸서 다루는 방법으로 우회했고, 이를 함수형 인터페이스라고 하기로 했다.
  - Java8의 람다식은 사실 함수형 인터페이스의 익명 객체의 생성자 호출을 보기좋게 표현한 것일 뿐이다
  - Java8은 개발자가 함수형 인터페이스를 일일이 만들어서 사용하지 않아도 되도록, 많이 쓰이는 함수형태에 대하여 이미 정의된 Functional Interface 들을 제공한다.

## Functional Interface

---

Java8에서 지원하는 새로운 개념인 Functional Interface는 기존에 없던 언어적 개념이 새로 생겨난 것이 아니다. Functional Interface 는 추상 method 가 하나인
Interface 를 뜻하는 것으로, 함수(메소드)를 값으로써 다루지 못하는 **Java라는 언어 안에서 함수를 하나의 값처럼 다루기 위한 우회 장치이다.**

**즉, Java8은 함수형 프로그래밍을 지원하기 위해 꼭 필요한 특징(함수의 파라미터로 함수 자체를 넘길수 있으며 하나의 값으로 취급됨)을 구현하기 위해서 하나의 추상 메소드만을 갖는 껍데기 객체를 사용하는 방법을
언어적 차원에서 Functional Interface라는 이름으로 표준화 하였다.**

```java
// FIExampleWithoutAnnotation.java
public interface FIExampleWithoutAnnotation {
    public abstract void method()
    // public abstract void method2()
    // 위와 같이 method2 를 추가 할 수 있다. 이 경우 Functional Interface가 아니게 된다.
}
```

위와 같이 추상 메소드를 하나밖에 갖지 않는 인터페이스는 Functional Interface로 기능 할 수 있다. 하지만 이를 컴파일단계에서 명확히하기 위해 아래와 같이 어노테이션을 추가한다.

```java
// FIExample.java
@FunctionalInterface
public interface FIExample {
    public abstract void method1()
    // public abstract void method2()
    // 위와 같이 method2 를 추가 할 경우 명시적으로 Functional Interface 임에도 불구하고
    // 여러개의 추상 메소드를 가지므로 컴파일러가 에러를 표함.
}
```

## Lambda 표현식

---

lambda 표현식은 Functional Interface 의 익명 객체를 생성하는 것과 동치이다. 즉 new 키워드를 통해 익명 객체 생성하여도 전혀 문제가 없지만, 가독성과 편의성을 이유로 Java8부터 추가된
문법이다.

함수형 프로그래밍 패러다임에서 High order function 을 구현할 때는 아래의 파라미터를 함수로 받는다. 하지만 Java 에서 함수(메소드)는 값이 될수 없으므로 위에서 말한 바와 같이 해당 함수만을
포함한 인터페이스(Functional Interface)를 전달 받는다.

아래는 Functional Interface 를 인자로 받는 HighOrderFunction 의 사용예시이다. 람다표현식에 대한 정확한 이해를 위해 아래와 같이 구현하였으며, Java8이상에서 아래와 같은 코드
패턴을 사용할 때는 람다표현식을 우선적으로 사용하는것이 가독성 측면에서 월등히 좋은 것 같다.

```java
// FunctionalInterfaceImpl.java
public class FunctionalInterfaceImpl implements FIExample {
    @Override
    public void method1() {
        System.out.println("method1 called");
    }
}

// HighOrderFunctionExample.java
public class HighOrderFunctionExample {
    public static void runMethod(FIExample methodRunner) {
        // 함수 자체를 인자로 받고싶은데, 자바에서는 불가능 하니 FunctionalInterface 를 인자로 받음
        methodRunner.method1();
    }
}

// Main.java
public class Main {
    public static void main(String[] args) {
        // 방법1. FuntionalInteface 를 구현한 클래스를 넘겨 받음.
        // 단 한번의 고차함수 사용을 위해 클래스를 구현해야해서 불편함.
        HighOrderFunctionExample.runMethod(new FunctionalIntefaceImpl());

        // 방법2. FunctionalInterface를 구현한 익명 클래스로 대신 사용
        HighOrderFunctionExample.runMethod(new FIExample() {
            @Override
            method1() {
                System.out.println("method1 called");
            }
        })

        // 방법3. Lambda 표현식 사용 (권장)
        HighOrderFunctionExample.runMethod(() -> {
            System.out.println("method1 called");
        })
    }
}
```

위 코드에서 방법 2와 방법 3은 정확히 같은 뜻이며, 표현방법만 다를 뿐이다. **요컨데, 람다표현식은 단지 Functional Interface를 구현하는 익명객체의 생성자 호출을 보기 편하게 만들어 놓은 것
뿐이다.**

## Java8이 제공하는 Functional Interface

---

Java8의 함수형 프로그래밍을 사용하다보면 자연스럽게 다양한 입출력을 갖는 함수형 인터페이스가 필요로 하고, 이를 매번 개발자가 구현하기는 힘든 일이다. 그래서 Java8은 일반적인 여러 타입의 입출력에 대하여
Generic을 이용하여 미리 만들어둔 Functional Interface들을 제공한다.

### Java8이 제공하는 기본 FunctionalInterface 들

| Interface name   | abstract method name | parameter type | return type |
| ---------------- | -------------------- | -------------- | ----------- |
| Runnable         | run()                | void           | void        |
| Supplier\<T\>    | get()                | void           | T           |
| Consumer\<T\>    | accept(T)            | T              | void        |
| Function\<T, R\> | apply(T)             | T              | R           |
| Predicate\<T\>   | test(T)              | T              | boolean     |
