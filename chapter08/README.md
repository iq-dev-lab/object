# Chapter 08. 의존성 관리하기

> *"협력을 위해서는 의존성이 필요하지만, 과도한 의존성은 애플리케이션을 수정하기 어렵게 만든다."*

## 📌 핵심 개념

이 장에서는 **충분히 협력적이면서도 유연한 객체**를 만들기 위해 의존성을 관리하는 방법을 학습합니다.

### 🎯 학습 목표

- 의존성의 본질과 변경에 미치는 영향 이해하기
- 런타임 의존성과 컴파일타임 의존성의 차이 파악하기
- 결합도를 낮추는 구체적인 기법 익히기
- 유연한 설계를 위한 의존성 관리 원칙 적용하기

---

## 📖 목차

1. [의존성 이해하기](#1-의존성-이해하기)
2. [유연한 설계](#2-유연한-설계)
3. [컨텍스트 확장하기](#3-컨텍스트-확장하기)
4. [실전 예제](#4-실전-예제)
5. [패턴별 실전 적용 가이드](#5-패턴별-실전-적용-가이드)
6. [핵심 정리](#6-핵심-정리)


---

## 1. 의존성 이해하기

> 📂 **코드**: [`Movie.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/Movie.java) | [`DiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/DiscountPolicy.java)

### 1.1 변경과 의존성

#### 🔍 의존성의 정의

```java
public class PeriodCondition implements DiscountCondition {
    private DayOfWeek dayOfWeek;        // DayOfWeek에 의존
    private LocalTime startTime;         // LocalTime에 의존
    private LocalTime endTime;           // LocalTime에 의존

    public boolean isSatisfiedBy(Screening screening) {  // Screening에 의존
        return screening.getStartTime().getDayOfWeek().equals(dayOfWeek) &&
                startTime.compareTo(screening.getStartTime().toLocalTime()) <= 0 &&
                endTime.compareTo(screening.getStartTime().toLocalTime()) >= 0;
    }
}
```

**의존성의 두 가지 의미:**

| 시점 | 의미 | 설명 |
|------|------|------|
| **실행 시점** | 존재 필요성 | 의존하는 객체가 정상적으로 동작하려면 실행 시에 의존 대상이 **반드시 존재**해야 함 |
| **구현 시점** | 변경 전파 | 의존 대상이 변경되면 의존하는 객체도 **함께 변경**될 수 있음 |

#### 💡 핵심 통찰

```
의존성은 방향성을 가지며 항상 단방향이다.

PeriodCondition → Screening (O)
Screening → PeriodCondition (X)

Screening이 변경되면 PeriodCondition이 영향을 받지만,
PeriodCondition이 변경되어도 Screening은 영향을 받지 않는다.
```

### 1.2 의존성 전이 (Transitive Dependency)

#### 📊 의존성 전이의 메커니즘

```
PeriodCondition → Screening → Movie → Money

A가 B에 의존하고, B가 C에 의존하면
A는 C에 간접적으로 의존하게 된다.
```

**의존성의 종류:**

| 종류 | 설명 | 코드 가시성 | 예시 |
|------|------|------------|------|
| **직접 의존성** (Direct Dependency) | 한 요소가 다른 요소에 직접 의존 | 명시적으로 드러남 | `PeriodCondition → Screening` |
| **간접 의존성** (Indirect Dependency) | 의존성 전이에 의해 영향 전파 | 암묵적으로 숨겨짐 | `PeriodCondition → Movie` |

#### ⚠️ 중요한 점

```
의존성 전이는 "가능성"을 의미할 뿐이다.

실제 전이 여부는:
1. 변경의 방향
2. 캡슐화의 정도
에 따라 달라진다.
```

### 1.3 런타임 의존성 vs 컴파일타임 의존성

#### 🎭 두 의존성의 차이

```java
// 컴파일타임: Movie는 DiscountPolicy 인터페이스에만 의존
public class Movie {
    private DiscountPolicy discountPolicy;  // 추상화에 의존
    
    public Movie(String title, Duration runningTime, Money fee, 
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

```java
// 런타임: 실제로는 구체적인 정책 인스턴스와 협력
Movie avatar = new Movie("아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new AmountDiscountPolicy(...)  // 구체 클래스 인스턴스
);

Movie titanic = new Movie("타이타닉",
    Duration.ofMinutes(180),
    Money.wons(11000),
    new PercentDiscountPolicy(...)  // 다른 구체 클래스 인스턴스
);
```

#### 📊 관계 다이어그램

```
[컴파일타임 의존성]
Movie ---------> DiscountPolicy (인터페이스)
                      ↑
                      |
        +-------------+-------------+
        |                           |
AmountDiscountPolicy    PercentDiscountPolicy


[런타임 의존성]
avatar(Movie) --------> AmountDiscountPolicy 인스턴스

titanic(Movie) -------> PercentDiscountPolicy 인스턴스
```

#### 💎 유연한 설계의 핵심

```
컴파일타임 구조와 런타임 구조 사이의 거리가 멀면 멀수록
설계는 더 유연해지고 재사용 가능해진다.

동일한 소스코드 구조로 다양한 실행 구조를 만들 수 있어야 한다.
```

### 1.4 컨텍스트 독립성

#### 🎯 개념 이해

**컨텍스트 독립성 (Context Independence):**
클래스가 사용될 특정한 문맥에 대해 **최소한의 가정만으로** 이뤄져 있다면 다른 문맥에서 재사용하기 수월해진다.

#### 비교: 컨텍스트 의존적 vs 독립적

```java
// ❌ 컨텍스트 의존적 - 구체적인 할인 정책에 강하게 결합
public class Movie {
    private AmountDiscountPolicy discountPolicy;  // 특정 정책에 의존
    
    public Movie(String title, Duration runningTime, Money fee) {
        this.discountPolicy = new AmountDiscountPolicy(...);  // 직접 생성
    }
    
    // 금액 할인 정책 문맥에서만 사용 가능
    // 다른 할인 정책 문맥에서는 재사용 불가능
}
```

```java
// ✅ 컨텍스트 독립적 - 추상화에 의존
public class Movie {
    private DiscountPolicy discountPolicy;  // 추상화에 의존
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {  // 외부에서 주입
        this.discountPolicy = discountPolicy;
    }
    
    // 어떤 할인 정책 문맥에서도 재사용 가능
    // 금액 할인, 비율 할인, 중복 할인 등 모든 정책과 협력 가능
}
```

#### 🎨 컨텍스트 독립성의 장점

```
1. 다양한 문맥에서 재사용 가능
   - 금액 할인 정책 문맥
   - 비율 할인 정책 문맥
   - 할인 없는 문맥
   - 중복 할인 문맥

2. 응집력 있는 객체 구성
   - 책임이 명확함
   - 변경이 용이함

3. 유연한 설계
   - 객체 구성 방법을 재설정 가능
   - 변경 가능한 시스템으로 나아갈 수 있음
```

### 1.5 의존성 해결하기

#### 🔧 의존성 해결의 정의

```
컴파일타임 의존성을 실행 컨텍스트에 맞는
적절한 런타임 의존성으로 교체하는 것
```

#### 세 가지 해결 방법

**1️⃣ 생성자를 통한 의존성 해결**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}

// 사용
Movie avatar = new Movie("아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new AmountDiscountPolicy(...)  // 생성 시점에 의존성 해결
);
```

**장점:**
- 객체 생성 시점에 의존성이 명확히 설정됨
- 객체의 상태가 항상 완전함 (불완전한 상태 방지)
- 가장 권장되는 방식

**2️⃣ Setter 메서드를 통한 의존성 해결**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}

// 사용
Movie avatar = new Movie("아바타", Duration.ofMinutes(120), Money.wons(10000));
avatar.setDiscountPolicy(new AmountDiscountPolicy(...));  // 생성 후 의존성 해결

// 런타임에 의존성 변경 가능
avatar.setDiscountPolicy(new PercentDiscountPolicy(...));
```

**장점:**
- 실행 시점에 의존 대상 변경 가능
- 유연성 향상

**단점:**
- 객체 생성 후 setter 호출 전까지 상태 불완전
- NullPointerException 위험

**3️⃣ 메서드 인자를 통한 의존성 해결**

```java
public class Movie {
    public Money calculateMovieFee(Screening screening, 
                                    DiscountPolicy discountPolicy) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}

// 사용
avatar.calculateMovieFee(screening, new AmountDiscountPolicy(...));
```

**적합한 경우:**
- 협력 대상에 대해 지속적인 의존 관계가 필요 없을 때
- 메서드 실행 시마다 의존 대상이 달라져야 할 때
- 일시적인 의존성이 필요할 때

#### 🎯 최선의 조합

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 생성자로 기본 의존성 설정 (상태 완전성 보장)
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // Setter로 유연성 제공 (런타임 변경 가능)
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}
```

**이점:**
- 객체 생성 시 완전한 상태 보장 (생성자)
- 필요시 런타임에 유연한 변경 가능 (setter)


---

## 2. 유연한 설계

> 📂 **코드**: [`AmountDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/AmountDiscountPolicy.java) | [`PercentDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/PercentDiscountPolicy.java)

### 2.1 의존성과 결합도

#### 🤔 의존성이 모두 나쁜가?

```
NO! 의존성은 객체들의 협력을 가능하게 만드는 매개체이다.

문제는 의존성의 "존재"가 아니라 의존성의 "정도"이다.
```

#### 📊 의존성 vs 결합도

| 개념 | 의미 | 표현 방식 | 예시 |
|------|------|----------|------|
| **의존성** | 관계의 유무 | 존재한다 / 존재하지 않는다 | "A는 B에 의존한다" |
| **결합도** | 관계의 정도 | 강하다 / 느슨하다 | "A와 B는 강하게 결합되어 있다" |

#### 🎯 바람직한 의존성

```java
// ❌ 바람직하지 못한 의존성 - 구체 클래스에 의존
public class Movie {
    private PercentDiscountPolicy discountPolicy;  // 강한 결합도
    
    // PercentDiscountPolicy 문맥에서만 재사용 가능
    // AmountDiscountPolicy가 필요하면 Movie를 수정해야 함
}
```

```java
// ✅ 바람직한 의존성 - 추상화에 의존
public class Movie {
    private DiscountPolicy discountPolicy;  // 느슨한 결합도
    
    // 모든 DiscountPolicy 문맥에서 재사용 가능
    // 새로운 정책 추가 시 Movie는 변경 불필요
}
```

#### 💡 결합도 판단 기준

```
바람직한 의존성 (느슨한 결합도):
✅ 다양한 환경에서 재사용 가능
✅ 컨텍스트에 독립적
✅ 변경의 영향이 제한적

바람직하지 못한 의존성 (강한 결합도):
❌ 특정 문맥에서만 재사용 가능
❌ 컨텍스트에 강하게 결합
❌ 변경의 영향이 광범위
```

### 2.2 지식이 결합을 낳는다

#### 🧠 결합도의 본질

```
결합도는 한 요소가 다른 요소에 대해 알고 있는 정보의 양으로 결정된다.

더 많이 안다 = 더 강하게 결합된다 = 더 적은 컨텍스트에서 재사용 가능
더 적게 안다 = 더 느슨하게 결합된다 = 더 많은 컨텍스트에서 재사용 가능
```

#### 📊 지식의 양과 결합도 비교

**강한 결합도 - 구체 클래스에 의존**

```java
public class Movie {
    private AmountDiscountPolicy discountPolicy;
    
    // Movie가 알아야 하는 것들:
    // 1. 할인 정책이 AmountDiscountPolicy라는 사실
    // 2. AmountDiscountPolicy의 생성 방법
    // 3. AmountDiscountPolicy의 구체적인 메서드
    // 4. AmountDiscountPolicy가 속한 상속 계층
    // 5. AmountDiscountPolicy의 내부 구현 방식
    
    // → 너무 많은 지식 = 강한 결합도
}
```

**느슨한 결합도 - 추상화에 의존**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // Movie가 알아야 하는 것들:
    // 1. 할인 정책이 존재한다는 사실
    // 2. calculateDiscountAmount 메시지를 이해한다는 사실
    
    // → 최소한의 지식 = 느슨한 결합도
}
```

### 2.3 추상화에 의존하라

#### 🎨 추상화의 힘

**추상화 (Abstraction):**
특정 절차나 물체를 의도적으로 생략하거나 감춤으로써 복잡도를 극복하는 방법

```
추상화는 대상에 대해 알아야 하는 지식의 양을 줄여
결합도를 느슨하게 유지할 수 있게 한다.
```

#### 📊 추상화 레벨과 결합도

```
높은 결합도 (많은 지식 필요)
    ↓
구체 클래스 의존성 (Concrete Class Dependency)
    - 구체적인 구현까지 모두 알아야 함
    - 가장 많은 지식 필요
    
    ↓ (추상화 수준 상승)
    
추상 클래스 의존성 (Abstract Class Dependency)
    - 메서드 내부 구현은 감춤
    - 자식 클래스 종류는 감춤
    - 하지만 상속 계층은 알아야 함
    
    ↓ (추상화 수준 상승)
    
인터페이스 의존성 (Interface Dependency)
    - 오직 퍼블릭 인터페이스만 알면 됨
    - 상속 계층도 알 필요 없음
    - 가장 적은 지식 필요
    ↓
낮은 결합도 (적은 지식 필요)
```

#### 💻 코드로 보는 추상화 레벨

**Level 1: 구체 클래스 의존 (높은 결합도)**

```java
public class Movie {
    private AmountDiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // 구체적인 구현까지 알아야 함
        this.discountPolicy = new AmountDiscountPolicy(
            Money.wons(800),
            new SequenceCondition(1),
            new SequenceCondition(10)
        );
    }
    
    // AmountDiscountPolicy에 강하게 결합
    // PercentDiscountPolicy로 변경하려면 Movie를 수정해야 함
}
```

**Level 2: 추상 클래스 의존 (중간 결합도)**

```java
public abstract class DiscountPolicy {
    protected abstract Money getDiscountAmount(Screening screening);
    
    // 템플릿 메서드는 공개되지만
    // 구체적인 할인 금액 계산 방식은 감춰짐
}

public class Movie {
    private DiscountPolicy discountPolicy;  // 추상 클래스 의존
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // DiscountPolicy의 자식이라면 모두 협력 가능
    // 하지만 DiscountPolicy 계층이라는 것은 알아야 함
}
```

**Level 3: 인터페이스 의존 (낮은 결합도)**

```java
public interface DiscountPolicy {
    Money calculateDiscountAmount(Screening screening);
    // 순수한 인터페이스만 정의
}

public class Movie {
    private DiscountPolicy discountPolicy;  // 인터페이스 의존
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // DiscountPolicy 인터페이스를 구현한 어떤 객체와도 협력 가능
    // 상속 계층, 구현 방식 등 내부 사항을 전혀 알 필요 없음
}
```

#### 💎 핵심 원칙

```
의존하는 대상이 더 추상적일수록 결합도는 더 낮아진다.

구체 클래스 < 추상 클래스 < 인터페이스
(높은 결합도)              (낮은 결합도)
```

### 2.4 명시적인 의존성

#### 🎯 명시적 vs 숨겨진 의존성

**숨겨진 의존성 (Hidden Dependency) ❌**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // 내부에서 구체 클래스 직접 생성
        this.discountPolicy = new AmountDiscountPolicy(...);
    }
    
    // 문제점:
    // 1. 퍼블릭 인터페이스에 의존성이 드러나지 않음
    // 2. Movie가 AmountDiscountPolicy에 의존한다는 것을 알려면 내부 구현을 봐야 함
    // 3. 다른 정책으로 변경하려면 Movie 클래스를 수정해야 함
}

// 사용 시
Movie avatar = new Movie("아바타", Duration.ofMinutes(120), Money.wons(10000));
// 어떤 할인 정책을 사용하는지 알 수 없음!
```

**명시적인 의존성 (Explicit Dependency) ✅**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 생성자 시그니처에 의존성 명시
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 장점:
    // 1. 퍼블릭 인터페이스를 통해 의존성이 명확히 드러남
    // 2. 코드를 읽는 것만으로 의존 관계를 파악 가능
    // 3. Movie 수정 없이 실행 컨텍스트에서 적절한 정책 선택 가능
}

// 사용 시
Movie avatar = new Movie("아바타", 
    Duration.ofMinutes(120), 
    Money.wons(10000),
    new AmountDiscountPolicy(...)  // 어떤 정책을 사용하는지 명확!
);
```

#### 📊 비교 분석

| 측면 | 숨겨진 의존성 | 명시적인 의존성 |
|------|-------------|----------------|
| **가시성** | 내부 구현을 봐야 알 수 있음 | 퍼블릭 인터페이스에 드러남 |
| **유연성** | 변경하려면 클래스 수정 필요 | 실행 시점에 자유롭게 교체 |
| **재사용성** | 특정 컨텍스트에 고정됨 | 다양한 컨텍스트에서 재사용 |
| **테스트** | 특정 구현에 고정되어 테스트 어려움 | Mock 객체 주입으로 쉽게 테스트 |

#### 💡 명시적 의존성의 다양한 형태

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 1. 생성자를 통한 명시적 의존성
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 2. Setter를 통한 명시적 의존성
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 3. 메서드 인자를 통한 명시적 의존성
    public Money calculateMovieFee(Screening screening, 
                                    DiscountPolicy discountPolicy) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

#### 🎯 핵심 원칙

```
"클래스가 다른 클래스에 의존하는 것은 부끄러운 일이 아니다."

의존성은 협력을 가능하게 하므로 바람직하다.
경계해야 할 것은 의존성 자체가 아니라 의존성을 감추는 것이다.

숨겨진 의존성을 밝은 곳으로 드러내면
설계가 유연하고 재사용 가능해진다.
```

### 2.5 new는 해롭다

#### ⚠️ new의 문제점

**1️⃣ 구체 클래스에 대한 직접적인 의존**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // new는 구체 클래스 이름을 직접 명시해야 함
        this.discountPolicy = new AmountDiscountPolicy(...);
        //                        ↑
        //                   구체 클래스에 의존
        //                   추상화가 아님!
    }
}
```

**문제:** Movie가 추상화(DiscountPolicy)가 아닌 구체 클래스(AmountDiscountPolicy)에 의존하게 되어 결합도가 높아진다.

**2️⃣ 생성자 인자에 대한 지식 필요**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        this.discountPolicy = new AmountDiscountPolicy(
            Money.wons(800),                    // 할인 금액
            new SequenceCondition(1),            // 첫 번째 조건
            new SequenceCondition(10),           // 두 번째 조건
            new PeriodCondition(                 // 세 번째 조건
                DayOfWeek.MONDAY,
                LocalTime.of(10, 0),
                LocalTime.of(11, 59)
            ),
            new PeriodCondition(                 // 네 번째 조건
                DayOfWeek.THURSDAY,
                LocalTime.of(10, 0),
                LocalTime.of(20, 59)
            )
        );
    }
    
    // Movie가 알아야 하는 것들:
    // - AmountDiscountPolicy의 생성자 시그니처
    // - 필요한 인자의 타입과 순서
    // - SequenceCondition, PeriodCondition의 생성 방법
    // - DayOfWeek, LocalTime의 사용 방법
    
    // → 너무 많은 지식 = 강한 결합도!
}
```

#### 📊 의존성 폭발

```
Movie가 new를 사용하면:

