# 학습 기록: 스프링 핵심 원리 질의응답 (완전 복기)

## 2026-05-09 학습 세션

오늘 진행된 모든 질문과 답변을 토씨 하나 틀리지 않고, 저의 모든 기술적 설명과 비유를 포함하여 대화 내용 그대로 기록한 문서입니다.

---

### Q1. static void main() 코드에서 `MemberService memberService = new MemberServiceImpl();`, `OrderService orderService = new OrderServiceImpl();` 이렇게 두 개의 코드가 나와 있는데 이건 각각 memberService와 orderService라는 비어있는 객체를 만드는 거죠?

**A:** 아니요, 단순히 "비어있는 객체"를 만드는 것은 아닙니다. 이 코드에는 자바의 **다형성(Polymorphism)**과 **객체 생성**이라는 중요한 개념이 담겨 있습니다.

구체적으로 어떤 의미인지 3가지 포인트로 설명해 드릴게요.

**1. "비어있지 않은" 객체입니다**
`new MemberServiceImpl()`을 호출하는 순간, 컴퓨터 메모리에는 이 클래스가 정의한 **모든 로직과 변수가 담긴 실제 객체(인스턴스)**가 생성됩니다.

현재 코드를 보면 `MemberServiceImpl` 내부에는 이미 이런 코드가 들어있죠?
```java
private final MemberRepository memberRepository = new MemoryMemberRepository();
```
따라서 `memberService` 객체가 만들어질 때, 그 안에는 회원 정보를 저장하고 관리할 수 있는 **`MemoryMemberRepository`라는 실무자(저장소)까지 함께 준비**됩니다. 즉, 바로 일을 할 수 있는 상태의 "완성된 기계"를 만드는 것과 같습니다.

**2. 왼쪽(인터페이스) vs 오른쪽(구현체)**
*   **왼쪽 (`MemberService`)**: "이 객체는 앞으로 어떤 **역할**을 할 것이다"라는 선언입니다. (주문하기, 가입하기 등)
*   **오른쪽 (`new MemberServiceImpl()`)**: "그 역할을 수행할 **실제 모델**은 이것이다"라고 지정하는 것입니다.

이것을 현실 세계에 비유하면 이렇습니다.
> **역할**: "나는 배달 서비스가 필요해" (`MemberService`)
> **실제 모델**: "그 서비스로 '쿠팡이츠'를 쓸 거야" (`new MemberServiceImpl()`)

**3. 현재 코드의 "한계" (중요!)**
질문하신 코드의 핵심은 **"내가 직접 객체를 찍어서 만들고 있다"**는 점입니다.

지금은 `OrderService`가 필요할 때 마다 직접 `new OrderServiceImpl()`을 하고 있죠? 이렇게 되면 나중에 `OrderServiceImpl`이 아니라 `AdvancedOrderServiceImpl`로 바꾸고 싶을 때, **이 `main` 메서드가 있는 모든 소스 코드를 찾아가서 직접 수정**해야 합니다.

**요약하자면:**
1.  비어있는 게 아니라, **비즈니스 로직과 내부 의존성(저장소 등)을 모두 갖춘 객체**를 만드는 것입니다.
2.  하지만 이렇게 직접 `new`를 하는 방식은 나중에 **유지보수하기가 어렵기 때문에**, 나중에 스프링을 배우면서 이 `new`를 대신 해주는 기능을 배우시게 될 겁니다.

지금 작성하신 코드는 "배달 앱을 쓰기 위해 직접 오토바이를 사고 배달원을 고용하는 단계"라고 보시면 됩니다. 나중에는 "전화 한 통으로 배달을 요청(스프링 컨테이너)"하는 방식으로 변할 거예요!

---

### Q2. 그러면 이 뜻은 orderService를 필요할 때마다 Member를 한 명 임의로 만들고 있다는 뜻이네요?

**A:** 조금 더 정확히 짚어드리면, **"주문을 테스트해보기 위해 테스트용 회원 데이터(Member)를 하나 수동으로 만들어서 저장소에 넣어두고 있는 상황"**이라고 보시면 됩니다.

그런데 이 코드에는 지금 공부하시는 단계에서 **가장 중요한 "함정"**이 하나 숨어 있습니다. 이 부분을 이해하시면 스프링의 핵심인 DI(의존관계 주입)를 왜 배우는지 바로 무릎을 탁 치게 되실 거예요!

**1. "주문을 하려면 회원이 있어야 한다" (도메인 로직)**
`OrderServiceImpl.createOrder()` 메서드를 나중에 열어보시면, 내부에서 `memberRepository.findById(memberId)`를 호출합니다. 즉, **"회원 등급(VIP인지 아닌지)을 알아야 할인을 해줄 수 있기 때문에"** 회원을 먼저 만들어야 하는 게 맞습니다.

**2. 숨겨진 심각한 문제: "서로 다른 저장소"**
질문하신 `main` 메서드 코드를 다시 볼게요.
```java
MemberService memberService = new MemberServiceImpl();
OrderService orderService = new OrderServiceImpl();
```

현재 작성하신 각 클래스의 내부를 보면 이렇습니다:
*   `MemberServiceImpl` 안에도 `new MemoryMemberRepository()`가 있습니다.
*   `OrderServiceImpl` 안에도 `new MemoryMemberRepository()`가 있습니다.

**자, 여기서 문제가 발생합니다.** 자바에서 `new`를 두 번 하면, 메모리에는 **서로 다른 두 개의 저장소**가 만들어집니다.
1.  `memberService.join(member)`를 하면 **A 저장소**에 회원이 저장됩니다.
2.  `orderService.createOrder()`를 하면 **B 저장소**에서 회원을 찾으려고 합니다.
3.  **결과**: B 저장소에는 회원이 없기 때문에 에러가 나거나 회원을 찾을 수 없게 됩니다. (지금은 `static`을 써서 운 좋게 공유될 수도 있지만, 설계상으로는 매우 위험합니다.)

**3. 그래서 무엇을 배우게 되나요?**
지금 질문하신 코드는 **"주문 서비스(오토바이)를 새로 살 때마다, 회원 저장소(주유소)도 새로 하나씩 만들어서 들고 다니는 꼴"**입니다. 비효율적이고 데이터도 안 맞죠.

앞으로 강의에서 배우실 내용은 이겁니다:
*   "야, `new` 좀 그만해!"
*   "저장소(`MemoryMemberRepository`)는 딱 **하나만** 만들어서, `MemberService`랑 `OrderService`가 **나눠 갖게(주입받게) 하자!**"

**요약하자면:**
*   네, 주문 기능을 확인하려고 임의의 회원(`Member`)을 수동으로 만들고 있는 게 맞습니다.
*   하지만 더 큰 문제는 `Service`들을 만들 때마다 **내부의 저장소들까지 제각각 새로 만들어지고 있다는 점**입니다.
*   이 문제를 해결하는 과정이 바로 **"AppConfig"**의 등장과 **"스프링 컨테이너"**로의 발전 과정입니다.

이 "저장소가 두 개 생기는 문제"를 직접 코드로 확인해보시면 스프링이 왜 대단한지 훨씬 더 깊게 체감하실 수 있을 거예요!

---

### Q3. 그러면 지금은 주문을 할 때마다 생성을 하고 있다는 개념이네요?

