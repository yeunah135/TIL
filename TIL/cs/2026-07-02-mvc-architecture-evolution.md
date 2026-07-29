# MVC부터 헥사고날/클린 아키텍처까지

## 배운 이유

과거에 학습했던 MVC는 Model, View, Controller라는 세 가지 역할 구분만 존재하는 개념적인 모델이었습니다. 하지만 실무 예제(ARChat 프로젝트)를 보면서 MVC2, 헥사고날, 클린 아키텍처 등 새로운 개념들이 등장해 혼란스러웠습니다. 오늘은 이 개념들이 서로 어떤 관계인지, 어떻게 진화해왔는지 정리했습니다.

핵심은 다음과 같습니다.

- 고전 MVC는 **"누가 어떤 책임을 지는가"**에 대한 답입니다.
- MVC2는 **"요청이 코드상 어떤 경로로 흐르는가"**를 구체화한 답입니다.
- 헥사고날/클린 아키텍처는 **"Model이 외부 기술에 오염되지 않도록 의존성을 어떻게 통제할 것인가"**에 대한 답입니다.

즉, 셋은 서로 경쟁 관계가 아니라 점진적으로 더 세분화된 관계라는 것을 알게 되었습니다.

---

## 1. 고전 MVC (개념 단계)

Model(데이터/로직), View(화면), Controller(중재자)라는 역할 구분만 존재합니다. 다만 이 구분은 "요청이 어떤 경로로 흐르는지"까지는 정의하지 않습니다. 그래서 실제로 구현하면 JSP 하나에 로직과 화면이 모두 섞이는 MVC1 형태가 되기 쉬웠다는 것을 알게 되었습니다.

---

## 2. MVC2 (서블릿 기반)

MVC2는 "요청 진입점을 누가 맡는가"를 명확히 한 구조입니다. 서블릿이 Controller 역할을 전담하고, 처리 결과를 Request Scope에 담아 JSP로 forward하는 방식입니다.

ARChat 프로젝트의 `ChatController`를 통해 이 흐름을 확인할 수 있었습니다.

```java
@Override
protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    Chat chat = new Chat(
            req.getParameter("message"),
            "USER",
            req.getSession().getId(),
            req.getParameter("model"),
            ZonedDateTime.now().toString()
    );
    chatService.save(chat);
    resp.sendRedirect("%s/%s".formatted(req.getContextPath(), "chat"));
}
```

여기서 `chatService.save(chat)`이 Model 역할(`GeminiChatService`)을 호출하고, 결과를 담아 `chat.jsp`(View)로 넘기는 구조입니다. 개념적 MVC의 Controller/Model/View가 각각 **서블릿 / 서비스+리포지토리 / JSP**라는 구체적인 클래스로 실체화된 것을 확인했습니다.

---

## 3. 헥사고날 / 클린 아키텍처

여기서부터는 MVC의 확장이 아니라, **"Model이 외부 기술(API, DB 등)에 직접 의존하지 않게 막는 방법"**에 대한 답이라는 점이 새롭게 이해된 부분입니다.

ARChat의 `AIChatService`는 AI를 직접 호출하지 않고, `ChatProvider`라는 **포트(인터페이스)**만 알고 있습니다. 실제 Gemini 호출은 `GenAIChatProvider`, Groq 호출은 `GroqChatProvider`가 담당하는데, 이것이 바로 **어댑터**입니다.

```java
if (chat.model().contains("gemini") || chat.model().contains("gemma")) {
    aiResponse = genAIChatProvider.useAI(chat, history);
} else {
    aiResponse = groqChatProvider.useAI(chat, history);
}
```

`AIChatService` 입장에서는 "AI를 하나 호출했다"는 사실만 중요하고, 그것이 Gemini API인지 Groq API인지는 `ChatProvider` 포트 뒤에 숨겨져 있습니다. 나중에 다른 AI API를 추가하더라도 `AIChatService`의 핵심 로직은 수정하지 않아도 되는 구조라는 것을 알게 되었습니다.