Movie
 ├─> AmountDiscountPolicy (직접 의존)
 ├─> Money (인자로 필요)
 ├─> SequenceCondition (인자로 필요)
 ├─> PeriodCondition (인자로 필요)
 ├─> DayOfWeek (PeriodCondition 인자)
 └─> LocalTime (PeriodCondition 인자)

하나의 new가 6개의 의존성을 만들어낸다!
```

#### ✅ 해결 방법: 사용과 생성의 책임 분리

```java
// ❌ Before: 사용 + 생성 책임을 모두 가짐
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // 생성 책임
        this.discountPolicy = new AmountDiscountPolicy(...);
    }
    
    public Money calculateMovieFee(Screening screening) {
        // 사용 책임
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

```java
// ✅ After: 사용 책임만 가짐
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 생성 책임은 클라이언트로 이동
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 오직 사용 책임만
    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

```java
// 클라이언트가 생성 책임을 담당
Movie avatar = new Movie(
    "아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new AmountDiscountPolicy(         // 클라이언트가 생성
        Money.wons(800),
        new SequenceCondition(1),
        new SequenceCondition(10),
        new PeriodCondition(DayOfWeek.MONDAY, LocalTime.of(10, 0), LocalTime.of(11, 59)),
        new PeriodCondition(DayOfWeek.THURSDAY, LocalTime.of(10, 0), LocalTime.of(20, 59))
    )
);
```

#### 🎯 개선 효과

```
Before:
Movie → AmountDiscountPolicy (강한 결합)
      → Money
      → SequenceCondition
      → PeriodCondition
      → DayOfWeek
      → LocalTime

After:
Movie → DiscountPolicy (느슨한 결합)

모든 구체적인 의존성은 클라이언트로 이동!
```

#### 💡 설계 개선 정리

```
✅ 사용과 생성의 책임 분리
   - Movie는 사용만
   - 클라이언트는 생성만

✅ 의존성을 생성자에 명시적으로 드러냄
   - public Movie(..., DiscountPolicy discountPolicy)

✅ 구체 클래스가 아닌 추상화에 의존
   - DiscountPolicy (인터페이스/추상 클래스)

✅ 객체 생성 책임을 클라이언트로 이동
   - Movie 내부에서 new 제거
```

### 2.6 가끔은 생성해도 무방하다

#### 🤔 트레이드오프: 결합도 vs 사용성

```
완벽하게 결합도를 낮추면 사용성이 떨어질 수 있다.

생성자에 항상 모든 의존성을 전달하는 것은
클라이언트 입장에서 번거로울 수 있다.
```

#### 💡 해결책: 복수의 생성자 제공

**기본 객체를 제공하는 간편한 생성자 + 유연한 생성자**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 1. 기본 정책을 사용하는 간편한 생성자
    public Movie(String title, Duration runningTime, Money fee) {
        this(title, runningTime, fee, new AmountDiscountPolicy(...));
        // 생성자 체이닝을 통해 기본 정책 설정
    }
    
    // 2. 유연한 생성자 (원하는 정책을 주입 가능)
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.title = title;
        this.runningTime = runningTime;
        this.fee = fee;
        this.discountPolicy = discountPolicy;
    }
}
```

#### 📊 사용 시나리오

```java
// 시나리오 1: 대부분의 경우 - 기본 정책 사용
Movie movie1 = new Movie("아바타", Duration.ofMinutes(120), Money.wons(10000));
// 간편하게 생성! AmountDiscountPolicy가 자동 적용

// 시나리오 2: 특별한 경우 - 다른 정책 필요
Movie movie2 = new Movie(
    "타이타닉",
    Duration.ofMinutes(180),
    Money.wons(11000),
    new PercentDiscountPolicy(...)  // 원하는 정책 명시적 주입
);

// 시나리오 3: 할인 없는 영화
Movie movie3 = new Movie(
    "인터스텔라",
    Duration.ofMinutes(169),
    Money.wons(12000),
    new NoneDiscountPolicy()  // 할인 없음
);
```

#### 🎯 메서드 오버로딩에도 적용 가능

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 1. 기본 정책을 사용하는 간편한 메서드
    public Money calculateMovieFee(Screening screening) {
        return calculateMovieFee(screening, this.discountPolicy);
    }
    
    // 2. 일시적으로 다른 정책을 적용하는 유연한 메서드
    public Money calculateMovieFee(Screening screening, 
                                    DiscountPolicy discountPolicy) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}

// 사용
Money fee1 = movie.calculateMovieFee(screening);  // 기본 정책 사용
Money fee2 = movie.calculateMovieFee(screening, new PercentDiscountPolicy(...));  // 특정 정책 사용
```

#### ⚖️ 트레이드오프 분석

| 측면 | 완전히 결합도 제거 | 기본 객체 제공 |
|------|-------------------|---------------|
| **결합도** | 매우 낮음 | 약간 높음 (구체 클래스 사용) |
| **사용성** | 약간 불편함 | 매우 편리함 |
| **유연성** | 항상 명시적 주입 필요 | 기본 + 선택적 주입 가능 |
| **적합한 경우** | 다양한 정책이 동등하게 중요 | 하나의 정책이 주로 사용됨 |

#### 💎 설계 원칙

```
"설계는 트레이드오프의 산물이다"

구체 클래스에 의존하게 되더라도
클래스의 사용성이 더 중요하다면
결합도를 높이는 방향으로 코드를 작성할 수 있다.

가급적 구체 클래스 의존성을 제거하되,
사용성을 위해 필요하다면 적절히 타협하라.
```

#### 🏭 Factory 패턴으로 두 마리 토끼 잡기

```java
// 모든 결합도를 Factory로 모아서 사용성과 유연성 동시 확보
public class MovieFactory {
    public Movie createAvatarMovie() {
        return new Movie(
            "아바타",
            Duration.ofMinutes(120),
            Money.wons(10000),
            new AmountDiscountPolicy(...)
        );
    }
    
    public Movie createDiscountedMovie(String title, Duration runningTime, 
                                       Money fee, DiscountPolicy policy) {
        return new Movie(title, runningTime, fee, policy);
    }
}

// Movie 클래스는 결합도가 낮게 유지되고
// Factory가 사용성을 책임진다
```

### 2.7 표준 클래스에 대한 의존은 해롭지 않다

#### 💡 변경 가능성과 의존성

```
의존성이 불편한 이유는?
→ 변경에 대한 영향을 암시하기 때문

변경될 확률이 거의 없는 클래스라면?
→ 의존성이 문제가 되지 않는다!
```

#### ✅ 안전한 의존성

**JDK 표준 클래스**

```java
public abstract class DiscountPolicy {
    // ArrayList는 JDK 표준 클래스
    private List<DiscountCondition> conditions = new ArrayList<>();
    //                                            ↑
    //                                  new를 사용해도 OK!
    
    public DiscountPolicy(DiscountCondition... conditions) {
        this.conditions = Arrays.asList(conditions);
        //                ↑
        //        Arrays도 표준 클래스
    }
}
```

**왜 안전한가?**
- JDK는 거의 변경되지 않음
- 변경되더라도 하위 호환성 보장
- 전 세계 개발자들이 사용하므로 신뢰도 높음

#### 🎯 추상적인 타입 사용의 이점

```java
// ❌ 구체적인 타입에 의존
private ArrayList<DiscountCondition> conditions = new ArrayList<>();

// ✅ 추상적인 타입에 의존 (권장)
private List<DiscountCondition> conditions = new ArrayList<>();
//     ↑
//  인터페이스 타입 사용
```

**추상적인 타입 사용의 장점:**

```java
public class DiscountPolicy {
    private List<DiscountCondition> conditions;
    
    // List 인터페이스 타입이므로 다양한 구현체로 교체 가능
    public void switchConditions(List<DiscountCondition> conditions) {
        this.conditions = conditions;
    }
}

// 사용
policy.switchConditions(new ArrayList<>());     // ArrayList
policy.switchConditions(new LinkedList<>());    // LinkedList
policy.switchConditions(new Vector<>());        // Vector
// 모두 가능! (List 인터페이스 구현체)
```

#### 📊 의존성 판단 기준

| 클래스 종류 | 변경 가능성 | 의존 방식 | 예시 |
|------------|-----------|----------|------|
| **표준 클래스** | 거의 없음 | 직접 생성 OK | `ArrayList`, `HashMap`, `String` |
| **안정적인 외부 라이브러리** | 매우 낮음 | 직접 생성 가능 | 잘 관리되는 오픈소스 |
| **프로젝트 내 구체 클래스** | 높음 | 추상화를 통해 의존 | 비즈니스 로직 클래스 |
| **자주 변경되는 클래스** | 매우 높음 | 반드시 추상화 필요 | 변경 가능한 정책, 전략 |

#### 💎 설계 습관

```
표준 클래스에 의존하더라도:

✅ 가능한 추상적인 타입 사용
   List<T> list = new ArrayList<>();  // O
   ArrayList<T> list = new ArrayList<>();  // X

✅ 의존성을 명시적으로 드러내기
   public DiscountPolicy(List<DiscountCondition> conditions)

✅ 변경 가능성을 항상 고려
   "이 클래스가 변경될 가능성이 있는가?"
```

#### 🎯 핵심 원칙

```
변경될 확률이 낮은 표준 클래스라도
추상화와 명시적 의존성을 사용하는 것은
좋은 설계 습관이다.

의존성에 의한 영향이 적더라도
일관된 설계 원칙을 적용하라.
```


---

## 3. 컨텍스트 확장하기

> 📂 **코드**: [`NoneDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/NoneDiscountPolicy.java) | [`OverlappedDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/OverlappedDiscountPolicy.java)

### 3.1 설계의 유연성 검증

#### 🎯 유연한 설계의 증거

```
Movie를 수정하지 않고도
새로운 컨텍스트에 대응할 수 있는가?

이것이 설계 유연성의 척도이다.
```

### 3.2 케이스 1: 할인 혜택이 없는 영화

#### ❌ 잘못된 접근: 예외 케이스 추가

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        this(title, runningTime, fee, null);  // null로 "할인 없음" 표현
    }
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    public Money calculateMovieFee(Screening screening) {
        // 예외 케이스 처리
        if (discountPolicy == null) {  // 특별한 처리 필요
            return fee;
        }
        
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

**문제점:**
1. null 체크 로직 추가 → 복잡도 증가
2. Movie와 DiscountPolicy의 협력 방식 변경
3. NullPointerException 위험
4. 일관성 없는 설계

#### ✅ 올바른 접근: 새로운 정책 클래스 추가

```java
// 할인하지 않는 정책을 명시적인 클래스로 표현
public class NoneDiscountPolicy extends DiscountPolicy {
    @Override
    protected Money getDiscountAmount(Screening screening) {
        return Money.ZERO;  // 할인 금액 0원
    }
}
```

```java
// Movie는 전혀 수정할 필요 없음!
Movie starWars = new Movie(
    "스타워즈",
    Duration.ofMinutes(210),
    Money.wons(10000),
    new NoneDiscountPolicy()  // 할인 없음을 명시적으로 표현
);
```

**장점:**
1. Movie 클래스는 수정 불필요 (OCP 준수)
2. null 체크 불필요
3. 일관된 협력 방식 유지
4. 명시적이고 이해하기 쉬움

### 3.3 케이스 2: 중복 할인 정책

#### 🎯 요구사항

```
"금액 할인과 비율 할인을 동시에 적용하고 싶다"

예) 800원 할인 + 10% 할인을 중복 적용
```

#### ❌ 잘못된 접근: Movie 클래스 수정

```java
public class Movie {
    private List<DiscountPolicy> discountPolicies;  // 리스트로 변경
    
    public Movie(String title, Duration runningTime, Money fee,
                 List<DiscountPolicy> discountPolicies) {  // 시그니처 변경
        this.discountPolicies = discountPolicies;
    }
    
    public Money calculateMovieFee(Screening screening) {
        Money result = fee;
        // 반복문으로 모든 정책 적용
        for (DiscountPolicy policy : discountPolicies) {
            result = result.minus(policy.calculateDiscountAmount(screening));
        }
        return result;
    }
}
```

**문제점:**
1. Movie 클래스 수정 필요 (OCP 위반)
2. 기존 단일 할인 정책 코드와 호환성 깨짐
3. 버그 발생 가능성
4. 협력 방식 변경

#### ✅ 올바른 접근: Composite 패턴

```java
// 여러 할인 정책을 조합하는 새로운 정책
public class OverlappedDiscountPolicy extends DiscountPolicy {
    private List<DiscountPolicy> discountPolicies = new ArrayList<>();
    
    public OverlappedDiscountPolicy(DiscountPolicy... discountPolicies) {
        this.discountPolicies = Arrays.asList(discountPolicies);
    }
    
    @Override
    protected Money getDiscountAmount(Screening screening) {
        Money result = Money.ZERO;
        for (DiscountPolicy each : discountPolicies) {
            result = result.plus(each.calculateDiscountAmount(screening));
        }
        return result;
    }
}
```

```java
// Movie는 전혀 수정할 필요 없음!
Movie avatar = new Movie(
    "아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new OverlappedDiscountPolicy(
        new AmountDiscountPolicy(Money.wons(800),
            new SequenceCondition(1),
            new SequenceCondition(10)
        ),
        new PercentDiscountPolicy(0.1,
            new PeriodCondition(DayOfWeek.MONDAY,
                LocalTime.of(10, 0), LocalTime.of(11, 59))
        )
    )
);
```

**장점:**
1. Movie 클래스는 수정 불필요 (OCP 준수)
2. 기존 코드와 완벽한 호환성
3. Composite 패턴으로 무한 조합 가능
4. 일관된 협력 방식 유지

### 3.4 조합 가능한 행동

#### 🎨 선언적인 설계

```java
// 복잡한 할인 정책도 객체 조합으로 선언적으로 표현
Movie movie = new Movie(
    "복잡한 영화",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new OverlappedDiscountPolicy(                      // 중복 할인
        new AmountDiscountPolicy(Money.wons(800),      // 금액 할인
            new SequenceCondition(1),                   // 첫 번째 상영
            new SequenceCondition(10)                   // 열 번째 상영
        ),
        new PercentDiscountPolicy(0.1,                 // 비율 할인
            new PeriodCondition(                        // 기간 조건
                DayOfWeek.MONDAY,
                LocalTime.of(10, 0),
                LocalTime.of(11, 59)
            )
        )
    )
);
```

#### 💡 읽기 쉬운 코드

```
위 코드를 읽는 것만으로:

"이 영화는
 - 첫 번째 상영과 열 번째 상영에는 800원 할인
 - 월요일 10시~12시 상영에는 10% 할인
 - 두 조건이 맞으면 중복 적용"

을 즉시 이해할 수 있다!
```

#### 🎯 유연한 설계의 특징

```
1. 객체가 "어떻게(How)" 하는지가 아니라
   "무엇을(What)" 하는지를 표현

2. 작은 객체들의 행동을 조합하여
   새로운 행동 생성 가능

3. 코드 수정 없이
   객체 조합 변경만으로 확장 가능

4. 선언적이고 직관적인 코드
```

#### 📊 확장 가능성 비교

**경직된 설계:**
```
새로운 요구사항
    ↓
Movie 클래스 수정
    ↓
버그 위험 증가
    ↓
테스트 다시 필요
    ↓
배포 위험 증가
```

**유연한 설계:**
```
새로운 요구사항
    ↓
새로운 DiscountPolicy 구현 추가
    ↓
기존 코드 수정 불필요
    ↓
Movie는 안전하게 유지
    ↓
안전한 확장
```


---

## 4. 실전 예제

### 예제 1: 주문 시스템

#### ❌ Before: 높은 결합도

```java
public class OrderService {
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // ❌ Order 내부를 깊이 파고듦
        if (order.getCustomer().getAddress().getCity().equals("서울")) {
            // ❌ 복잡한 조건 판단
            if (order.getItems().stream()
                    .mapToDouble(item -> item.getPrice() * item.getQuantity())
                    .sum() > 50000) {
                // ❌ 직접 상태 변경
                order.setDeliveryFee(0);
            } else {
                order.setDeliveryFee(3000);
            }
        }
        
        // ❌ 상태 묻고 변경
        if (order.getStatus() == OrderStatus.PENDING) {
            order.setStatus(OrderStatus.CONFIRMED);
        }
    }
}
```

**문제점:**
1. **기차 충돌**: `order.getCustomer().getAddress().getCity()`
2. **강한 결합도**: Order, Customer, Address, OrderItem의 내부 구조에 의존
3. **낮은 응집도**: 비즈니스 로직이 Service에 분산
4. **Tell, Don't Ask 위반**: 상태를 묻고 직접 변경

#### ✅ After: 느슨한 결합도

```java
public class OrderService {
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // ✅ 묻지 말고 시켜라
        order.applyDeliveryFee();
        order.confirm();
    }
}

public class Order {
    private Customer customer;
    private List<OrderItem> items;
    private Money deliveryFee;
    private OrderStatus status;
    
    // ✅ 정보 전문가: Order가 배송비 계산 책임
    public void applyDeliveryFee() {
        if (customer.livesInSeoul() && isOverMinimumAmount()) {
            this.deliveryFee = Money.ZERO;
        } else {
            this.deliveryFee = Money.wons(3000);
        }
    }
    
    // ✅ 내부에서만 사용하는 private 메서드
    private boolean isOverMinimumAmount() {
        return calculateTotal().isGreaterThan(Money.wons(50000));
    }
    
    // ✅ 명령: 상태 변경 책임도 Order에
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("대기 중인 주문만 확정할 수 있습니다");
        }
        this.status = OrderStatus.CONFIRMED;
    }
    
    // ✅ 쿼리: 상태 변경 없음
    public boolean isConfirmed() {
        return status == OrderStatus.CONFIRMED;
    }
    
    // ✅ 정보 전문가: Order가 자신의 총액 계산
    private Money calculateTotal() {
        return items.stream()
                   .map(OrderItem::calculateAmount)
                   .reduce(Money.ZERO, Money::plus);
    }
}

public class Customer {
    private Address address;
    
    // ✅ 의도를 드러내는 메서드
    public boolean livesInSeoul() {
        return address.isInCity("서울");
    }
}

public class OrderItem {
    private Money price;
    private int quantity;
    
    // ✅ 정보 전문가: 자신의 금액 계산
    public Money calculateAmount() {
        return price.times(quantity);
    }
}
```

**개선 효과:**

| 측면 | Before | After |
|------|--------|-------|
| **결합도** | Order, Customer, Address, OrderItem의 내부 구조에 모두 의존 | OrderService는 Order에만 의존 |
| **응집도** | 비즈니스 로직이 Service에 흩어짐 | 각 객체가 자신의 책임만 수행 |
| **변경 영향** | Address 구조 변경 → Service 수정 필요 | Address 변경 → Customer만 수정 |
| **테스트** | 모든 객체를 준비해야 테스트 가능 | 각 객체를 독립적으로 테스트 |

---

### 예제 2: 게시판 시스템

#### ❌ Before: 절차적 설계

```java
public class PostService {
    public void updatePost(Long postId, PostUpdateRequest request) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException());
        
        // ❌ 상태를 묻고 판단
        if (post.getAuthor().getId().equals(request.getUserId())) {
            // ❌ 직접 상태 변경
            post.setTitle(request.getTitle());
            post.setContent(request.getContent());
            post.setUpdatedAt(LocalDateTime.now());
            
            // ❌ 비즈니스 로직이 서비스에
            if (post.getContent().length() > 10000) {
                throw new ContentTooLongException();
            }
            
            postRepository.save(post);
        } else {
            throw new UnauthorizedException();
        }
    }
    
    public List<Post> getPublishedPosts() {
        List<Post> posts = postRepository.findAll();
        
        // ❌ 필터링 로직이 서비스에
        return posts.stream()
            .filter(post -> post.getStatus() == PostStatus.PUBLISHED)
            .filter(post -> post.getPublishedAt() != null)
            .filter(post -> post.getPublishedAt().isBefore(LocalDateTime.now()))
            .collect(Collectors.toList());
    }
}
```

#### ✅ After: 객체지향 설계

```java
public class PostService {
    public void updatePost(Long postId, PostUpdateRequest request) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException());
        
        // ✅ 묻지 말고 시켜라
        post.update(
            request.getUserId(),
            request.getTitle(),
            request.getContent()
        );
        
        postRepository.save(post);
    }
    
    public List<Post> getPublishedPosts() {
        List<Post> posts = postRepository.findAll();
        
        // ✅ Post에게 판단 위임
        return posts.stream()
            .filter(Post::isPublished)
            .collect(Collectors.toList());
    }
}

public class Post {
    private Long id;
    private User author;
    private String title;
    private String content;
    private PostStatus status;
    private LocalDateTime publishedAt;
    private LocalDateTime updatedAt;
    
    // ✅ 모든 로직이 Post 내부에
    public void update(Long userId, String newTitle, String newContent) {
        // ✅ 권한 검증도 Post가 수행
        validateAuthor(userId);
        
        // ✅ 비즈니스 규칙도 Post가 관리
        validateContent(newContent);
        
        this.title = newTitle;
        this.content = newContent;
        this.updatedAt = LocalDateTime.now();
    }
    
    private void validateAuthor(Long userId) {
        if (!author.hasId(userId)) {
            throw new UnauthorizedException("작성자만 수정할 수 있습니다");
        }
    }
    
    private void validateContent(String content) {
        if (content.length() > 10000) {
            throw new ContentTooLongException("내용이 너무 깁니다");
        }
    }
    
    // ✅ 쿼리: 발행 여부 판단
    public boolean isPublished() {
        return status == PostStatus.PUBLISHED 
            && publishedAt != null 
            && publishedAt.isBefore(LocalDateTime.now());
    }
}

public class User {
    private Long id;
    
    // ✅ 의도를 드러내는 인터페이스
    public boolean hasId(Long id) {
        return this.id.equals(id);
    }
}
```

---

### 예제 3: 결제 시스템

#### 의존성 주입을 통한 유연한 설계

```java
// ❌ Before: 구체 클래스에 강하게 결합
public class PaymentService {
    private CreditCardPaymentGateway gateway;
    