**A:** 질문하신 "주문을 할 때마다 생성하고 있다"는 표현은 두 가지 측면에서 볼 수 있는데, 현재 코드 상황에서는 **"주문 한 번을 테스트하기 위해 매번 전체 기계를 새로 조립하고 있다"**는 의미로 이해하시면 정확합니다.

조금 더 구체적으로 나누어 설명해 드릴게요.

**1. 서비스(기계)를 매번 새로 만듭니다.**
현재 `main` 메서드 방식의 가장 큰 특징은, 프로그램을 **한 번 실행해서 주문 하나를 확인하고 나면 프로그램이 종료**된다는 점입니다. 

*   만약 여러분이 주문을 10번 하고 싶다면? 지금 방식으로는 `main` 메서드를 10번 실행해야 합니다.
*   실행할 때마다 `new MemberServiceImpl()`, `new OrderServiceImpl()`이 실행되죠.
*   즉, **"커피 한 잔(주문)을 마시기 위해 매번 카페 건물(서비스)을 새로 짓고 있는 것"**과 같습니다.

**2. 데이터(회원)도 매번 새로 만듭니다.**
DB가 없기 때문에, 주문을 하려면 그 주문을 할 '회원'이 반드시 메모리에 있어야 합니다.
*   그래서 `Member member = new Member(...)` 코드를 통해 매번 가짜 회원을 만들고 `join` 시켜주는 과정이 필요합니다.
*   진짜 서비스라면 DB에 회원이 이미 저장되어 있겠지만, 지금은 **"가게 문 열자마자 테스트용 손님을 한 명 섭외해서 자리에 앉히는 과정"**까지 코드에 포함된 것입니다.

**3. "주문할 때마다 생성"하는 게 왜 문제일까요? (중요!)**
지금은 주문이 하나라 괜찮지만, 만약 서비스가 커져서 `main` 안에서 주문을 수천 번 해야 한다고 가정해 봅시다.

```java
// 이런 식으로 수천 번 반복한다면?
for(int i=0; i<1000; i++) {
    OrderService orderService = new OrderServiceImpl(); // 매번 기계를 새로 만든다? (비효율)
    orderService.createOrder(...);
}
```

나중에 스프링을 도입하면 이 개념이 이렇게 바뀝니다:
*   **스프링 전**: 주문할 때마다(혹은 앱을 켤 때마다) 내가 직접 `new`로 기계를 조립한다.
*   **스프링 후**: 앱을 켤 때 스프링이 기계(`Service`)를 **딱 한 번만** 미리 만들어둔다. 우리는 만들어진 기계를 **빌려와서** `createOrder()`만 호출한다. (이것을 **싱글톤**이라고 부릅니다.)

**요약하자면:**
네, 맞습니다! 지금은 **"주문이라는 행위 한 번을 위해, 서비스 객체 생성부터 회원 데이터 생성까지 모든 과정을 수동으로 매번 반복하고 있는 단계"**입니다. 

이 불편함(매번 직접 `new` 하고, 직접 데이터를 만드는 과정)을 느끼는 것이 이 강의의 의도입니다. 그래야 나중에 스프링이 이걸 얼마나 편하게 자동으로 해주는지 알 수 있거든요!

---

### Q4. `MemberService memberService = new MemberServiceImpl();` 여기에 보면 `new MemberServiceImpl()`을 사용하고 있는데 변수명 전에 `MemberService`라는 것을 쓰고 있습니다. 확인해보니 `MemberService`는 인터페이스인데 이 부분을 왜 인터페이스를 써야 하는지 모르겠습니다. (인터페이스의 역할도 잘 모르겠습니다).

**A:** 인터페이스라는 개념이 처음에는 정말 추상적이고 왜 쓰는지 이해하기 어렵습니다. 하지만 이 개념을 이해하는 순간, **자바 개발자에서 객체 지향 설계자**로 한 단계 업그레이드되시는 거예요!

가장 쉬운 비유와 코드로 설명해 드릴게요.

---

**1. 인터페이스란? (현실 세계의 비유)**

인터페이스는 한마디로 **"역할(Role)"**입니다. 그리고 그 뒤에 붙는 `Impl`(구현체)은 그 역할을 수행하는 **"배우(Actor)"**입니다.

*   **인터페이스 (`MemberService`)**: "회원 가입"과 "회원 조회"라는 기능이 있어야 한다는 **설계도/약속**입니다.
*   **구현체 (`MemberServiceImpl`)**: 그 설계도를 보고 실제로 **코드를 작성해서 움직이게 만든 실제 몸체**입니다.

**비유를 들어볼까요?**
우리가 운전을 할 때, **'자동차'라는 인터페이스(역할)**만 알면 됩니다. 핸들을 돌리면 바퀴가 돌아가고, 브레이크를 밟으면 멈춘다는 '약속'만 알면 되죠. 
그 차가 아반떼든, 테슬라이든, 트럭이든(구현체) 상관없이 운전자는 동일하게 운전할 수 있습니다.

---

**2. 왜 왼쪽(`타입`)에 인터페이스를 쓰나요?**

질문하신 코드:
`MemberService memberService = new MemberServiceImpl();`

만약 인터페이스를 안 쓰고 이렇게 썼다고 가정해 봅시다:
`MemberServiceImpl memberService = new MemberServiceImpl();`

지금 당장은 아무 차이가 없어 보이지만, 나중에 **기능을 바꿔야 할 때** 엄청난 차이가 발생합니다.

#### 상황 예시: "더 빠른 서비스"가 나와서 교체해야 한다면?
어느 날, 아주 성능이 좋은 `FastMemberServiceImpl`이라는 새로운 클래스를 만들었다고 해볼게요.

*   **인터페이스를 썼을 때 (현재 상황):**
    ```java
    // 여기 한 줄만 바꾸면 끝!
    MemberService memberService = new FastMemberServiceImpl(); 
    
    // 이 아래에 memberService를 사용하는 수백 줄의 코드는 수정할 필요가 없습니다.
    memberService.join(member); 
    ```

*   **인터페이스를 안 썼을 때:**
    ```java
    FastMemberServiceImpl memberService = new FastMemberServiceImpl(); 
    
    // 만약 다른 메서드에서 타입을 MemberServiceImpl로 고정해서 쓰고 있었다면?
    // 그 모든 메서드의 파라미터와 변수 타입을 일일이 다 찾아서 고쳐야 합니다. (대공사 발생)
    ```

---

**3. 인터페이스의 결정적인 역할: "유연성"**

인터페이스를 쓰는 이유는 **"사용하는 쪽(클라이언트)"**을 배려하기 위해서입니다.

지금 작성하신 `OrderServiceImpl` 코드를 보면, 내부에서 `MemberService`를 사용하죠? 
`OrderServiceImpl` 입장에서는 상대방이 `MemberServiceImpl`인지, `FastMemberServiceImpl`인지 **알 필요가 없습니다.** 그저 "회원 가입 기능이 있는 `MemberService`구나!"라는 것만 알면 됩니다.

이것을 전문 용어로 **"결합도를 낮춘다"**고 표현하며, 이게 가능해야 나중에 프로그램을 수정할 때 고생을 덜 하게 됩니다.

---

**요약하자면:**