이 구조는 정석 클린 아키텍처가 아니라 **약식(타협형)** 구조였습니다. `ChatProvider`처럼 아웃고잉 포트는 유지하되, 인커밍 포트는 생략해서 `ChatController`가 서비스를 직접 호출하는 방식입니다. 보일러플레이트를 줄이면서도 핵심 가치(외부 기술 교체로부터 도메인 로직을 보호하는 것)는 지키는 실무적 타협이라는 점이 인상 깊었습니다.

---

## 이번 수업 학습 목표

PDF 자료의 학습 체크리스트와 오늘 정리한 내용을 바탕으로 이번 수업에서 배워야 할 점을 정리했습니다.

**1. 아키텍처 패턴을 왜 쓰는지 이해하기**

모듈성(Modularity)과 관심사 분리(SoC)가 왜 중요한지, 코드 변경이 시스템 전체로 퍼지는 것을 막기 위한 장치라는 것을 이해해야 합니다.

**2. MVC 1 → MVC 2 → Spring MVC의 구조적 차이 파악하기**

- MVC 1: JSP가 로직과 화면을 동시에 처리해 결합도가 높습니다.
- MVC 2: 서블릿이 Controller를 전담하고, JSP는 View만 담당합니다 (ARChat의 `ChatController` 사례).
- Spring MVC: `DispatcherServlet`이라는 프론트 컨트롤러가 요청을 중앙에서 일괄 처리합니다.

**3. 레이어드 아키텍처의 의존성 규칙 숙달하기**

상위 계층에서 하위 계층으로만 흐르는 단방향 의존성을 이해하고, 서비스가 구체 클래스가 아닌 `Repository` 인터페이스에 의존해야 하는 이유(DIP)를 SRP, OCP와 연결지어 파악해야 합니다.

**4. 클린/헥사고날 아키텍처로 비즈니스 로직 격리하기**

포트(인터페이스)와 어댑터(구현체)로 코어를 보호하는 구조를 이해해야 합니다. ARChat의 `ChatProvider` 포트와 `GenAIChatProvider`/`GroqChatProvider` 어댑터가 실제 사례이며, 이 구조 덕분에 테스트가 쉬워지고 외부 기술 교체에도 도메인 로직이 흔들리지 않는다는 것을 알아야 합니다.

**5. 프로젝트 규모에 맞는 실무적 타협점 찾기**

정석 클린 아키텍처는 보일러플레이트가 많아 생산성이 떨어질 수 있으므로, 실무에서는 "MVC+레이어드" 또는 "약식 클린 아키텍처"(아웃고잉 포트만 유지, 인커밍 포트 생략)처럼 절충안을 쓴다는 것을 이해해야 합니다. ARChat도 이 약식 클린 아키텍처 방식을 채택하고 있습니다.

한 줄로 요약하면, 완벽한 아키텍처보다 프로젝트 규모에 맞게 격리와 생산성 사이에서 어디까지 타협할지 판단하는 안목을 기르는 것이 이번 수업의 핵심 목표라고 할 수 있습니다.

---

## 알게 된 점

- MVC는 "역할 분리"의 문제를 풀고, 헥사고날/클린 아키텍처는 "의존성 방향"의 문제를 푸는 것이었습니다. 둘은 별개의 개념이 아니라, MVC2의 Model 자리를 Domain + Application + Port 구조로 더 세분화한 결과라는 것을 이해했습니다.
- 정석 클린 아키텍처를 그대로 적용하면 포트와 DTO가 과도하게 늘어나 생산성이 떨어질 수 있으며, 실무에서는 아웃고잉 포트만 유지하는 약식 구조가 자주 쓰인다는 것을 알게 되었습니다.
- 포트(인터페이스)와 어댑터(구현체) 구조 덕분에, 새로운 AI 모델을 추가할 때 핵심 서비스 로직(`AIChatService`)을 수정하지 않고 어댑터만 추가하면 된다는 확장성의 이점을 코드로 직접 확인했습니다.