    public PaymentService() {
        // ❌ 내부에서 직접 생성
        this.gateway = new CreditCardPaymentGateway();
    }
    
    public PaymentResult process(Payment payment) {
        return gateway.charge(payment);
    }
}
```

```java
// ✅ After: 추상화에 의존
public class PaymentService {
    private PaymentGateway gateway;
    
    // ✅ 생성자를 통한 의존성 주입
    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
    
    // ✅ Setter를 통한 런타임 변경 가능
    public void setGateway(PaymentGateway gateway) {
        this.gateway = gateway;
    }
    
    public PaymentResult process(Payment payment) {
        return gateway.charge(payment);
    }
}

// 추상화
public interface PaymentGateway {
    PaymentResult charge(Payment payment);
}

// 다양한 구현체
public class CreditCardPaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Payment payment) {
        // 신용카드 결제 로직
    }
}

public class KakaoPayGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Payment payment) {
        // 카카오페이 결제 로직
    }
}

public class NaverPayGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Payment payment) {
        // 네이버페이 결제 로직
    }
}
```

**사용 예시:**

```java
// 신용카드 결제 서비스
PaymentService creditCardService = new PaymentService(
    new CreditCardPaymentGateway()
);

// 카카오페이 결제 서비스
PaymentService kakaoPayService = new PaymentService(
    new KakaoPayGateway()
);

// 런타임에 게이트웨이 변경
creditCardService.setGateway(new NaverPayGateway());
```


---

## 5. 패턴별 실전 적용 가이드

### 1. 추상화에 의존하라

#### Step 1: 구체 클래스 의존성 찾기

```java
// 코드 리뷰 체크리스트
□ 필드 타입이 구체 클래스인가?
□ 생성자/메서드에서 new를 사용하는가?
□ 특정 구현에 결합되어 있는가?
```

#### Step 2: 인터페이스 추출

```java
// Before
public class NotificationService {
    private EmailSender emailSender;
    
    public NotificationService() {
        this.emailSender = new EmailSender();
    }
}

// After: 인터페이스 추출
public interface MessageSender {
    void send(String to, String message);
}

public class EmailSender implements MessageSender {
    @Override
    public void send(String to, String message) {
        // 이메일 전송
    }
}

public class SmsSender implements MessageSender {
    @Override
    public void send(String to, String message) {
        // SMS 전송
    }
}

public class NotificationService {
    private MessageSender sender;
    
    public NotificationService(MessageSender sender) {
        this.sender = sender;
    }
}
```

#### Step 3: 유연성 확인

```java
// 다양한 구현으로 쉽게 교체 가능
NotificationService emailNotification = 
    new NotificationService(new EmailSender());

NotificationService smsNotification = 
    new NotificationService(new SmsSender());

NotificationService pushNotification = 
    new NotificationService(new PushNotificationSender());
```

---

### 2. 명시적인 의존성 사용하기

#### Step 1: 숨겨진 의존성 찾기

```java
// 패턴: 내부에서 new 사용
public class SomeClass {
    private Dependency dep;
    
    public SomeClass() {
        this.dep = new ConcreteDependency();
    }
}
```

#### Step 2: 의존성을 외부로 드러내기

```java
// Before: 숨겨진 의존성
public class ReportGenerator {
    private DataSource dataSource;
    
    public ReportGenerator() {
        this.dataSource = new DatabaseDataSource();
    }
}

// After: 명시적인 의존성
public class ReportGenerator {
    private DataSource dataSource;
    
    // 생성자 시그니처로 의존성이 명확히 드러남
    public ReportGenerator(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

#### Step 3: 테스트 용이성 확인

```java
// 명시적 의존성 덕분에 테스트가 쉬워짐
@Test
void testReportGeneration() {
    // Mock 객체 주입 가능
    DataSource mockDataSource = mock(DataSource.class);
    ReportGenerator generator = new ReportGenerator(mockDataSource);
    
    // 테스트 수행
}
```

---

### 3. new를 신중하게 사용하기

#### Step 1: new 사용 위치 파악

```java
// 현재 코드에서 new 사용 찾기
□ 비즈니스 로직 클래스에서 new 사용?
□ 특정 구현에 결합되는가?
□ 테스트하기 어려운가?
```

#### Step 2: 생성 책임 분리

```java
// Before: 사용과 생성이 섞임
public class OrderProcessor {
    public void process(Order order) {
        // ❌ 사용하는 곳에서 생성
        PaymentValidator validator = new CreditCardValidator();
        
        if (validator.validate(order.getPayment())) {
            // 처리 로직
        }
    }
}

// After: 생성 책임을 외부로
public class OrderProcessor {
    private PaymentValidator validator;
    
    // ✅ 생성된 객체를 받음
    public OrderProcessor(PaymentValidator validator) {
        this.validator = validator;
    }
    
    public void process(Order order) {
        if (validator.validate(order.getPayment())) {
            // 처리 로직
        }
    }
}
```

#### Step 3: Factory 패턴 고려

```java
// 복잡한 생성 로직은 Factory로
public class PaymentValidatorFactory {
    public static PaymentValidator create(PaymentType type) {
        switch (type) {
            case CREDIT_CARD:
                return new CreditCardValidator();
            case BANK_TRANSFER:
                return new BankTransferValidator();
            case MOBILE_PAYMENT:
                return new MobilePaymentValidator();
            default:
                throw new IllegalArgumentException();
        }
    }
}

// 사용
OrderProcessor processor = new OrderProcessor(
    PaymentValidatorFactory.create(PaymentType.CREDIT_CARD)
);
```

---

### 4. 컨텍스트 독립성 확보하기

#### Step 1: 컨텍스트 의존성 찾기

```java
// 질문 리스트
□ 이 클래스가 특정 환경을 가정하는가?
□ 특정 구현 방식에 의존하는가?
□ 다른 프로젝트에서 재사용 가능한가?
```

#### Step 2: 추상화 레벨 높이기

```java
// Before: 특정 컨텍스트에 의존
public class UserService {
    public void notifyUser(User user, String message) {
        // ❌ 이메일로만 알림
        String email = user.getEmail();
        EmailSender.send(email, message);
    }
}

// After: 컨텍스트 독립적
public class UserService {
    private NotificationSender sender;
    
    public UserService(NotificationSender sender) {
        this.sender = sender;
    }
    
    public void notifyUser(User user, String message) {
        // ✅ 어떤 방식이든 가능
        sender.send(user.getContactInfo(), message);
    }
}
```

#### Step 3: 다양한 컨텍스트에서 검증

```java
// 이메일 알림 컨텍스트
UserService emailService = new UserService(new EmailNotificationSender());

// SMS 알림 컨텍스트  
UserService smsService = new UserService(new SmsNotificationSender());

// 푸시 알림 컨텍스트
UserService pushService = new UserService(new PushNotificationSender());

// 모두 동일한 UserService 코드를 재사용!
```

---

### 5. 의존성 해결 방법 선택하기

#### 결정 가이드

```java
// 1. 생성자 주입 (가장 권장)
public class Service {
    private final Dependency dep;
    
    public Service(Dependency dep) {
        this.dep = dep;
    }
}
// 사용: 필수 의존성, 불변성 보장

// 2. Setter 주입
public class Service {
    private Dependency dep;
    
    public void setDependency(Dependency dep) {
        this.dep = dep;
    }
}
// 사용: 선택적 의존성, 런타임 변경 필요

// 3. 메서드 인자
public class Service {
    public void execute(Dependency dep) {
        dep.doSomething();
    }
}
// 사용: 일시적 협력, 호출마다 다른 객체
```

#### 의사결정 플로우차트

```
의존성이 필수인가?
    ├─ YES → 생성자 주입
    └─ NO → Setter 주입 or 메서드 인자
    
런타임에 변경이 필요한가?
    ├─ YES → Setter 주입 추가
    └─ NO → 생성자만 사용
    
매번 다른 객체가 필요한가?
    ├─ YES → 메서드 인자
    └─ NO → 생성자 or Setter
```


---

## 6. 핵심 정리

### 🎯 의존성 관리의 핵심 원칙

#### 1. 추상화에 의존하라

```java
// ❌ 구체 클래스 의존
private AmountDiscountPolicy policy;

// ✅ 추상화 의존
private DiscountPolicy policy;
```

```
의존 대상이 추상적일수록 결합도가 낮아진다.

구체 클래스 < 추상 클래스 < 인터페이스
```

#### 2. 명시적인 의존성을 사용하라

```java
// ❌ 숨겨진 의존성
public Movie(String title) {
    this.policy = new AmountDiscountPolicy(...);
}

// ✅ 명시적인 의존성
public Movie(String title, DiscountPolicy policy) {
    this.policy = policy;# Chapter 08. 의존성 관리하기

> *"협력을 위해서는 의존성이 필요하지만, 과도한 의존성은 애플리케이션을 수정하기 어렵게 만든다."*

## 📌 핵심 개념

이 장에서는 **충분히 협력적이면서도 유연한 객체**를 만들기 위해 의존성을 관리하는 방법을 학습합니다.

### 🎯 학습 목표

- 의존성의 본질과 변경에 미치는 영향 이해하기
- 런타임 의존성과 컴파일타임 의존성의 차이 파악하기
- 결합도를 낮추는 구체적인 기법 익히기
- 유연한 설계를 위한 의존성 관리 원칙 적용하기

---

## 📖 목차

1. [의존성 이해하기](#1-의존성-이해하기)
2. [유연한 설계](#2-유연한-설계)
3. [컨텍스트 확장하기](#3-컨텍스트-확장하기)
4. [실전 예제](#4-실전-예제)
5. [패턴별 실전 적용 가이드](#5-패턴별-실전-적용-가이드)
6. [핵심 정리](#6-핵심-정리)


---

## 1. 의존성 이해하기

> 📂 **코드**: [`Movie.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/Movie.java) | [`DiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/DiscountPolicy.java)

### 1.1 변경과 의존성

#### 🔍 의존성의 정의

```java
public class PeriodCondition implements DiscountCondition {
    private DayOfWeek dayOfWeek;        // DayOfWeek에 의존
    private LocalTime startTime;         // LocalTime에 의존
    private LocalTime endTime;           // LocalTime에 의존

    public boolean isSatisfiedBy(Screening screening) {  // Screening에 의존
        return screening.getStartTime().getDayOfWeek().equals(dayOfWeek) &&
                startTime.compareTo(screening.getStartTime().toLocalTime()) <= 0 &&
                endTime.compareTo(screening.getStartTime().toLocalTime()) >= 0;
    }
}
```

**의존성의 두 가지 의미:**

| 시점 | 의미 | 설명 |
|------|------|------|
| **실행 시점** | 존재 필요성 | 의존하는 객체가 정상적으로 동작하려면 실행 시에 의존 대상이 **반드시 존재**해야 함 |
| **구현 시점** | 변경 전파 | 의존 대상이 변경되면 의존하는 객체도 **함께 변경**될 수 있음 |

#### 💡 핵심 통찰

```
의존성은 방향성을 가지며 항상 단방향이다.

PeriodCondition → Screening (O)
Screening → PeriodCondition (X)

Screening이 변경되면 PeriodCondition이 영향을 받지만,
PeriodCondition이 변경되어도 Screening은 영향을 받지 않는다.
```

### 1.2 의존성 전이 (Transitive Dependency)

#### 📊 의존성 전이의 메커니즘

```
PeriodCondition → Screening → Movie → Money

A가 B에 의존하고, B가 C에 의존하면
A는 C에 간접적으로 의존하게 된다.
```

**의존성의 종류:**

| 종류 | 설명 | 코드 가시성 | 예시 |
|------|------|------------|------|
| **직접 의존성** (Direct Dependency) | 한 요소가 다른 요소에 직접 의존 | 명시적으로 드러남 | `PeriodCondition → Screening` |
| **간접 의존성** (Indirect Dependency) | 의존성 전이에 의해 영향 전파 | 암묵적으로 숨겨짐 | `PeriodCondition → Movie` |

#### ⚠️ 중요한 점

```
의존성 전이는 "가능성"을 의미할 뿐이다.

실제 전이 여부는:
1. 변경의 방향
2. 캡슐화의 정도
에 따라 달라진다.
```

### 1.3 런타임 의존성 vs 컴파일타임 의존성

#### 🎭 두 의존성의 차이

```java
// 컴파일타임: Movie는 DiscountPolicy 인터페이스에만 의존
public class Movie {
    private DiscountPolicy discountPolicy;  // 추상화에 의존
    
    public Movie(String title, Duration runningTime, Money fee, 
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

```java
// 런타임: 실제로는 구체적인 정책 인스턴스와 협력
Movie avatar = new Movie("아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new AmountDiscountPolicy(...)  // 구체 클래스 인스턴스
);

Movie titanic = new Movie("타이타닉",
    Duration.ofMinutes(180),
    Money.wons(11000),
    new PercentDiscountPolicy(...)  // 다른 구체 클래스 인스턴스
);
```

#### 📊 관계 다이어그램

```
[컴파일타임 의존성]
Movie ---------> DiscountPolicy (인터페이스)
                      ↑
                      |
        +-------------+-------------+
        |                           |
AmountDiscountPolicy    PercentDiscountPolicy


[런타임 의존성]
avatar(Movie) --------> AmountDiscountPolicy 인스턴스

titanic(Movie) -------> PercentDiscountPolicy 인스턴스
```

#### 💎 유연한 설계의 핵심

```
컴파일타임 구조와 런타임 구조 사이의 거리가 멀면 멀수록
설계는 더 유연해지고 재사용 가능해진다.

동일한 소스코드 구조로 다양한 실행 구조를 만들 수 있어야 한다.
```

### 1.4 컨텍스트 독립성

#### 🎯 개념 이해

**컨텍스트 독립성 (Context Independence):**
클래스가 사용될 특정한 문맥에 대해 **최소한의 가정만으로** 이뤄져 있다면 다른 문맥에서 재사용하기 수월해진다.

#### 비교: 컨텍스트 의존적 vs 독립적

```java
// ❌ 컨텍스트 의존적 - 구체적인 할인 정책에 강하게 결합
public class Movie {
    private AmountDiscountPolicy discountPolicy;  // 특정 정책에 의존
    
    public Movie(String title, Duration runningTime, Money fee) {
        this.discountPolicy = new AmountDiscountPolicy(...);  // 직접 생성
    }
    
    // 금액 할인 정책 문맥에서만 사용 가능
    // 다른 할인 정책 문맥에서는 재사용 불가능
}
```

```java
// ✅ 컨텍스트 독립적 - 추상화에 의존
public class Movie {
    private DiscountPolicy discountPolicy;  // 추상화에 의존
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {  // 외부에서 주입
        this.discountPolicy = discountPolicy;
    }
    
    // 어떤 할인 정책 문맥에서도 재사용 가능
    // 금액 할인, 비율 할인, 중복 할인 등 모든 정책과 협력 가능
}
```

#### 🎨 컨텍스트 독립성의 장점

```
1. 다양한 문맥에서 재사용 가능
   - 금액 할인 정책 문맥
   - 비율 할인 정책 문맥
   - 할인 없는 문맥
   - 중복 할인 문맥

2. 응집력 있는 객체 구성
   - 책임이 명확함
   - 변경이 용이함

3. 유연한 설계
   - 객체 구성 방법을 재설정 가능
   - 변경 가능한 시스템으로 나아갈 수 있음
```

### 1.5 의존성 해결하기

#### 🔧 의존성 해결의 정의

```
컴파일타임 의존성을 실행 컨텍스트에 맞는
적절한 런타임 의존성으로 교체하는 것
```

#### 세 가지 해결 방법

**1️⃣ 생성자를 통한 의존성 해결**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}

// 사용
Movie avatar = new Movie("아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new AmountDiscountPolicy(...)  // 생성 시점에 의존성 해결
);
```

**장점:**
- 객체 생성 시점에 의존성이 명확히 설정됨
- 객체의 상태가 항상 완전함 (불완전한 상태 방지)
- 가장 권장되는 방식

**2️⃣ Setter 메서드를 통한 의존성 해결**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}

// 사용
Movie avatar = new Movie("아바타", Duration.ofMinutes(120), Money.wons(10000));
avatar.setDiscountPolicy(new AmountDiscountPolicy(...));  // 생성 후 의존성 해결

// 런타임에 의존성 변경 가능
avatar.setDiscountPolicy(new PercentDiscountPolicy(...));
```

**장점:**
- 실행 시점에 의존 대상 변경 가능
- 유연성 향상

**단점:**
- 객체 생성 후 setter 호출 전까지 상태 불완전
- NullPointerException 위험

**3️⃣ 메서드 인자를 통한 의존성 해결**

```java
public class Movie {
    public Money calculateMovieFee(Screening screening, 
                                    DiscountPolicy discountPolicy) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}

// 사용
avatar.calculateMovieFee(screening, new AmountDiscountPolicy(...));
```

**적합한 경우:**
- 협력 대상에 대해 지속적인 의존 관계가 필요 없을 때
- 메서드 실행 시마다 의존 대상이 달라져야 할 때
- 일시적인 의존성이 필요할 때

#### 🎯 최선의 조합

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 생성자로 기본 의존성 설정 (상태 완전성 보장)
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // Setter로 유연성 제공 (런타임 변경 가능)
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
}
```

**이점:**
- 객체 생성 시 완전한 상태 보장 (생성자)
- 필요시 런타임에 유연한 변경 가능 (setter)


---

## 2. 유연한 설계

> 📂 **코드**: [`AmountDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/AmountDiscountPolicy.java) | [`PercentDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/PercentDiscountPolicy.java)

### 2.1 의존성과 결합도

#### 🤔 의존성이 모두 나쁜가?

```
NO! 의존성은 객체들의 협력을 가능하게 만드는 매개체이다.

문제는 의존성의 "존재"가 아니라 의존성의 "정도"이다.
```

#### 📊 의존성 vs 결합도

| 개념 | 의미 | 표현 방식 | 예시 |
|------|------|----------|------|
| **의존성** | 관계의 유무 | 존재한다 / 존재하지 않는다 | "A는 B에 의존한다" |
| **결합도** | 관계의 정도 | 강하다 / 느슨하다 | "A와 B는 강하게 결합되어 있다" |

#### 🎯 바람직한 의존성

```java
// ❌ 바람직하지 못한 의존성 - 구체 클래스에 의존
public class Movie {
    private PercentDiscountPolicy discountPolicy;  // 강한 결합도
    
    // PercentDiscountPolicy 문맥에서만 재사용 가능
    // AmountDiscountPolicy가 필요하면 Movie를 수정해야 함
}
```

```java
// ✅ 바람직한 의존성 - 추상화에 의존
public class Movie {
    private DiscountPolicy discountPolicy;  // 느슨한 결합도
    
    // 모든 DiscountPolicy 문맥에서 재사용 가능
    // 새로운 정책 추가 시 Movie는 변경 불필요
}
```

#### 💡 결합도 판단 기준

```
바람직한 의존성 (느슨한 결합도):
✅ 다양한 환경에서 재사용 가능
✅ 컨텍스트에 독립적
✅ 변경의 영향이 제한적

바람직하지 못한 의존성 (강한 결합도):
❌ 특정 문맥에서만 재사용 가능
❌ 컨텍스트에 강하게 결합
❌ 변경의 영향이 광범위
```

### 2.2 지식이 결합을 낳는다

#### 🧠 결합도의 본질

```
결합도는 한 요소가 다른 요소에 대해 알고 있는 정보의 양으로 결정된다.

더 많이 안다 = 더 강하게 결합된다 = 더 적은 컨텍스트에서 재사용 가능
더 적게 안다 = 더 느슨하게 결합된다 = 더 많은 컨텍스트에서 재사용 가능
```

#### 📊 지식의 양과 결합도 비교

**강한 결합도 - 구체 클래스에 의존**

```java
public class Movie {
    private AmountDiscountPolicy discountPolicy;
    
    // Movie가 알아야 하는 것들:
    // 1. 할인 정책이 AmountDiscountPolicy라는 사실
    // 2. AmountDiscountPolicy의 생성 방법
    // 3. AmountDiscountPolicy의 구체적인 메서드
    // 4. AmountDiscountPolicy가 속한 상속 계층
    // 5. AmountDiscountPolicy의 내부 구현 방식
    
    // → 너무 많은 지식 = 강한 결합도
}
```

**느슨한 결합도 - 추상화에 의존**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // Movie가 알아야 하는 것들:
    // 1. 할인 정책이 존재한다는 사실
    // 2. calculateDiscountAmount 메시지를 이해한다는 사실
    
    // → 최소한의 지식 = 느슨한 결합도
}
```

### 2.3 추상화에 의존하라

#### 🎨 추상화의 힘

**추상화 (Abstraction):**
특정 절차나 물체를 의도적으로 생략하거나 감춤으로써 복잡도를 극복하는 방법

```
추상화는 대상에 대해 알아야 하는 지식의 양을 줄여
결합도를 느슨하게 유지할 수 있게 한다.
```

#### 📊 추상화 레벨과 결합도

```
높은 결합도 (많은 지식 필요)
    ↓
구체 클래스 의존성 (Concrete Class Dependency)
    - 구체적인 구현까지 모두 알아야 함
    - 가장 많은 지식 필요
    
    ↓ (추상화 수준 상승)
    
추상 클래스 의존성 (Abstract Class Dependency)
    - 메서드 내부 구현은 감춤
    - 자식 클래스 종류는 감춤
    - 하지만 상속 계층은 알아야 함
    
    ↓ (추상화 수준 상승)
    
인터페이스 의존성 (Interface Dependency)
    - 오직 퍼블릭 인터페이스만 알면 됨
    - 상속 계층도 알 필요 없음
    - 가장 적은 지식 필요
    ↓
낮은 결합도 (적은 지식 필요)
```

#### 💻 코드로 보는 추상화 레벨

**Level 1: 구체 클래스 의존 (높은 결합도)**

```java
public class Movie {
    private AmountDiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // 구체적인 구현까지 알아야 함
        this.discountPolicy = new AmountDiscountPolicy(
            Money.wons(800),
            new SequenceCondition(1),
            new SequenceCondition(10)
        );
    }
    
    // AmountDiscountPolicy에 강하게 결합
    // PercentDiscountPolicy로 변경하려면 Movie를 수정해야 함
}
```

**Level 2: 추상 클래스 의존 (중간 결합도)**

```java
public abstract class DiscountPolicy {
    protected abstract Money getDiscountAmount(Screening screening);
    
    // 템플릿 메서드는 공개되지만
    // 구체적인 할인 금액 계산 방식은 감춰짐
}

public class Movie {
    private DiscountPolicy discountPolicy;  // 추상 클래스 의존
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // DiscountPolicy의 자식이라면 모두 협력 가능
    // 하지만 DiscountPolicy 계층이라는 것은 알아야 함
}
```

**Level 3: 인터페이스 의존 (낮은 결합도)**

```java
public interface DiscountPolicy {
    Money calculateDiscountAmount(Screening screening);
    // 순수한 인터페이스만 정의
}

public class Movie {
    private DiscountPolicy discountPolicy;  // 인터페이스 의존
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // DiscountPolicy 인터페이스를 구현한 어떤 객체와도 협력 가능
    // 상속 계층, 구현 방식 등 내부 사항을 전혀 알 필요 없음
}
```

#### 💎 핵심 원칙

```
의존하는 대상이 더 추상적일수록 결합도는 더 낮아진다.

구체 클래스 < 추상 클래스 < 인터페이스
(높은 결합도)              (낮은 결합도)
```

### 2.4 명시적인 의존성

#### 🎯 명시적 vs 숨겨진 의존성

**숨겨진 의존성 (Hidden Dependency) ❌**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // 내부에서 구체 클래스 직접 생성
        this.discountPolicy = new AmountDiscountPolicy(...);
    }
    