1.  **역할과 구현을 분리**하기 위해서입니다. (배역과 배우를 나누듯이)
2.  **교체가 쉽기 때문**입니다. (아반떼 타다가 테슬라로 바꿔도 운전법은 같으니까요)
3.  **확장에 열려있기 때문**입니다. (새로운 기능을 만들어도 기존 코드를 건드리지 않아도 됩니다)

지금 당장은 "어차피 하나만 만드는데 왜 이렇게 복잡하게 해?"라고 느끼시는 게 당연합니다. 하지만 조금만 더 강의를 들으시다 보면, **"아! 할인 정책을 '정액'에서 '정률'로 바꿀 때, 인터페이스 덕분에 주문 서비스 코드를 하나도 안 고쳐도 되는구나!"** 하는 순간이 올 거예요. 그때 인터페이스의 진가가 발휘됩니다!

---

### Q5. 한마디로 인터페이스는 JavaScript의 TypeScript 같은 역할을 해주는 건가요?

**A:** 네, 정확합니다! **TypeScript에서 `interface`나 `type`을 정의해서 객체의 "형태(Shape)"를 규정하는 것과 개념적으로 매우 비슷합니다.**

**1. "계약(Contract)"으로서의 역할 (TS와 공통점)**
TypeScript에서 `interface User { id: number; name: string; }`라고 정의하면, 그 인터페이스를 사용하는 모든 객체는 반드시 `id`와 `name`을 가져야 하죠?

자바의 인터페이스도 마찬가지입니다.
`MemberService` 인터페이스가 `join()`과 `findMember()`를 정의했다면, 이를 구현하는 `MemberServiceImpl`은 **반드시 이 메서드들을 구현해야만 합니다.** "이 이름을 가진 기능을 꼭 만들어라!"라고 계약을 맺는 것과 같습니다.

**2. "타입 체크" (TS와 공통점)**
TypeScript에서 변수 타입을 인터페이스로 지정하는 것과 같습니다.
```typescript
// TypeScript
const service: MemberService = new MemberServiceImpl();
```
이렇게 하면 `service`라는 변수는 `MemberService`에 정의된 기능만 쓸 수 있게 제한됩니다. 자바에서도 동일하게 인터페이스를 타입으로 써서 **"나는 세부 구현 내용은 몰라도 되고, 인터페이스에 정의된 기능만 쓸 거야"**라고 선언하는 것입니다.

**3. "결정적인 차이점": 다형성과 런타임 (Java의 특징)**
TypeScript의 인터페이스는 컴파일이 끝나고 JavaScript로 변환되면 사라지지만, **자바의 인터페이스는 프로그램이 실행되는 중(Runtime)에도 살아있습니다.**

이게 왜 중요하냐면, 스프링은 이 인터페이스를 보고 **실행 중에** 부품을 갈아 끼우기 때문입니다.

*   **TypeScript**: "코딩할 때 실수하지 않게 타입을 체크해줄게!" (정적 분석 위주)
*   **Java/Spring**: "타입 체크는 기본이고, 나중에 **실행 중에 `FixDiscount`를 `RateDiscount`로 소스 코드 수정 없이 확 바꿔버릴 수 있게** 해줄게!" (유연한 교체 위주)

**요약**
*   **비슷한 점**: 객체가 가져야 할 규칙을 정한다. 타입 안전성을 보장한다.
*   **다른 점**: 자바에서는 인터페이스가 **"부품을 갈아 끼우기 위한 규격(잭/소켓)"**의 의미가 훨씬 강합니다.

그래서 지금 공부하시는 `MemberService memberService = ...` 코드는 **"나는 MemberService라는 규격(인터페이스)에 맞는 부품이면 뭐든지 꽂아서 쓸 준비가 되어 있어!"**라고 선언하는 TypeScript식 선언이라고 이해하시면 아주 정확합니다.

---

### Q6. `MemberService memberService = new MemberServiceImpl();` 여기서 `memberService`는 뭔가요? (객체?) 뭐라고 부르나요?

**A:** 가장 정확한 용어부터 말씀드리면, `memberService`는 **"참조 변수(Reference Variable)"**라고 부릅니다.

조금 더 쉽게 이해하실 수 있도록 이 한 줄을 3개로 쪼개서 명칭을 정리해 드릴게요.

```java
MemberService memberService = new MemberServiceImpl();
// [1. 타입]    [2. 변수명]        [3. 실제 객체(인스턴스)]
```

**1. `memberService` (변수명 / 참조 변수)**
*   **용어**: **변수** 또는 **참조 변수**라고 부릅니다.
*   **역할**: 메모리에 생성된 실제 객체를 가리키는 **"이름표"** 혹은 **"리모컨"**입니다. 
*   우리가 `memberService.join()`이라고 명령을 내리면, 이 이름표를 보고 메모리에 있는 실제 객체를 찾아가서 일을 시키는 것이죠.

**2. `MemberService` (타입 / 인터페이스)**
*   **용어**: **변수의 타입(Type)**입니다.
*   **특징**: 여기서는 인터페이스를 타입으로 썼습니다. 그래서 `memberService`라는 리모컨에는 `MemberService` 인터페이스에 정의된 버튼(메서드)들만 존재하게 됩니다.

**3. `new MemberServiceImpl()` (객체 / 인스턴스)**
*   **용어**: **객체(Object)** 또는 **인스턴스(Instance)**라고 부릅니다.
*   **특징**: `new` 키워드를 통해 메모리(Heap 영역)에 실제로 만들어진 **"실체"**입니다.

---

**이해를 돕기 위한 비유: "리모컨과 TV"**

자바에서 이 한 줄을 이해할 때 가장 좋은 비유는 **리모컨**입니다.

*   **인터페이스(`MemberService`)**: 리모컨의 **버튼 구성** (전원, 채널 변경 등 규칙)
*   **참조 변수(`memberService`)**: 내 손에 들려있는 **리모컨 그 자체**
*   **객체(`new MemberServiceImpl`)**: 거실에 놓여있는 **실제 TV 기계**

우리는 리모컨(`memberService`)을 통해서만 실제 TV(`객체`)를 조작할 수 있습니다. 

**여기서 중요한 포인트!**
만약 리모컨의 버튼 구성(`인터페이스`)이 '범용 리모컨 규격'으로 되어 있다면, 나중에 TV 기계를 삼성에서 LG로 바꿔도(`new LG_TV()`), 우리는 **리모컨의 버튼을 누르는 방식(코드)을 바꿀 필요가 없습니다.**

**요약**
- **`memberService`**는 무엇인가요? -> **참조 변수**입니다. (객체를 가리키는 이름)
- **`new MemberServiceImpl()`**은 무엇인가요? -> **객체(인스턴스)**입니다. (실제로 동작하는 실체)

"멤버 서비스 객체를 `memberService`라는 변수에 담았다"라고 표현하시면 아주 적절합니다!

---

### Q7. `private final MemberRepository memberRepository = new MemoryMemberRepository();` 여기에 `private final`은 무슨 뜻인가요?

**A:** `private`와 `final`은 자바에서 클래스의 내부를 안전하게 보호하고, 실수로 값이 바뀌는 것을 방지하기 위해 사용하는 아주 중요한 키워드들입니다.

각각의 의미를 나누어 설명해 드릴게요.

---

**1. `private` (접근 제어자) - "나만 쓸 거야"**