    // 문제점:
    // 1. 퍼블릭 인터페이스에 의존성이 드러나지 않음
    // 2. Movie가 AmountDiscountPolicy에 의존한다는 것을 알려면 내부 구현을 봐야 함
    // 3. 다른 정책으로 변경하려면 Movie 클래스를 수정해야 함
}

// 사용 시
Movie avatar = new Movie("아바타", Duration.ofMinutes(120), Money.wons(10000));
// 어떤 할인 정책을 사용하는지 알 수 없음!
```

**명시적인 의존성 (Explicit Dependency) ✅**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 생성자 시그니처에 의존성 명시
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 장점:
    // 1. 퍼블릭 인터페이스를 통해 의존성이 명확히 드러남
    // 2. 코드를 읽는 것만으로 의존 관계를 파악 가능
    // 3. Movie 수정 없이 실행 컨텍스트에서 적절한 정책 선택 가능
}

// 사용 시
Movie avatar = new Movie("아바타", 
    Duration.ofMinutes(120), 
    Money.wons(10000),
    new AmountDiscountPolicy(...)  // 어떤 정책을 사용하는지 명확!
);
```

#### 📊 비교 분석

| 측면 | 숨겨진 의존성 | 명시적인 의존성 |
|------|-------------|----------------|
| **가시성** | 내부 구현을 봐야 알 수 있음 | 퍼블릭 인터페이스에 드러남 |
| **유연성** | 변경하려면 클래스 수정 필요 | 실행 시점에 자유롭게 교체 |
| **재사용성** | 특정 컨텍스트에 고정됨 | 다양한 컨텍스트에서 재사용 |
| **테스트** | 특정 구현에 고정되어 테스트 어려움 | Mock 객체 주입으로 쉽게 테스트 |

#### 💡 명시적 의존성의 다양한 형태

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 1. 생성자를 통한 명시적 의존성
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 2. Setter를 통한 명시적 의존성
    public void setDiscountPolicy(DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 3. 메서드 인자를 통한 명시적 의존성
    public Money calculateMovieFee(Screening screening, 
                                    DiscountPolicy discountPolicy) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

#### 🎯 핵심 원칙

```
"클래스가 다른 클래스에 의존하는 것은 부끄러운 일이 아니다."

의존성은 협력을 가능하게 하므로 바람직하다.
경계해야 할 것은 의존성 자체가 아니라 의존성을 감추는 것이다.

숨겨진 의존성을 밝은 곳으로 드러내면
설계가 유연하고 재사용 가능해진다.
```

### 2.5 new는 해롭다

#### ⚠️ new의 문제점

**1️⃣ 구체 클래스에 대한 직접적인 의존**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // new는 구체 클래스 이름을 직접 명시해야 함
        this.discountPolicy = new AmountDiscountPolicy(...);
        //                        ↑
        //                   구체 클래스에 의존
        //                   추상화가 아님!
    }
}
```

**문제:** Movie가 추상화(DiscountPolicy)가 아닌 구체 클래스(AmountDiscountPolicy)에 의존하게 되어 결합도가 높아진다.

**2️⃣ 생성자 인자에 대한 지식 필요**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        this.discountPolicy = new AmountDiscountPolicy(
            Money.wons(800),                    // 할인 금액
            new SequenceCondition(1),            // 첫 번째 조건
            new SequenceCondition(10),           // 두 번째 조건
            new PeriodCondition(                 // 세 번째 조건
                DayOfWeek.MONDAY,
                LocalTime.of(10, 0),
                LocalTime.of(11, 59)
            ),
            new PeriodCondition(                 // 네 번째 조건
                DayOfWeek.THURSDAY,
                LocalTime.of(10, 0),
                LocalTime.of(20, 59)
            )
        );
    }
    
    // Movie가 알아야 하는 것들:
    // - AmountDiscountPolicy의 생성자 시그니처
    // - 필요한 인자의 타입과 순서
    // - SequenceCondition, PeriodCondition의 생성 방법
    // - DayOfWeek, LocalTime의 사용 방법
    
    // → 너무 많은 지식 = 강한 결합도!
}
```