*   **의미**: 이 변수는 **이 클래스(`MemberServiceImpl`) 내부에서만** 접근할 수 있다는 뜻입니다.
*   **왜 쓰나요?**: 외부(다른 클래스나 `main` 메서드 등)에서 `memberRepository`를 직접 건드리지 못하게 하기 위해서입니다. 
*   **비유**: 식당 주방 안의 요리 도구와 같습니다. 손님(외부 클래스)이 직접 주방 도구를 만지면 위험하겠죠? 주방장(클래스 내부)만 만질 수 있게 벽을 치는 것입니다. 이를 **캡슐화(Encapsulation)**라고 부릅니다.

**2. `final` (예약어) - "절대로 안 바꿀 거야"**

*   **의미**: 한 번 값이 정해지면 **다시는 바꿀 수 없다**는 뜻입니다. (TypeScript의 `const`와 거의 똑같습니다.)
*   **왜 쓰나요?**: `memberService` 기계가 만들어질 때 한 번 저장소를 정해두면, 프로그램이 실행되는 도중에 이 저장소가 갑자기 다른 걸로 바뀌면 안 되기 때문입니다.
*   **만약 `final`이 없다면?**: 실수로 코드 중간에 `memberRepository = null;` 같은 코드를 넣어도 자바가 잡아주지 못합니다. 하지만 `final`이 있으면 자바가 **"어? 이거 안 바꾸기로 약속했잖아!"**라며 에러를 내서 실수를 막아줍니다.

---

**3. 왜 둘을 같이 쓰나요? (`private final`)**

보통 서비스 클래스에서 저장소(Repository)를 선언할 때 이 조합을 가장 많이 씁니다.

```java
private final MemberRepository memberRepository = new MemoryMemberRepository();
```

1.  **안전성**: 외부에서 내 저장소를 훔쳐보거나 수정할 수 없습니다 (`private`).
2.  **불변성**: 한 번 조립된 저장소 부품은 서비스가 끝날 때까지 교체되지 않고 안정적으로 유지됩니다 (`final`).

---

**💡 보너스: 나중에 배울 "생성자 주입" 미리보기**

지금은 코드가 이렇죠?
`private final MemberRepository memberRepository = new MemoryMemberRepository();`

나중에 강의가 진행되면 **"의존관계 주입(DI)"**이라는 것을 배우면서 코드가 이렇게 바뀔 거예요.

```java
public class MemberServiceImpl implements MemberService {
    private final MemberRepository memberRepository; // 일단 선언만!

    // 생성자를 통해 외부에서 넣어준다
    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```

이때도 `final`이 아주 중요한 역할을 합니다. **"생성할 때 딱 한 번만 저장소를 넣어주고, 그 뒤로는 절대 안 바꿀 거야!"**라는 의지를 보여주는 것이죠. 

**요약하자면:**
- **`private`**: "외부에서 보지 마!" (보안/보호)
- **`final`**: "나중에 바꾸지 마!" (안전/상수)

이렇게 이해하시면 완벽합니다! (알겠습니다. 기술적인 관점에서 해당 코드의 의미와 시스템적 영향을 상세히 설명해 드리겠습니다.)

**1. private (Access Modifier)**
기술적으로 private 접근 제어자는 해당 멤버 변수의 가시성(Visibility)을 해당 클래스(MemberServiceImpl) 내부로 제한합니다.
* 캡슐화(Encapsulation): 객체의 내부 상태를 외부(다른 클래스나 패키지)로부터 격리합니다. 외부에서 직접 이 참조 변수에 접근하여 다른 객체로 치환하거나 null을 대입하는 행위를 원천적으로 차단합니다.
* 컴파일 타임 체크: 외부 클래스에서 해당 필드에 접근을 시도할 경우 자바 컴파일러가 에러를 발생시켜 객체의 무결성을 보호합니다.

**2. final (Keyword for Immutability)**
final 키워드는 참조 변수의 재할당(Re-assignment)을 금지합니다.
* 상수화: 변수에 값이 한 번 할당되면(초기화되면), 객체의 생명주기 동안 다른 객체를 가리키도록 변경할 수 없습니다.
* 안전한 초기화(Safe Publication): 멀티스레드 환경에서 final 필드는 생성자가 완료되는 시점에 값이 확정됨을 보장받으므로, 스레드 간 가시성 문제를 해결하는 데 도움을 줍니다.
* 설계적 의도: 해당 서비스 클래스가 특정 저장소(Repository)에 의존하고 있으며, 이 의존관계가 실행 도중 변하지 않아야 함을 명시적으로 선언하는 것입니다.

---

### Q8. 본래 MVC 패턴이 백엔드에 유명한데 Repository가 모델 쪽인가요? (데이터를 접근하기 때문)

**A:** 기술적으로 엄밀히 따지면, **Repository는 MVC 패턴에서 'Model' 영역에 포함되는 구성 요소**가 맞습니다. 하지만 현대적인 백엔드 아키텍처(Layered Architecture)에서는 이를 더 세분화하여 관리합니다.

기술적 관점에서 상세히 분석해 드리겠습니다.

**1. MVC 패턴에서의 Model의 정의**
전통적인 MVC 패턴에서 **Model**은 단순히 데이터만을 의미하는 것이 아니라, **"애플리케이션의 상태와 비즈니스 로직, 데이터 접근 로직을 모두 포함하는 영역"**을 뜻합니다.

*   **View**: 사용자 인터페이스(UI)를 담당.
*   **Controller**: 사용자 요청을 수신하고 Model과 View를 연결.
*   **Model**: 데이터 저장, 조회, 비즈니스 규칙 처리. (여기에 Repository가 포함됨)

**2. 현대적 백엔드 계층 구조 (Layered Architecture)**
현재 공부하고 계신 프로젝트는 MVC 패턴을 더 세분화한 **계층형 아키텍처**를 따르고 있습니다. 이 구조에서 'Model'은 다음과 같이 나뉩니다.

1.  **Domain Entity (Pure Model)**:
    *   `Member`, `Order`, `Grade`와 같이 데이터 자체의 구조와 핵심 비즈니스 규칙을 담고 있는 클래스입니다.
    *   RDBMS의 테이블과 매핑되는 가장 핵심적인 데이터 모델입니다.

2.  **Repository (Data Access Layer)**:
    *   데이터베이스나 메모리 저장소에 접근하여 **Entity를 영속화(Persistence)**하는 역할을 합니다.
    *   기술적으로는 **데이터 액세스 로직(DAL, Data Access Layer)**에 해당합니다.

3.  **Service (Business Logic Layer)**:
    *   여러 Repository를 조합하여 실제 비즈니스 요구사항(예: 주문 생성 시 회원 조회 + 할인 적용 + 주문 저장)을 처리합니다.
    *   도메인 모델들 사이의 워크플로우를 제어합니다.

**3. Repository가 'Model'에 속하는 기술적 이유**
Repository가 Model 영역에 속하는 이유는 **도메인 모델(Entity)의 생명주기를 관리**하기 때문입니다.

*   **추상화**: 서비스 계층이 데이터 저장소의 구체적인 구현(MySQL, Oracle, Memory 등)에 의존하지 않게 합니다.
*   **상태 관리**: 도메인 객체의 상태를 저장소에 반영하거나, 저장소의 데이터를 도메인 객체로 변환(Mapping)하여 상위 계층으로 전달합니다.

**4. Spring MVC에서의 'Model'과 주의할 점**
스프링 프레임워크를 사용하다 보면 `org.springframework.ui.Model`이라는 인터페이스를 접하게 됩니다. 