#### 📊 의존성 폭발

```
Movie가 new를 사용하면:

Movie
 ├─> AmountDiscountPolicy (직접 의존)
 ├─> Money (인자로 필요)
 ├─> SequenceCondition (인자로 필요)
 ├─> PeriodCondition (인자로 필요)
 ├─> DayOfWeek (PeriodCondition 인자)
 └─> LocalTime (PeriodCondition 인자)

하나의 new가 6개의 의존성을 만들어낸다!
```

#### ✅ 해결 방법: 사용과 생성의 책임 분리

```java
// ❌ Before: 사용 + 생성 책임을 모두 가짐
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        // 생성 책임
        this.discountPolicy = new AmountDiscountPolicy(...);
    }
    
    public Money calculateMovieFee(Screening screening) {
        // 사용 책임
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

```java
// ✅ After: 사용 책임만 가짐
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 생성 책임은 클라이언트로 이동
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    // 오직 사용 책임만
    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

```java
// 클라이언트가 생성 책임을 담당
Movie avatar = new Movie(
    "아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new AmountDiscountPolicy(         // 클라이언트가 생성
        Money.wons(800),
        new SequenceCondition(1),
        new SequenceCondition(10),
        new PeriodCondition(DayOfWeek.MONDAY, LocalTime.of(10, 0), LocalTime.of(11, 59)),
        new PeriodCondition(DayOfWeek.THURSDAY, LocalTime.of(10, 0), LocalTime.of(20, 59))
    )
);
```

#### 🎯 개선 효과

```
Before:
Movie → AmountDiscountPolicy (강한 결합)
      → Money
      → SequenceCondition
      → PeriodCondition
      → DayOfWeek
      → LocalTime

After:
Movie → DiscountPolicy (느슨한 결합)

모든 구체적인 의존성은 클라이언트로 이동!
```

#### 💡 설계 개선 정리

```
✅ 사용과 생성의 책임 분리
   - Movie는 사용만
   - 클라이언트는 생성만

✅ 의존성을 생성자에 명시적으로 드러냄
   - public Movie(..., DiscountPolicy discountPolicy)

✅ 구체 클래스가 아닌 추상화에 의존
   - DiscountPolicy (인터페이스/추상 클래스)

✅ 객체 생성 책임을 클라이언트로 이동
   - Movie 내부에서 new 제거
```

### 2.6 가끔은 생성해도 무방하다

#### 🤔 트레이드오프: 결합도 vs 사용성

```
완벽하게 결합도를 낮추면 사용성이 떨어질 수 있다.

생성자에 항상 모든 의존성을 전달하는 것은
클라이언트 입장에서 번거로울 수 있다.
```

#### 💡 해결책: 복수의 생성자 제공

**기본 객체를 제공하는 간편한 생성자 + 유연한 생성자**

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 1. 기본 정책을 사용하는 간편한 생성자
    public Movie(String title, Duration runningTime, Money fee) {
        this(title, runningTime, fee, new AmountDiscountPolicy(...));
        // 생성자 체이닝을 통해 기본 정책 설정
    }
    
    // 2. 유연한 생성자 (원하는 정책을 주입 가능)
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.title = title;
        this.runningTime = runningTime;
        this.fee = fee;
        this.discountPolicy = discountPolicy;
    }
}
```

#### 📊 사용 시나리오

```java
// 시나리오 1: 대부분의 경우 - 기본 정책 사용
Movie movie1 = new Movie("아바타", Duration.ofMinutes(120), Money.wons(10000));
// 간편하게 생성! AmountDiscountPolicy가 자동 적용

// 시나리오 2: 특별한 경우 - 다른 정책 필요
Movie movie2 = new Movie(
    "타이타닉",
    Duration.ofMinutes(180),
    Money.wons(11000),
    new PercentDiscountPolicy(...)  // 원하는 정책 명시적 주입
);

// 시나리오 3: 할인 없는 영화
Movie movie3 = new Movie(
    "인터스텔라",
    Duration.ofMinutes(169),
    Money.wons(12000),
    new NoneDiscountPolicy()  // 할인 없음
);
```

#### 🎯 메서드 오버로딩에도 적용 가능

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    // 1. 기본 정책을 사용하는 간편한 메서드
    public Money calculateMovieFee(Screening screening) {
        return calculateMovieFee(screening, this.discountPolicy);
    }
    
    // 2. 일시적으로 다른 정책을 적용하는 유연한 메서드
    public Money calculateMovieFee(Screening screening, 
                                    DiscountPolicy discountPolicy) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}

// 사용
Money fee1 = movie.calculateMovieFee(screening);  // 기본 정책 사용
Money fee2 = movie.calculateMovieFee(screening, new PercentDiscountPolicy(...));  // 특정 정책 사용
```

#### ⚖️ 트레이드오프 분석

| 측면 | 완전히 결합도 제거 | 기본 객체 제공 |
|------|-------------------|---------------|
| **결합도** | 매우 낮음 | 약간 높음 (구체 클래스 사용) |
| **사용성** | 약간 불편함 | 매우 편리함 |
| **유연성** | 항상 명시적 주입 필요 | 기본 + 선택적 주입 가능 |
| **적합한 경우** | 다양한 정책이 동등하게 중요 | 하나의 정책이 주로 사용됨 |

#### 💎 설계 원칙

```
"설계는 트레이드오프의 산물이다"

구체 클래스에 의존하게 되더라도
클래스의 사용성이 더 중요하다면
결합도를 높이는 방향으로 코드를 작성할 수 있다.

가급적 구체 클래스 의존성을 제거하되,
사용성을 위해 필요하다면 적절히 타협하라.
```

#### 🏭 Factory 패턴으로 두 마리 토끼 잡기

```java
// 모든 결합도를 Factory로 모아서 사용성과 유연성 동시 확보
public class MovieFactory {
    public Movie createAvatarMovie() {
        return new Movie(
            "아바타",
            Duration.ofMinutes(120),
            Money.wons(10000),
            new AmountDiscountPolicy(...)
        );
    }
    
    public Movie createDiscountedMovie(String title, Duration runningTime, 
                                       Money fee, DiscountPolicy policy) {
        return new Movie(title, runningTime, fee, policy);
    }
}

// Movie 클래스는 결합도가 낮게 유지되고
// Factory가 사용성을 책임진다
```

### 2.7 표준 클래스에 대한 의존은 해롭지 않다

#### 💡 변경 가능성과 의존성

```
의존성이 불편한 이유는?
→ 변경에 대한 영향을 암시하기 때문

변경될 확률이 거의 없는 클래스라면?
→ 의존성이 문제가 되지 않는다!
```

#### ✅ 안전한 의존성

**JDK 표준 클래스**

```java
public abstract class DiscountPolicy {
    // ArrayList는 JDK 표준 클래스
    private List<DiscountCondition> conditions = new ArrayList<>();
    //                                            ↑
    //                                  new를 사용해도 OK!
    
    public DiscountPolicy(DiscountCondition... conditions) {
        this.conditions = Arrays.asList(conditions);
        //                ↑
        //        Arrays도 표준 클래스
    }
}
```

**왜 안전한가?**
- JDK는 거의 변경되지 않음
- 변경되더라도 하위 호환성 보장
- 전 세계 개발자들이 사용하므로 신뢰도 높음

#### 🎯 추상적인 타입 사용의 이점

```java
// ❌ 구체적인 타입에 의존
private ArrayList<DiscountCondition> conditions = new ArrayList<>();

// ✅ 추상적인 타입에 의존 (권장)
private List<DiscountCondition> conditions = new ArrayList<>();
//     ↑
//  인터페이스 타입 사용
```

**추상적인 타입 사용의 장점:**

```java
public class DiscountPolicy {
    private List<DiscountCondition> conditions;
    
    // List 인터페이스 타입이므로 다양한 구현체로 교체 가능
    public void switchConditions(List<DiscountCondition> conditions) {
        this.conditions = conditions;
    }
}

// 사용
policy.switchConditions(new ArrayList<>());     // ArrayList
policy.switchConditions(new LinkedList<>());    // LinkedList
policy.switchConditions(new Vector<>());        // Vector
// 모두 가능! (List 인터페이스 구현체)
```

#### 📊 의존성 판단 기준

| 클래스 종류 | 변경 가능성 | 의존 방식 | 예시 |
|------------|-----------|----------|------|
| **표준 클래스** | 거의 없음 | 직접 생성 OK | `ArrayList`, `HashMap`, `String` |
| **안정적인 외부 라이브러리** | 매우 낮음 | 직접 생성 가능 | 잘 관리되는 오픈소스 |
| **프로젝트 내 구체 클래스** | 높음 | 추상화를 통해 의존 | 비즈니스 로직 클래스 |
| **자주 변경되는 클래스** | 매우 높음 | 반드시 추상화 필요 | 변경 가능한 정책, 전략 |

#### 💎 설계 습관

```
표준 클래스에 의존하더라도:

✅ 가능한 추상적인 타입 사용
   List<T> list = new ArrayList<>();  // O
   ArrayList<T> list = new ArrayList<>();  // X

✅ 의존성을 명시적으로 드러내기
   public DiscountPolicy(List<DiscountCondition> conditions)

✅ 변경 가능성을 항상 고려
   "이 클래스가 변경될 가능성이 있는가?"
```

#### 🎯 핵심 원칙

```
변경될 확률이 낮은 표준 클래스라도
추상화와 명시적 의존성을 사용하는 것은
좋은 설계 습관이다.

의존성에 의한 영향이 적더라도
일관된 설계 원칙을 적용하라.
```


---

## 3. 컨텍스트 확장하기