*   **Spring UI Model**: 이는 컨트롤러에서 뷰(Thymeleaf, JSP 등)로 데이터를 전달하기 위한 **데이터 바구니(Map 형태의 컨테이너)** 역할을 수행하는 객체입니다.
*   **Architectural Model**: 질문하신 "아키텍처 관점의 Model"과는 다른 개념이므로 혼동에 주의해야 합니다.

**요약**
*   **Repository는 MVC의 대분류상 'Model'에 포함됩니다.**
*   기술적으로 더 정확한 표현은 **데이터 액세스 계층(Data Access Layer)**입니다.
*   구조적으로는 **`Controller(Web) -> Service(Business) -> Repository(Data Access) -> Entity(Domain)`**의 흐름 중 하단부에 위치하여 데이터의 영속성을 책임집니다.

---

### Q9. 그러면 궁금한 부분이 Repository 와 Service레이어를 나눈 이유가 무엇인가요?

**A:** 기술적으로 Repository 계층과 Service 계층을 분리하는 가장 핵심적인 이유는 **관심사의 분리(Separation of Concerns, SoC)**를 통해 시스템의 **유지보수성, 확장성, 테스트 가능성**을 높이기 위함입니다. 구체적인 기술적 이유는 다음과 같습니다.

**1. 역할과 책임의 명확한 정의 (Single Responsibility Principle)**
*   **Repository Layer (Data Access)**: 데이터 저장소(DB, Memory, File 등)에 접근하는 **저전력 기술적 로직**에 집중합니다. `CRUD(Create, Read, Update, Delete)`와 같이 데이터를 저장하고 조회하는 "기술적인 데이터 처리"가 주 역할입니다.
*   **Service Layer (Business Logic)**: 애플리케이션의 핵심 비즈니스 요구사항을 처리합니다. 여러 Repository를 호출하여 데이터를 가져온 뒤, 이를 가공하고 비즈니스 규칙(예: 등급별 할인 정책 적용, 포인트 적립 등)을 적용하는 **고전력 도메인 로직**을 담당합니다.

**2. 비즈니스 로직의 원자성 보장 (Transaction Management)**
기술적으로 가장 중요한 이유 중 하나는 **트랜잭션(Transaction) 처리**입니다.
*   하나의 비즈니스 서비스는 보통 여러 번의 데이터베이스 접근을 필요로 합니다. 
*   예를 들어 '주문 생성' 서비스는 `MemberRepository`에서 회원을 조회하고, `OrderRepository`에 주문을 저장하며, `ItemRepository`의 재고를 감소시켜야 합니다.
*   이 모든 과정은 **하나의 단위(Unit of Work)**로 처리되어야 하며, 중간에 오류가 발생하면 전체를 취소(Rollback)해야 합니다. 이 트랜잭션 경계 설정을 Service 레이어에서 수행합니다.

**3. 코드의 재사용성 및 중복 제거**
*   동일한 데이터 조회 로직이 여러 비즈니스 기능에서 필요할 수 있습니다.
*   `findById`와 같은 조회 로직은 Repository에 한 번만 작성해두면, 가입 서비스, 주문 서비스, 마이페이지 조회 서비스 등 다양한 서비스 계층에서 호출하여 재사용할 수 있습니다.
*   만약 서비스와 저장소 로직이 합쳐져 있다면, 기능이 추가될 때마다 유사한 데이터 접근 코드가 중복되어 발생하게 됩니다.

**4. 데이터 저장소 기술의 변경 및 확장성 (Decoupling)**
*   현재 프로젝트처럼 `MemoryMemberRepository`에서 `DbMemberRepository`로 저장소 기술을 변경해야 할 때, 레이어가 분리되어 있으면 **서비스 로직은 전혀 수정할 필요가 없습니다.**
*   서비스 계층은 `MemberRepository` 인터페이스에만 의존하고 있기 때문에, 저장소 계층의 구현체가 바뀌어도 비즈니스 로직의 무결성이 유지됩니다.

**5. 테스트 용이성 (Testability)**
*   **단위 테스트(Unit Test)**를 수행할 때 매우 유리합니다.
*   Service 레이어를 테스트할 때 실제 데이터베이스(Repository)를 연결하지 않고, 가짜 객체(**Mock Object**)를 주입하여 비즈니스 로직만 독립적으로 검증할 수 있습니다. 
*   반대로 Repository를 테스트할 때는 복잡한 비즈니스 로직 없이 쿼리나 데이터 저장 기능만 집중해서 테스트할 수 있습니다.

**요약**
| 계층 | 기술적 집중 분야 | 주요 특징 |
| :--- | :--- | :--- |
| **Service** | 비즈니스 로직, 트랜잭션 관리 | 요구사항 중심, 여러 Repository 조합 |
| **Repository** | 데이터 영속성(Persistence), 쿼리 실행 | 기술 중심, 데이터베이스 직접 접근 |

만약 이 두 계층을 하나로 합친다면, 비즈니스 로직과 SQL/데이터 접근 코드가 뒤섞여 스파게티 코드가 되며, 데이터베이스 기술이 바뀔 때마다 비즈니스 로직 전체를 수정해야 하는 심각한 유지보수 결함이 발생하게 됩니다.

---

### Q10. 
```java
public MemberService memberService() {
    return new MemberServiceImpl(new MemoryMemberRepository());
}

public OrderService orderService() {
    return new OrderServiceImpl(new MemoryMemberRepository(), new FixDiscountPolicy());
}
```
여기 보면 DI 를 한것 같습니다. 여기서 한 단계 더 리팩토링을 할 수 있는 부분은 `new MemoryMemberRepository()` 도 한군데서 실행한 다음, 그 객체를 `MemberServiceImpl`, `OrderServiceImpl`에 넣어놓을수 있는게 더 좋은것 같은데 어떤가요?

**A:** 사용자의 통찰이 매우 정확합니다. 현재 코드의 가장 큰 기술적 결함은 **데이터 정합성(Data Consistency)의 파괴**입니다. 사용자가 제안한 방식은 **중복 제거**뿐만 아니라 **싱글톤(Singleton) 원칙**을 지키기 위한 필수적인 리팩토링 단계입니다.

기술적 관점에서 왜 제안하신 리팩토링이 필요한지 설명해 드리겠습니다.

**1. 현재 코드의 기술적 문제점: 데이터 분리**
현재 코드에서는 `new MemoryMemberRepository()`가 두 번 호출됩니다.
*   `memberService()`가 호출될 때 **저장소 객체 A** 생성
*   `orderService()`가 호출될 때 **저장소 객체 B** 생성

`MemoryMemberRepository` 내부에서 `Map` 자료구조를 사용하여 데이터를 저장한다면, `memberService`를 통해 가입한 회원은 **A**에 저장되지만, `orderService`는 **B**에서 회원을 찾으려 하므로 `null`이 반환되어 로직이 실패하게 됩니다. 즉, 같은 DB를 써야 하는데 서로 다른 DB를 할당받은 꼴이 됩니다.

**2. 제안하신 리팩토링 적용 (AppConfig 개편)**
구체적인 구현 방법은 다음과 같습니다. 각 구성 요소의 생성 로직을 분리하여 **참조값을 공유**하게 만듭니다.

```java
public class AppConfig {

    // 1. 저장소 객체를 생성하는 역할을 별도 메서드로 추출
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }

    // 2. 할인 정책을 생성하는 역할을 별도 메서드로 추출
    public DiscountPolicy discountPolicy() {
        return new FixDiscountPolicy();
    }

    // 3. 서비스들은 추출된 메서드를 호출하여 객체를 주입받음
    public MemberService memberService() {
        return new MemberServiceImpl(memberRepository());
    }

    public OrderService orderService() {
        return new OrderServiceImpl(
            memberRepository(), 
            discountPolicy()
        );
    }
}
```

**3. 리팩토링의 기술적 이점**

1.  **메서드 추출(Extract Method)을 통한 역할 명시**: 
    *   이제 `AppConfig`를 보면 "이 애플리케이션은 `MemoryMemberRepository`와 `FixDiscountPolicy`를 사용한다"는 전체 구성을 한눈에 파악할 수 있습니다.
2.  **중복 제거**: 
    *   저장소나 할인 정책이 변경될 때, 여러 메서드를 수정할 필요 없이 `memberRepository()`나 `discountPolicy()` 메서드의 반환부만 수정하면 됩니다.
3.  **제어의 역전(IoC) 심화**: 
    *   구현체 생성 로직이 완전히 독립되어, 서비스 객체들은 자신이 어떤 저장소를 쓰는지에 대한 지식 없이 오직 주입받는 역할에만 집중하게 됩니다.

**4. 남아있는 기술적 의문: "그래도 new가 두 번 호출되지 않나?"**
순수 자바 코드(Pure Java)로만 작성된 위 `AppConfig`를 실행하면, `memberService()`와 `orderService()`가 각각 `memberRepository()`를 호출하므로 여전히 `new`가 두 번 실행되는 것처럼 보입니다.

이 문제를 기술적으로 해결하는 방식이 바로 **Spring Container**입니다.
*   나중에 `@Configuration` 어노테이션을 붙이면, 스프링은 **CGLIB**라는 바이트코드 조작 라이브러리를 사용하여 `memberRepository()`가 **무조건 딱 한 번만 호출되도록 보장(Singleton)**합니다.
*   이미 생성된 객체가 있다면 새로 생성하지 않고 기존의 참조값을 반환하여, 모든 서비스가 **동일한 인스턴스**를 공유하게 만듭니다.

**요약**
1.  먼저 위와 같이 메서드를 추출하여 **중복을 제거**하고 역할을 명확히 하는 리팩토링을 진행하십시오.
2.  그 이후에 스프링 컨테이너를 도입하여 실제 **런타임 싱글톤**을 보장받는 단계로 나아가는 것이 정석적인 학습 경로입니다.

---

### Q11. `AppConfig` 리팩토링 관련: 서비스를 호출할때 `memberRepository()`를 호출하면 여전히 `new MemoryMemberRepository()`가 각각 `MemberService`, `OrderService()`에 호출이 되서 `MemberService`에서 생성된 고객과 `OrderService`에서 생성된 고객이 동일한 고객이 아닌 다른 고객이 되지 않나요?

**A:** 사용자의 지적이 **기술적으로 100% 정확합니다.** 

순수 자바 코드(Pure Java) 관점에서 보면, `memberService()`를 호출할 때 한 번, `orderService()`를 호출할 때 또 한 번 `memberRepository()` 메서드가 실행되므로, 결과적으로 **`new MemoryMemberRepository()`는 두 번 실행됩니다.** 즉, 서로 다른 인스턴스가 생성되어 데이터 정합성 문제가 여전히 해결되지 않은 상태입니다.

그럼에도 불구하고 강의에서 이 단계로 리팩토링을 하는 기술적 이유는 다음과 같습니다.

**1. 설계적 관점: "중앙 집중화된 구성도" 확보**
이 리팩토링의 1차 목표는 인스턴스의 개수를 조절하는 것이 아니라, **애플리케이션의 전체 구성 정보를 한눈에 파악하기 위함**입니다.
*   **리팩토링 전**: 각 서비스 내부를 일일이 열어봐야 어떤 저장소와 정책을 쓰는지 알 수 있었습니다.
*   **리팩토링 후**: `AppConfig`의 메서드들만 보면 "이 앱은 `MemoryMemberRepository`를 저장소로 쓰고, `FixDiscountPolicy`를 할인 정책으로 쓴다"는 것을 중앙에서 관리할 수 있게 됩니다. 즉, **구현체를 변경할 때 `AppConfig`만 수정하면 되는 구조**를 만든 것입니다.

**2. Spring Framework의 핵심 기능: 싱글톤 보장**
사용자가 제안하신 "진짜로 객체를 딱 하나만 생성해서 공유하는 문제"는 **Spring 컨테이너**가 해결해주는 영역입니다. 이후 강의에서 다음과 같은 변화가 일어납니다.

```java
@Configuration // 이 어노테이션이 핵심입니다.
public class AppConfig {

    @Bean
    public MemberRepository memberRepository() {
        return new MemoryMemberRepository();
    }
    
    // ... 이하 생략
}
```

기술적으로 `@Configuration` 어노테이션이 붙으면 스프링은 다음과 같이 동작합니다.
1.  **CGLIB 바이트코드 조작**: 스프링은 `AppConfig` 클래스를 상속받은 임의의 프록시 클래스를 만들어 실행합니다.
2.  **싱글톤 레지스트리**: `memberRepository()`가 처음 호출될 때 객체를 생성하여 스프링 컨테이너에 등록합니다.
3.  **호출 가로채기**: 이후 `memberService()`나 `orderService()`에서 `memberRepository()`를 다시 호출하면, 스프링은 메서드 내부의 `new`를 실행하는 대신 **이미 컨테이너에 등록된 객체의 참조값을 반환**합니다.

**3. 만약 순수 자바에서 해결하려면?**
스프링 없이 순수 자바에서 사용자가 의도한 대로 동작하게 만들려면 코드를 다음과 같이 구성해야 합니다.

```java
public class AppConfig {
    // 미리 하나를 생성해둔다.
    private final MemberRepository memberRepository = new MemoryMemberRepository();

    public MemberRepository memberRepository() {
        return memberRepository; // 항상 동일한 참조를 반환
    }
    
    // ...
}
```

**요약**
1.  질문하신 대로 현재의 순수 자바 `AppConfig` 코드는 **실제로 `new`가 두 번 호출되어 서로 다른 저장소가 생성되는 것이 맞습니다.**
2.  지금 단계의 리팩토링은 **"구현체의 선택권을 중앙으로 모으는 구조적 설계"**에 집중한 것입니다.
3.  "동일한 객체를 공유(싱글톤)"하는 기술적 문제는 이후 과정인 **스프링 컨테이너(`@Configuration`) 도입**을 통해 자동으로 해결되게 됩니다.

---

### Q12. 스프링 컨테이너(@Configuration, @Bean)를 도입을 했는데 뭔가 느낌은 더 복잡해진것 같은 느낌입니다. `AppConfig`을 가져와서 하는게 더 직관적인것처럼 보이는데 왜 스프링 컨테이너를 통해서 하는게 좋나요? 스프링 컨테이너를 사용하는것에 대한 장점이 무엇인가요?

**A:** 처음 스프링 컨테이너를 접하면, 단순히 자바 코드로도 가능한 일을 왜 복잡한 어노테이션과 `ApplicationContext`라는 생소한 객체를 써서 돌려 하는지 의구심이 드는 것이 당연합니다.