> 📂 **코드**: [`NoneDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/NoneDiscountPolicy.java) | [`OverlappedDiscountPolicy.java`](https://github.com/eternity-oop/object/blob/master/chapter08/src/main/java/org/eternity/movie/pricing/OverlappedDiscountPolicy.java)

### 3.1 설계의 유연성 검증

#### 🎯 유연한 설계의 증거

```
Movie를 수정하지 않고도
새로운 컨텍스트에 대응할 수 있는가?

이것이 설계 유연성의 척도이다.
```

### 3.2 케이스 1: 할인 혜택이 없는 영화

#### ❌ 잘못된 접근: 예외 케이스 추가

```java
public class Movie {
    private DiscountPolicy discountPolicy;
    
    public Movie(String title, Duration runningTime, Money fee) {
        this(title, runningTime, fee, null);  // null로 "할인 없음" 표현
    }
    
    public Movie(String title, Duration runningTime, Money fee,
                 DiscountPolicy discountPolicy) {
        this.discountPolicy = discountPolicy;
    }
    
    public Money calculateMovieFee(Screening screening) {
        // 예외 케이스 처리
        if (discountPolicy == null) {  // 특별한 처리 필요
            return fee;
        }
        
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

**문제점:**
1. null 체크 로직 추가 → 복잡도 증가
2. Movie와 DiscountPolicy의 협력 방식 변경
3. NullPointerException 위험
4. 일관성 없는 설계

#### ✅ 올바른 접근: 새로운 정책 클래스 추가

```java
// 할인하지 않는 정책을 명시적인 클래스로 표현
public class NoneDiscountPolicy extends DiscountPolicy {
    @Override
    protected Money getDiscountAmount(Screening screening) {
        return Money.ZERO;  // 할인 금액 0원
    }
}
```

```java
// Movie는 전혀 수정할 필요 없음!
Movie starWars = new Movie(
    "스타워즈",
    Duration.ofMinutes(210),
    Money.wons(10000),
    new NoneDiscountPolicy()  // 할인 없음을 명시적으로 표현
);
```

**장점:**
1. Movie 클래스는 수정 불필요 (OCP 준수)
2. null 체크 불필요
3. 일관된 협력 방식 유지
4. 명시적이고 이해하기 쉬움

### 3.3 케이스 2: 중복 할인 정책

#### 🎯 요구사항

```
"금액 할인과 비율 할인을 동시에 적용하고 싶다"

예) 800원 할인 + 10% 할인을 중복 적용
```

#### ❌ 잘못된 접근: Movie 클래스 수정

```java
public class Movie {
    private List<DiscountPolicy> discountPolicies;  // 리스트로 변경
    
    public Movie(String title, Duration runningTime, Money fee,
                 List<DiscountPolicy> discountPolicies) {  // 시그니처 변경
        this.discountPolicies = discountPolicies;
    }
    
    public Money calculateMovieFee(Screening screening) {
        Money result = fee;
        // 반복문으로 모든 정책 적용
        for (DiscountPolicy policy : discountPolicies) {
            result = result.minus(policy.calculateDiscountAmount(screening));
        }
        return result;
    }
}
```

**문제점:**
1. Movie 클래스 수정 필요 (OCP 위반)
2. 기존 단일 할인 정책 코드와 호환성 깨짐
3. 버그 발생 가능성
4. 협력 방식 변경

#### ✅ 올바른 접근: Composite 패턴

```java
// 여러 할인 정책을 조합하는 새로운 정책
public class OverlappedDiscountPolicy extends DiscountPolicy {
    private List<DiscountPolicy> discountPolicies = new ArrayList<>();
    
    public OverlappedDiscountPolicy(DiscountPolicy... discountPolicies) {
        this.discountPolicies = Arrays.asList(discountPolicies);
    }
    
    @Override
    protected Money getDiscountAmount(Screening screening) {
        Money result = Money.ZERO;
        for (DiscountPolicy each : discountPolicies) {
            result = result.plus(each.calculateDiscountAmount(screening));
        }
        return result;
    }
}
```

```java
// Movie는 전혀 수정할 필요 없음!
Movie avatar = new Movie(
    "아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new OverlappedDiscountPolicy(
        new AmountDiscountPolicy(Money.wons(800),
            new SequenceCondition(1),
            new SequenceCondition(10)
        ),
        new PercentDiscountPolicy(0.1,
            new PeriodCondition(DayOfWeek.MONDAY,
                LocalTime.of(10, 0), LocalTime.of(11, 59))
        )
    )
);
```

**장점:**
1. Movie 클래스는 수정 불필요 (OCP 준수)
2. 기존 코드와 완벽한 호환성
3. Composite 패턴으로 무한 조합 가능
4. 일관된 협력 방식 유지

### 3.4 조합 가능한 행동

#### 🎨 선언적인 설계

```java
// 복잡한 할인 정책도 객체 조합으로 선언적으로 표현
Movie movie = new Movie(
    "복잡한 영화",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new OverlappedDiscountPolicy(                      // 중복 할인
        new AmountDiscountPolicy(Money.wons(800),      // 금액 할인
            new SequenceCondition(1),                   // 첫 번째 상영
            new SequenceCondition(10)                   // 열 번째 상영
        ),
        new PercentDiscountPolicy(0.1,                 // 비율 할인
            new PeriodCondition(                        // 기간 조건
                DayOfWeek.MONDAY,
                LocalTime.of(10, 0),
                LocalTime.of(11, 59)
            )
        )
    )
);
```

#### 💡 읽기 쉬운 코드

```
위 코드를 읽는 것만으로:

"이 영화는
 - 첫 번째 상영과 열 번째 상영에는 800원 할인
 - 월요일 10시~12시 상영에는 10% 할인
 - 두 조건이 맞으면 중복 적용"

을 즉시 이해할 수 있다!
```

#### 🎯 유연한 설계의 특징

```
1. 객체가 "어떻게(How)" 하는지가 아니라
   "무엇을(What)" 하는지를 표현

2. 작은 객체들의 행동을 조합하여
   새로운 행동 생성 가능

3. 코드 수정 없이
   객체 조합 변경만으로 확장 가능

4. 선언적이고 직관적인 코드
```

#### 📊 확장 가능성 비교

**경직된 설계:**
```
새로운 요구사항
    ↓
Movie 클래스 수정
    ↓
버그 위험 증가
    ↓
테스트 다시 필요
    ↓
배포 위험 증가
```

**유연한 설계:**
```
새로운 요구사항
    ↓
새로운 DiscountPolicy 구현 추가
    ↓
기존 코드 수정 불필요
    ↓
Movie는 안전하게 유지
    ↓
안전한 확장
```


---

## 4. 실전 예제

### 예제 1: 주문 시스템

#### ❌ Before: 높은 결합도

```java
public class OrderService {
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // ❌ Order 내부를 깊이 파고듦
        if (order.getCustomer().getAddress().getCity().equals("서울")) {
            // ❌ 복잡한 조건 판단
            if (order.getItems().stream()
                    .mapToDouble(item -> item.getPrice() * item.getQuantity())
                    .sum() > 50000) {
                // ❌ 직접 상태 변경
                order.setDeliveryFee(0);
            } else {
                order.setDeliveryFee(3000);
            }
        }
        
        // ❌ 상태 묻고 변경
        if (order.getStatus() == OrderStatus.PENDING) {
            order.setStatus(OrderStatus.CONFIRMED);
        }
    }
}
```

**문제점:**
1. **기차 충돌**: `order.getCustomer().getAddress().getCity()`
2. **강한 결합도**: Order, Customer, Address, OrderItem의 내부 구조에 의존
3. **낮은 응집도**: 비즈니스 로직이 Service에 분산
4. **Tell, Don't Ask 위반**: 상태를 묻고 직접 변경

#### ✅ After: 느슨한 결합도

```java
public class OrderService {
    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId);
        
        // ✅ 묻지 말고 시켜라
        order.applyDeliveryFee();
        order.confirm();
    }
}

public class Order {
    private Customer customer;
    private List<OrderItem> items;
    private Money deliveryFee;
    private OrderStatus status;
    
    // ✅ 정보 전문가: Order가 배송비 계산 책임
    public void applyDeliveryFee() {
        if (customer.livesInSeoul() && isOverMinimumAmount()) {
            this.deliveryFee = Money.ZERO;
        } else {
            this.deliveryFee = Money.wons(3000);
        }
    }
    
    // ✅ 내부에서만 사용하는 private 메서드
    private boolean isOverMinimumAmount() {
        return calculateTotal().isGreaterThan(Money.wons(50000));
    }
    
    // ✅ 명령: 상태 변경 책임도 Order에
    public void confirm() {
        if (status != OrderStatus.PENDING) {
            throw new IllegalStateException("대기 중인 주문만 확정할 수 있습니다");
        }
        this.status = OrderStatus.CONFIRMED;
    }
    
    // ✅ 쿼리: 상태 변경 없음
    public boolean isConfirmed() {
        return status == OrderStatus.CONFIRMED;
    }
    
    // ✅ 정보 전문가: Order가 자신의 총액 계산
    private Money calculateTotal() {
        return items.stream()
                   .map(OrderItem::calculateAmount)
                   .reduce(Money.ZERO, Money::plus);
    }
}

public class Customer {
    private Address address;
    
    // ✅ 의도를 드러내는 메서드
    public boolean livesInSeoul() {
        return address.isInCity("서울");
    }
}

public class OrderItem {
    private Money price;
    private int quantity;
    
    // ✅ 정보 전문가: 자신의 금액 계산
    public Money calculateAmount() {
        return price.times(quantity);
    }
}
```

**개선 효과:**

| 측면 | Before | After |
|------|--------|-------|
| **결합도** | Order, Customer, Address, OrderItem의 내부 구조에 모두 의존 | OrderService는 Order에만 의존 |
| **응집도** | 비즈니스 로직이 Service에 흩어짐 | 각 객체가 자신의 책임만 수행 |
| **변경 영향** | Address 구조 변경 → Service 수정 필요 | Address 변경 → Customer만 수정 |
| **테스트** | 모든 객체를 준비해야 테스트 가능 | 각 객체를 독립적으로 테스트 |

---

### 예제 2: 게시판 시스템

#### ❌ Before: 절차적 설계

```java
public class PostService {
    public void updatePost(Long postId, PostUpdateRequest request) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException());
        
        // ❌ 상태를 묻고 판단
        if (post.getAuthor().getId().equals(request.getUserId())) {
            // ❌ 직접 상태 변경
            post.setTitle(request.getTitle());
            post.setContent(request.getContent());
            post.setUpdatedAt(LocalDateTime.now());
            
            // ❌ 비즈니스 로직이 서비스에
            if (post.getContent().length() > 10000) {
                throw new ContentTooLongException();
            }
            
            postRepository.save(post);
        } else {
            throw new UnauthorizedException();
        }
    }
    
    public List<Post> getPublishedPosts() {
        List<Post> posts = postRepository.findAll();
        
        // ❌ 필터링 로직이 서비스에
        return posts.stream()
            .filter(post -> post.getStatus() == PostStatus.PUBLISHED)
            .filter(post -> post.getPublishedAt() != null)
            .filter(post -> post.getPublishedAt().isBefore(LocalDateTime.now()))
            .collect(Collectors.toList());
    }
}
```

#### ✅ After: 객체지향 설계

```java
public class PostService {
    public void updatePost(Long postId, PostUpdateRequest request) {
        Post post = postRepository.findById(postId)
            .orElseThrow(() -> new PostNotFoundException());
        
        // ✅ 묻지 말고 시켜라
        post.update(
            request.getUserId(),
            request.getTitle(),
            request.getContent()
        );
        
        postRepository.save(post);
    }
    
    public List<Post> getPublishedPosts() {
        List<Post> posts = postRepository.findAll();
        
        // ✅ Post에게 판단 위임
        return posts.stream()
            .filter(Post::isPublished)
            .collect(Collectors.toList());
    }
}

public class Post {
    private Long id;
    private User author;
    private String title;
    private String content;
    private PostStatus status;
    private LocalDateTime publishedAt;
    private LocalDateTime updatedAt;
    
    // ✅ 모든 로직이 Post 내부에
    public void update(Long userId, String newTitle, String newContent) {
        // ✅ 권한 검증도 Post가 수행
        validateAuthor(userId);
        
        // ✅ 비즈니스 규칙도 Post가 관리
        validateContent(newContent);
        
        this.title = newTitle;
        this.content = newContent;
        this.updatedAt = LocalDateTime.now();
    }
    
    private void validateAuthor(Long userId) {
        if (!author.hasId(userId)) {
            throw new UnauthorizedException("작성자만 수정할 수 있습니다");
        }
    }
    
    private void validateContent(String content) {
        if (content.length() > 10000) {
            throw new ContentTooLongException("내용이 너무 깁니다");
        }
    }
    
    // ✅ 쿼리: 발행 여부 판단
    public boolean isPublished() {
        return status == PostStatus.PUBLISHED 
            && publishedAt != null 
            && publishedAt.isBefore(LocalDateTime.now());
    }
}

public class User {
    private Long id;
    
    // ✅ 의도를 드러내는 인터페이스
    public boolean hasId(Long id) {
        return this.id.equals(id);
    }
}
```

---

### 예제 3: 결제 시스템

#### 의존성 주입을 통한 유연한 설계

```java
// ❌ Before: 구체 클래스에 강하게 결합
public class PaymentService {
    private CreditCardPaymentGateway gateway;
    
    public PaymentService() {
        // ❌ 내부에서 직접 생성
        this.gateway = new CreditCardPaymentGateway();
    }
    
    public PaymentResult process(Payment payment) {
        return gateway.charge(payment);
    }
}
```

```java
// ✅ After: 추상화에 의존
public class PaymentService {
    private PaymentGateway gateway;
    
    // ✅ 생성자를 통한 의존성 주입
    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;
    }
    
    // ✅ Setter를 통한 런타임 변경 가능
    public void setGateway(PaymentGateway gateway) {
        this.gateway = gateway;
    }
    
    public PaymentResult process(Payment payment) {
        return gateway.charge(payment);
    }
}

// 추상화
public interface PaymentGateway {
    PaymentResult charge(Payment payment);
}

// 다양한 구현체
public class CreditCardPaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Payment payment) {
        // 신용카드 결제 로직
    }
}

public class KakaoPayGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Payment payment) {
        // 카카오페이 결제 로직
    }
}

public class NaverPayGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Payment payment) {
        // 네이버페이 결제 로직
    }
}
```

**사용 예시:**

```java
// 신용카드 결제 서비스
PaymentService creditCardService = new PaymentService(
    new CreditCardPaymentGateway()
);

// 카카오페이 결제 서비스
PaymentService kakaoPayService = new PaymentService(
    new KakaoPayGateway()
);

// 런타임에 게이트웨이 변경
creditCardService.setGateway(new NaverPayGateway());
```


---

## 5. 패턴별 실전 적용 가이드

### 1. 추상화에 의존하라

#### Step 1: 구체 클래스 의존성 찾기

```java
// 코드 리뷰 체크리스트
□ 필드 타입이 구체 클래스인가?
□ 생성자/메서드에서 new를 사용하는가?
□ 특정 구현에 결합되어 있는가?
```

#### Step 2: 인터페이스 추출

```java
// Before
public class NotificationService {
    private EmailSender emailSender;
    
    public NotificationService() {
        this.emailSender = new EmailSender();
    }
}

// After: 인터페이스 추출
public interface MessageSender {
    void send(String to, String message);
}

public class EmailSender implements MessageSender {
    @Override
    public void send(String to, String message) {
        // 이메일 전송
    }
}

public class SmsSender implements MessageSender {
    @Override
    public void send(String to, String message) {
        // SMS 전송
    }
}

public class NotificationService {
    private MessageSender sender;
    
    public NotificationService(MessageSender sender) {
        this.sender = sender;
    }
}
```

#### Step 3: 유연성 확인

```java
// 다양한 구현으로 쉽게 교체 가능
NotificationService emailNotification = 
    new NotificationService(new EmailSender());

NotificationService smsNotification = 
    new NotificationService(new SmsSender());

NotificationService pushNotification = 
    new NotificationService(new PushNotificationSender());
```

---

### 2. 명시적인 의존성 사용하기

#### Step 1: 숨겨진 의존성 찾기

```java
// 패턴: 내부에서 new 사용
public class SomeClass {
    private Dependency dep;
    
    public SomeClass() {
        this.dep = new ConcreteDependency();
    }
}
```

#### Step 2: 의존성을 외부로 드러내기

```java
// Before: 숨겨진 의존성
public class ReportGenerator {
    private DataSource dataSource;
    
    public ReportGenerator() {
        this.dataSource = new DatabaseDataSource();
    }
}

// After: 명시적인 의존성
public class ReportGenerator {
    private DataSource dataSource;
    
    // 생성자 시그니처로 의존성이 명확히 드러남
    public ReportGenerator(DataSource dataSource) {
        this.dataSource = dataSource;
    }
}
```

#### Step 3: 테스트 용이성 확인

```java
// 명시적 의존성 덕분에 테스트가 쉬워짐
@Test
void testReportGeneration() {
    // Mock 객체 주입 가능
    DataSource mockDataSource = mock(DataSource.class);
    ReportGenerator generator = new ReportGenerator(mockDataSource);
    
    // 테스트 수행
}
```

---

### 3. new를 신중하게 사용하기

#### Step 1: new 사용 위치 파악

```java
// 현재 코드에서 new 사용 찾기
□ 비즈니스 로직 클래스에서 new 사용?
□ 특정 구현에 결합되는가?
□ 테스트하기 어려운가?
```

#### Step 2: 생성 책임 분리

```java
// Before: 사용과 생성이 섞임
public class OrderProcessor {
    public void process(Order order) {
        // ❌ 사용하는 곳에서 생성
        PaymentValidator validator = new CreditCardValidator();
        
        if (validator.validate(order.getPayment())) {
            // 처리 로직
        }
    }
}

// After: 생성 책임을 외부로
public class OrderProcessor {
    private PaymentValidator validator;
    
    // ✅ 생성된 객체를 받음
    public OrderProcessor(PaymentValidator validator) {
        this.validator = validator;
    }
    
    public void process(Order order) {
        if (validator.validate(order.getPayment())) {
            // 처리 로직
        }
    }
}
```

#### Step 3: Factory 패턴 고려

```java
// 복잡한 생성 로직은 Factory로
public class PaymentValidatorFactory {
    public static PaymentValidator create(PaymentType type) {
        switch (type) {
            case CREDIT_CARD:
                return new CreditCardValidator();
            case BANK_TRANSFER:
                return new BankTransferValidator();
            case MOBILE_PAYMENT:
                return new MobilePaymentValidator();
            default:
                throw new IllegalArgumentException();
        }
    }
}

// 사용
OrderProcessor processor = new OrderProcessor(
    PaymentValidatorFactory.create(PaymentType.CREDIT_CARD)
);
```

---

### 4. 컨텍스트 독립성 확보하기

#### Step 1: 컨텍스트 의존성 찾기

```java
// 질문 리스트
□ 이 클래스가 특정 환경을 가정하는가?
□ 특정 구현 방식에 의존하는가?
□ 다른 프로젝트에서 재사용 가능한가?
```

#### Step 2: 추상화 레벨 높이기

```java
// Before: 특정 컨텍스트에 의존
public class UserService {
    public void notifyUser(User user, String message) {
        // ❌ 이메일로만 알림
        String email = user.getEmail();
        EmailSender.send(email, message);
    }
}

// After: 컨텍스트 독립적
public class UserService {
    private NotificationSender sender;
    
    public UserService(NotificationSender sender) {
        this.sender = sender;
    }
    
    public void notifyUser(User user, String message) {
        // ✅ 어떤 방식이든 가능
        sender.send(user.getContactInfo(), message);
    }
}
```

#### Step 3: 다양한 컨텍스트에서 검증

```java
// 이메일 알림 컨텍스트
UserService emailService = new UserService(new EmailNotificationSender());

// SMS 알림 컨텍스트  
UserService smsService = new UserService(new SmsNotificationSender());

// 푸시 알림 컨텍스트
UserService pushService = new UserService(new PushNotificationSender());

// 모두 동일한 UserService 코드를 재사용!
```

---

### 5. 의존성 해결 방법 선택하기

#### 결정 가이드

```java
// 1. 생성자 주입 (가장 권장)
public class Service {
    private final Dependency dep;
    
    public Service(Dependency dep) {
        this.dep = dep;
    }
}
// 사용: 필수 의존성, 불변성 보장

// 2. Setter 주입
public class Service {
    private Dependency dep;
    
    public void setDependency(Dependency dep) {
        this.dep = dep;
    }
}
// 사용: 선택적 의존성, 런타임 변경 필요

// 3. 메서드 인자
public class Service {
    public void execute(Dependency dep) {
        dep.doSomething();
    }
}
// 사용: 일시적 협력, 호출마다 다른 객체
```

#### 의사결정 플로우차트

```
의존성이 필수인가?
    ├─ YES → 생성자 주입
    └─ NO → Setter 주입 or 메서드 인자
    
런타임에 변경이 필요한가?
    ├─ YES → Setter 주입 추가
    └─ NO → 생성자만 사용
    
매번 다른 객체가 필요한가?
    ├─ YES → 메서드 인자
    └─ NO → 생성자 or Setter
```


---

## 6. 핵심 정리

### 🎯 의존성 관리의 핵심 원칙

#### 1. 추상화에 의존하라

```java
// ❌ 구체 클래스 의존
private AmountDiscountPolicy policy;

// ✅ 추상화 의존
private DiscountPolicy policy;
```

```
의존 대상이 추상적일수록 결합도가 낮아진다.

구체 클래스 < 추상 클래스 < 인터페이스
```

#### 2. 명시적인 의존성을 사용하라

```java
// ❌ 숨겨진 의존성
public Movie(String title) {
    this.policy = new AmountDiscountPolicy(...);
}

// ✅ 명시적인 의존성
public Movie(String title, DiscountPolicy policy) {
    this.policy = policy;
}
```

```
의존성은 퍼블릭 인터페이스를 통해 드러내라.
숨겨진 의존성은 재사용을 방해한다.
```

#### 3. new는 신중하게 사용하라

```java
// ❌ 객체 내부에서 생성
public Movie(String title) {
    this.policy = new AmountDiscountPolicy(...);
}

// ✅ 사용과 생성의 책임 분리
public Movie(String title, DiscountPolicy policy) {
    this.policy = policy;  // 생성된 객체를 전달받음
}
```

```
사용과 생성의 책임을 분리하라.
객체를 생성하는 책임은 클라이언트로 옮겨라.
```

#### 4. 컨텍스트 독립성을 유지하라

```java
// ✅ 특정 문맥에 의존하지 않음
public class Movie {
    private DiscountPolicy policy;  // 어떤 정책이든 협력 가능
}
```

```
클래스가 특정 문맥에 대해 최소한의 가정만 하면
다양한 문맥에서 재사용할 수 있다.
```

### 💎 의존성 관리의 목표

```
┌─────────────────────────────────────┐
│                                     │
│   협력을 위해 필요한 의존성은 유지          │
│              ⬇                      │
│   변경을 방해하는 의존성은 제거            │
│                                     │
└─────────────────────────────────────┘

이것이 객체지향 설계의 핵심이다.
```

### 🎨 유연한 설계의 특징

#### 작은 객체들의 조합

```java
Movie avatar = new Movie("아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new OverlappedDiscountPolicy(
        new AmountDiscountPolicy(Money.wons(800),
            new SequenceCondition(1),
            new SequenceCondition(10)
        ),
        new PercentDiscountPolicy(0.1,
            new PeriodCondition(DayOfWeek.MONDAY,
                LocalTime.of(10, 0), LocalTime.of(11, 59))
        )
    )
);
```

**특징:**
- 작고 응집도 높은 객체들
- 명확한 책임 분리
- 조합을 통한 확장
- 선언적인 표현

### 📊 설계 품질 비교

| 측면 | 경직된 설계 | 유연한 설계 |
|------|-----------|------------|
| **의존성** | 구체 클래스에 의존 | 추상화에 의존 |
| **결합도** | 강한 결합도 | 느슨한 결합도 |
| **재사용성** | 특정 문맥에만 가능 | 다양한 문맥에서 가능 |
| **변경** | 클래스 수정 필요 | 객체 조합 변경으로 대응 |
| **확장** | 코드 수정 필요 | 새로운 클래스 추가 |
| **테스트** | 어려움 | 쉬움 (Mock 주입) |

### 🎯 실천 가이드

#### 의존성을 추가할 때 자문하라

```
□ 이 의존성이 꼭 필요한가?
□ 추상화에 의존하고 있는가?
□ 의존성이 명시적으로 드러나는가?
□ 다른 컨텍스트에서 재사용 가능한가?
□ 변경이 발생하면 어디까지 영향을 미치는가?
```

#### 코드 리뷰 체크리스트

```
□ new 키워드가 적절한 위치에 있는가?
□ 생성자가 의존성을 명시적으로 드러내는가?
□ 구체 클래스 대신 추상화에 의존하는가?
□ 표준 클래스가 아닌데 직접 생성하는가?
□ 숨겨진 의존성이 있는가?
```

### 🏆 좋은 설계의 지표

```
1. 새로운 기능 추가 시 기존 클래스를 수정하지 않음
2. 객체 조합만으로 새로운 행동 생성 가능
3. 코드를 읽는 것만으로 동작 이해 가능
4. 테스트 작성이 쉬움
5. 다양한 컨텍스트에서 재사용 가능
```

### 💡 마지막 통찰

```
"훌륭한 객체지향 설계란
 객체가 어떻게 하는지를 표현하는 것이 아니라
 객체들의 조합을 선언적으로 표현함으로써
 객체들이 무엇을 하는지를 표현하는 설계다.

 그리고 이런 설계를 창조하는 데 있어서의 핵심은
 의존성을 관리하는 것이다."
```


---

## 🔗 연결고리

### 이전 장과의 연결
- **Chapter 07**: 객체 분해 기법을 학습 → 의존성 관리의 기초
- 프로시저 추상화와 데이터 추상화 → 추상화의 중요성 이해

### 다음 장 예고
- **Chapter 09**: 유연한 설계
  - 개방-폐쇄 원칙 (OCP)
  - 의존성 역전 원칙 (DIP)
  - 생성과 사용 분리
  - Factory 패턴

---

<div align="center">

**[⬆ 목차로 돌아가기](#-목차)**

</div>

<div align="center">

**[⬅️ 이전: Chapter 07](../chapter07/README.md) | [다음: Chapter 09 ➡️](../chapter09/README.md)**

</div>_

}
```

```
의존성은 퍼블릭 인터페이스를 통해 드러내라.
숨겨진 의존성은 재사용을 방해한다.
```

#### 3. new는 신중하게 사용하라

```java
// ❌ 객체 내부에서 생성
public Movie(String title) {
    this.policy = new AmountDiscountPolicy(...);
}

// ✅ 사용과 생성의 책임 분리
public Movie(String title, DiscountPolicy policy) {
    this.policy = policy;  // 생성된 객체를 전달받음
}
```

```
사용과 생성의 책임을 분리하라.
객체를 생성하는 책임은 클라이언트로 옮겨라.
```

#### 4. 컨텍스트 독립성을 유지하라

```java
// ✅ 특정 문맥에 의존하지 않음
public class Movie {
    private DiscountPolicy policy;  // 어떤 정책이든 협력 가능
}
```

```
클래스가 특정 문맥에 대해 최소한의 가정만 하면
다양한 문맥에서 재사용할 수 있다.
```

### 💎 의존성 관리의 목표

```
┌─────────────────────────────────────┐
│                                     │
│   협력을 위해 필요한 의존성은 유지          │
│              ⬇                      │
│   변경을 방해하는 의존성은 제거            │
│                                     │
└─────────────────────────────────────┘

이것이 객체지향 설계의 핵심이다.
```

### 🎨 유연한 설계의 특징

#### 작은 객체들의 조합

```java
Movie avatar = new Movie("아바타",
    Duration.ofMinutes(120),
    Money.wons(10000),
    new OverlappedDiscountPolicy(
        new AmountDiscountPolicy(Money.wons(800),
            new SequenceCondition(1),
            new SequenceCondition(10)
        ),
        new PercentDiscountPolicy(0.1,
            new PeriodCondition(DayOfWeek.MONDAY,
                LocalTime.of(10, 0), LocalTime.of(11, 59))
        )
    )
);
```

**특징:**
- 작고 응집도 높은 객체들
- 명확한 책임 분리
- 조합을 통한 확장
- 선언적인 표현

### 📊 설계 품질 비교

| 측면 | 경직된 설계 | 유연한 설계 |
|------|-----------|------------|
| **의존성** | 구체 클래스에 의존 | 추상화에 의존 |
| **결합도** | 강한 결합도 | 느슨한 결합도 |
| **재사용성** | 특정 문맥에만 가능 | 다양한 문맥에서 가능 |
| **변경** | 클래스 수정 필요 | 객체 조합 변경으로 대응 |
| **확장** | 코드 수정 필요 | 새로운 클래스 추가 |
| **테스트** | 어려움 | 쉬움 (Mock 주입) |

### 🎯 실천 가이드

#### 의존성을 추가할 때 자문하라

```
□ 이 의존성이 꼭 필요한가?
□ 추상화에 의존하고 있는가?
□ 의존성이 명시적으로 드러나는가?
□ 다른 컨텍스트에서 재사용 가능한가?
□ 변경이 발생하면 어디까지 영향을 미치는가?
```

#### 코드 리뷰 체크리스트

```
□ new 키워드가 적절한 위치에 있는가?
□ 생성자가 의존성을 명시적으로 드러내는가?
□ 구체 클래스 대신 추상화에 의존하는가?
□ 표준 클래스가 아닌데 직접 생성하는가?
□ 숨겨진 의존성이 있는가?
```

### 🏆 좋은 설계의 지표

```
1. 새로운 기능 추가 시 기존 클래스를 수정하지 않음
2. 객체 조합만으로 새로운 행동 생성 가능
3. 코드를 읽는 것만으로 동작 이해 가능
4. 테스트 작성이 쉬움
5. 다양한 컨텍스트에서 재사용 가능
```

### 💡 마지막 통찰

```
"훌륭한 객체지향 설계란
 객체가 어떻게 하는지를 표현하는 것이 아니라
 객체들의 조합을 선언적으로 표현함으로써
 객체들이 무엇을 하는지를 표현하는 설계다.

 그리고 이런 설계를 창조하는 데 있어서의 핵심은
 의존성을 관리하는 것이다."
```


---

## 🔗 연결고리

### 이전 장과의 연결
- **Chapter 07**: 객체 분해 기법을 학습 → 의존성 관리의 기초
- 프로시저 추상화와 데이터 추상화 → 추상화의 중요성 이해

### 다음 장 예고
- **Chapter 09**: 유연한 설계
  - 개방-폐쇄 원칙 (OCP)
  - 의존성 역전 원칙 (DIP)
  - 생성과 사용 분리
  - Factory 패턴

---

<div align="center">

**[⬆ 목차로 돌아가기](../README.md)**

</div>

<div align="center">

**[⬅️ 이전: Chapter 07](../chapter07/README.md) | [다음: Chapter 09 ➡️](../chapter09/README.md)**

</div>_