기술적인 관점에서 **"순수 자바 기반 AppConfig"**보다 **"스프링 컨테이너"**가 제공하는 강력한 이점들을 정리해 드리겠습니다.

**1. 싱글톤 컨테이너 (Singleton Registry)**
사용자께서 앞서 가장 크게 걱정하셨던 **"객체가 여러 번 생성되는 문제"**를 스프링 컨테이너가 기술적으로 해결합니다.

*   **기술적 차이**: 순수 자바 `AppConfig`는 메서드를 호출할 때마다 `new`를 실행하여 새로운 인스턴스를 반환할 위험이 큽니다. 반면, 스프링 컨테이너는 `@Bean`으로 등록된 객체를 **단 하나만 생성(Singleton)**하여 관리합니다.
*   **작동 원리**: 스프링은 `@Configuration`이 붙은 클래스를 CGLIB라는 바이트코드 조작 라이브러리를 통해 프록시 객체로 만듭니다. 이 프록시가 "이미 생성된 빈이 컨테이너에 있으면 그것을 반환하고, 없으면 새로 생성하여 등록"하는 로직을 자동으로 수행하여 **객체의 동일성(Identity)**을 보장합니다.

**2. 제어의 역전(IoC)의 완전한 구현**
순수 자바 코드에서는 개발자가 직접 `AppConfig` 객체를 생성하고 메서드를 호출해서 주입을 시켜줘야 합니다. 하지만 스프링 컨테이너를 사용하면:

*   **의존관계 주입(DI) 자동화**: 나중에 배우게 될 `@Autowired` 기능을 사용하면, 개발자가 `AppConfig`를 뒤져서 어떤 객체를 넣어줄지 고민하지 않아도 스프링이 컨테이너 내에서 적절한 타입을 찾아 **자동으로 연결**해줍니다.
*   **객체 관리의 주체**: 객체의 생성부터 소멸까지의 **생명주기(Lifecycle)**를 개발자가 아닌 프레임워크가 관리하게 됩니다.

**3. 부가 기능의 동적 적용 (AOP)**
실무에서 스프링을 쓰는 가장 결정적인 이유 중 하나입니다.

*   **프록시 기술**: 스프링 컨테이너는 객체를 빈으로 등록할 때, 그 객체를 감싸는 **프록시(Proxy)**를 만들 수 있습니다. 
*   이를 통해 실제 비즈니스 로직(Service) 코드를 전혀 수정하지 않고도, **트랜잭션 처리(`@Transactional`)**, **보안 검사**, **로그 기록** 등의 공통 기능을 실행 중에 동적으로 삽입할 수 있습니다. 순수 자바 객체(POJO)로는 구현하기 매우 까다로운 영역입니다.

**4. 확장성과 유연성**
*   **환경별 설정 관리**: 개발 환경(dev), 테스트 환경(test), 운영 환경(prod)에 따라 서로 다른 빈을 등록해야 할 때, 스프링 컨테이너의 `Profile` 기능을 이용하면 코드 수정 없이 설정만으로 부품을 갈아 끼울 수 있습니다.
*   **수많은 편의 기능**: 컨테이너는 단순히 객체를 담는 통을 넘어, 메시지 소스(국제화 처리), 이벤트 발행/구독, 리소스 로딩 등 대규모 애플리케이션 개발에 필요한 수많은 인프라적 기능을 기본적으로 제공합니다.

**5. 대규모 시스템에서의 복잡도 관리**
지금은 클래스가 5~10개 내외라 `AppConfig`가 직관적으로 보이지만, 실무에서는 수백, 수천 개의 클래스가 얽히게 됩니다. 

*   이때 모든 객체의 관계를 `AppConfig`에 수동으로 적는 것은 불가능에 가깝습니다. 스프링 컨테이너는 **컴포넌트 스캔(Component Scan)** 기능을 통해 특정 패키지 아래의 클래스들을 자동으로 찾아 빈으로 등록해주므로, 설정 정보 자체가 비대해지는 것을 방지합니다.

**요약**
| 구분 | 순수 자바 AppConfig | 스프링 컨테이너 |
| :--- | :--- | :--- |
| **객체 관리** | 호출 시마다 생성될 위험(비효율) | 싱글톤 보장(메모리 효율) |
| **의존성 연결** | 수동 호출 및 주입 | 자동 주입(@Autowired) 가능 |
| **추가 기능** | 직접 코드로 구현해야 함 | AOP를 통한 선언적 처리 가능 |
| **관리 규모** | 소규모에 적합 | 대규모 엔터프라이즈 시스템에 필수 |

---

### Q13.
1. 싱글톤 컨테이너에서 설명한거를 보면 약간 캐시의 계념을 사용하는 것 같은 느낌이 드는데 맞나요? `new`를 통해서 새로운 인스턴스를 생성하고, 동일한 인스턴스를 또 생성할 경우 기존에 만들었던 인스턴스를 반환하는 방법?
2. 그러면 나중에 프로젝트가 커지면 여러개의 커스텀 `@Configuration` 같은 것을 생성해서 추가로 더 사용할 수 있고, 예를 들어 100개의 서비스 코드에 사용했다고 하면 `@Configuration`이 선언된 한곳에서 관리할 수 있으니까 관리 시점에서 용의하다, 이렇게 보는 건가요?

**A:** 질문하신 두 가지 포인트 모두 핵심을 정확히 꿰뚫고 계십니다. 기술적인 관점에서 더 깊게 답변해 드리겠습니다.

**1. 싱글톤 컨테이너와 캐시(Cache)의 개념**
**네, 맞습니다. 기술적으로 스프링 컨테이너는 일종의 "객체 캐시 서버" 역할을 수행한다고 이해하셔도 무방합니다.**

*   **Singleton Registry**: 스프링 컨테이너 내부에는 생성된 빈(Bean)들을 저장하는 **Map 형태의 저장소**가 존재합니다. (Key: 빈 이름, Value: 객체 인스턴스)
*   **동작 방식**: 사용자가 `applicationContext.getBean()`을 호출하거나 다른 빈에서 의존성 주입을 요청할 때, 스프링은 매번 `new`를 실행하는 것이 아니라 이 내부 저장소(캐시)를 먼저 확인합니다.
*   **CGLIB의 마법**: 앞서 언급한 `@Configuration` 클래스는 자바의 일반적인 `new` 호출을 가로챕니다. 
    ```java
    // 스프링이 조작한 가짜 AppConfig의 로직 (의사 코드)
    if (beanRegistry.contains("memberRepository")) {
        return beanRegistry.get("memberRepository"); // 이미 있으면 캐시에서 반환
    } else {
        return super.memberRepository(); // 없으면 실제 new를 실행하고 캐시에 등록
    }
    ```

**2. 대규모 프로젝트에서의 관리 용이성**
**네, 관리 포인트가 중앙 집중화된다는 점이 매우 큰 장점입니다. 하지만 프로젝트가 정말 커지면 여기서 한 단계 더 진화한 방식을 사용합니다.**

**① 설정의 분리 및 모듈화**
말씀하신 대로 기능별로 `OrderConfig`, `MemberConfig`, `DbConfig` 등 여러 개의 `@Configuration` 클래스를 만들어 관리할 수 있습니다. 100개의 서비스가 있더라도 이 몇 개의 설정 파일만 보면 "우리 시스템이 어떻게 조립되어 있는지" 한눈에 파악할 수 있는 **지도(Map)** 역할을 하게 됩니다.

**② 컴포넌트 스캔(Component Scan)의 도입 (핵심)**
그런데 서비스가 100개, 1,000개가 넘어가면 아무리 설정 파일을 잘 나눠도 직접 `@Bean`을 하나하나 적는 것 자체가 노동이 됩니다. 이때 스프링의 진짜 위력이 나옵니다.
*   **자동 등록**: 각 클래스 위에 `@Component`만 붙여주면, 스프링이 패키지를 훑어서(Scan) 자동으로 컨테이너에 등록해줍니다.
*   **자동 연결**: `@Autowired`를 쓰면 `AppConfig`에서 누가 누구를 참조할지 직접 코드로 적어줄 필요조차 없어집니다.

---

### Q14. `@Component`를 다는 것은 `return new MemoryMemberRepository();`를 자동으로 해준다, 그 의미인가요? `@Bean`을 사용한 코드를 보면 `return new ...` 이 있어서 이해가 가는데 `@Component`를 하면 `return` 이 없는데 어떻게 사용이 되는건가요?

**A:** 네, 질문하신 내용이 정확합니다. 기술적인 관점에서 **`@Component`는 개발자가 직접 작성하던 `new` 과정을 스프링이 대신 수행하도록 위임하는 선언**입니다.

어떻게 `return` 문 없이도 객체가 생성되고 관리되는지 그 내부 동작 원리를 설명해 드리겠습니다.

**1. `@Bean` vs `@Component` 인스턴스화의 차이**
*   **`@Bean` (수동 방식)**: 개발자가 직접 자바 코드로 인스턴스화 로직을 작성합니다. (**개발자가 `new` 호출**)
*   **`@Component` (자동 방식)**: 스프링이 클래스의 정보를 읽어 직접 인스턴스화합니다. (**스프링이 `new` 호출**)

**2. `return` 문 없이 어떻게 사용되나요? (기술적 원리)**
`@Component`가 붙은 클래스는 스프링이 **리플렉션(Reflection API)**이라는 기술을 사용하여 객체를 생성합니다.
1.  **스캐닝(Scanning)**: 스프링 컨테이너가 시작될 때 `@ComponentScan` 범위에 있는 클래스들을 전부 뒤집니다.
2.  **클래스 분석**: `@Component` 어노테이션이 붙은 클래스를 발견하면, 해당 클래스의 **메타데이터(생성자, 필드, 메서드 정보 등)**를 읽어들입니다.
3.  **리플렉션을 통한 인스턴스화**: 스프링은 내부적으로 다음과 유사한 자바 코드를 실행하여 객체를 생성합니다.
    ```java
    Class<?> clazz = MemoryMemberRepository.class;
    Object instance = clazz.getDeclaredConstructor().newInstance(); 
    ```
4.  **컨테이너 등록**: 생성된 인스턴스를 스프링 컨테이너(Bean Registry)에 담아둡니다. 이때 `return` 문은 필요 없습니다.

---

### Q15.
```java
@Component
public class MemberServiceImpl implements MemberService {
    private final MemberRepository memberRepository;

    @Autowired // "이 생성자에 필요한 타입의 객체를 네 컨테이너 안에서 찾아서 넣어줘!"
    public MemberServiceImpl(MemberRepository memberRepository) {
        this.memberRepository = memberRepository;
    }
}
```
여기서도 `@Component`를 사용하고 있는데 그 뜻은 `MemberServiceImpl`의 `new` 값도 스프링 컨테이너가 관리하도록 하겠다 라는 의미롤 보는건가요?

**A:** 네, 정확합니다. 기술적으로 **`@Component`를 클래스 레벨에 선언하는 것은 해당 클래스의 인스턴스화(Instantiation), 즉 `new` 과정을 포함한 생명주기 전체를 스프링 컨테이너가 제어하겠다는 선언**입니다.

구체적인 기술적 의미는 다음과 같습니다.

**1. 빈(Bean) 등록 및 관리**
`MemberServiceImpl`에 `@Component`를 붙이면, 스프링 컨테이너는 직접 `new MemberServiceImpl(...)`을 실행하여 메모리에 올립니다. 이렇게 생성된 객체를 **스프링 빈(Spring Bean)**이라고 부르며, 컨테이너는 이 객체를 소멸할 때까지 관리합니다.

**2. 제어의 역전(IoC, Inversion of Control)**
객체의 생성 권한을 프레임워크(스프링)에 넘긴 것입니다.

**3. 재귀적 의존관계 해결 (Recursive Dependency Resolution)**
기술적으로 가장 중요한 부분은 **의존관계의 연쇄적 해결**입니다.
1. 스프링이 `MemberServiceImpl`을 생성하려고 시도합니다.
2. 생성자에 `@Autowired`가 있으므로 `MemberRepository`가 필요함을 인지합니다.
3. 컨테이너 내에서 해당 타입으로 등록된 다른 빈을 찾습니다.
4. 만약 해당 빈이 아직 생성되지 않았다면, **먼저 해당 저장소를 `new` 하여 생성**한 뒤, 그 참조값을 주입합니다.

---

### Q16. 즉 어떻게 보면 프론트엔드 리엑트의 Context인데, 캐시로 작동하는 컨텍스트 같은 느낌이 드네요.

**A:** 그 비유는 현대적인 개발 관점에서 **매우 날카롭고 정확한 통찰**입니다.

React의 **Context API**와 Spring의 **IoC 컨테이너**는 해결하고자 하는 근본적인 문제와 동작 방식에서 매우 유사한 평행이론을 가지고 있습니다. 기술적인 관점에서 그 유사성을 3가지 포인트로 짚어드리겠습니다.

**1. "Prop Drilling" vs "Manual DI" 해결**
"누가 누구를 거쳐서 전달하는가"가 아니라, **"중앙 저장소(Context/Container)에 올려두고 필요한 놈이 직접 꺼내 쓴다"**는 전략입니다.

**2. Provider(공급자)와 Bean(객체)의 캐싱**
React Context가 "상태의 단일 소스"라면, 스프링 컨테이너는 **"객체(인스턴스)의 단일 소스"** 역할을 합니다.

**3. useContext() vs @Autowired**
둘 다 **"내가 직접 객체를 생성하지 않고, 외부 저장소에서 나에게 맞는 것을 찾아달라고 요청"**하는 방식입니다.

---

**기술적인 결정적 차이 (차이점도 알아두시면 좋습니다)**

1. **데이터(Data) vs 로직(Behavior)**: React Context는 주로 상태를, Spring Container는 주로 행위(기능)를 관리합니다.
2. **재렌더링(Re-render) vs 생명주기(Lifecycle)**: React Context는 데이터가 바뀌면 재렌더링되지만, Spring Bean은 한 번 생성되면 애플리케이션 종료 시까지 그대로 머뭅니다.

**요약**
사용자의 말씀대로 스프링 컨테이너는 **"애플리케이션 전체에서 공유되는 로직 객체들을 캐싱해두고, 필요한 곳에 자동으로 꽂아주는 전역 컨텍스트(Global Context)"**라고 이해하시면 기술적으로 가장 완벽한 이해입니다.
