---
title: spring mvc
date: 2026-08-21 20:00:00 +0900
categories: [개발]
tags: [java,springboot,mvc]
---

# MVC 패턴과 Spring MVC 정리

이해 순서대로 정리한다: 서블릿 → 톰캣 → MVC 패턴 → Spring MVC 흐름 → REST API에서 달라지는 점 → Controller 어노테이션.

## 1. 서블릿(Servlet)이란

자바로 작성된, HTTP 요청을 처리하는 코드 단위다. "이런 요청이 오면 이렇게 처리한다"는 로직을 담은 클래스다.

## 2. 톰캣(Tomcat) = 서블릿 컨테이너

톰캣은 서블릿이 아니라, 서블릿을 담고 실행·관리해주는 그릇(컨테이너)이다.

- 요청이 들어오면 어떤 서블릿이 처리할지 찾아 연결한다.
- 서블릿 객체의 생명주기(생성 → 요청 처리 → 소멸)를 관리한다.
- 처리 결과를 HTTP 응답 형태로 클라이언트에 돌려준다.

즉 "톰캣(그릇) 안에 서블릿(코드)이 올라가서 돌아간다"는 관계다.

## 3. MVC 패턴이란

애플리케이션을 세 가지 역할로 나누는 설계 패턴이다. 목적은 관심사 분리(화면과 로직을 서로 독립적으로 수정 가능하게 함)다.

- **Model**: 데이터와 비즈니스 로직 담당.
- **View**: 화면(UI) 렌더링만 담당. **클라이언트 요청을 직접 받거나 응답을 보내는 주체가 아니다** — Model 데이터를 받아 눈에 보이는 결과물(HTML 등)로 만드는 역할일 뿐이다.
- **Controller**: 클라이언트의 요청을 실제로 받는 주체. 로직 실행을 지시하고, 어떤 View를 보여줄지 결정한다.

## 4. Spring MVC의 요청 처리 흐름

Spring MVC는 이 MVC 개념에 **DispatcherServlet**(프론트 컨트롤러)을 추가해 구현한 것이다. DispatcherServlet 자체도 서블릿의 한 종류이며, 톰캣이 이걸 실행시켜 모든 요청의 첫 진입점으로 삼는다.

1. 클라이언트가 HTTP 요청을 보낸다.
2. 톰캣이 실행 중인 **DispatcherServlet**이 요청을 가장 먼저 받는다.
3. DispatcherServlet이 알맞은 **Controller**를 찾아 요청을 넘긴다.
4. Controller가 필요하면 Service/Model을 호출해 로직을 수행한다.
5. Controller는 데이터와 함께, 보여줄 View의 이름을 반환한다.
6. DispatcherServlet이 그 이름에 맞는 **View**를 찾아 렌더링을 요청한다.
7. View가 최종 화면(HTML)을 만들고, 그 결과가 응답으로 클라이언트에 나간다.

## 5. REST API에서 달라지는 점

`@RestController`를 쓰면 위 흐름 중 6~7번(View 렌더링) 단계가 사라진다. Controller가 반환한 객체(예: `User`)가 Jackson 같은 라이브러리에 의해 자동으로 JSON 변환(직렬화)되어 그대로 응답 본문에 담긴다.

정확히 말하면 "Controller가 View 역할을 겸하는 것"이 아니라, **View라는 별도 계층 자체가 사라지고 그 자리를 자동 직렬화 기능이 대체하는 것**이다.

## 6. Controller 관련 주요 어노테이션

- **@Controller**: 클래스를 Controller로 지정한다. 기본적으로 메서드의 반환값을 View 이름으로 해석한다(Thymeleaf 등으로 화면 렌더링).
- **@RequestMapping**: 요청 경로를 매핑하는 원조 어노테이션. `method` 속성으로 HTTP 메서드를 직접 지정할 수 있다. 메서드 레벨보다는 **클래스 레벨에서 공통 경로(base path)를 지정**하는 용도로 자주 쓰인다. 메서드 레벨엔 아래의 축약형을 쓰는 게 관례다.
- **@GetMapping("/경로")**: 해당 URL의 HTTP GET 요청을 이 메서드가 처리하도록 매핑한다. `@RequestMapping(method = GET, value = "/경로")`의 축약형. 같은 방식으로 `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping`이 있으며 각 HTTP 메서드에 대응한다.
- **@ResponseBody**: `@Controller`의 기본 동작(반환값 → View 이름)을 바꿔, 반환값을 View 없이 그대로 응답 본문(JSON 등)으로 직렬화하게 한다. REST API의 "View 계층 생략" 이 어노테이션으로 구현된다.
- **@RestController**: `@Controller` + `@ResponseBody`를 합친 것. 클래스의 모든 메서드에 자동으로 `@ResponseBody`가 적용된다. REST API 전용 클래스에서는 이걸 표준으로 쓴다.
- **@PathVariable**: URL 경로 안의 값을 변수로 추출한다. (`/users/{id}` → `id`)
- **@RequestParam**: URL 쿼리스트링 값을 추출한다. (`?name=hong` → `name`)
- **@RequestBody**: 요청 본문(JSON)을 자바 객체로 변환해 받는다. `@ResponseBody`(응답을 내보냄)와 반대 방향(요청을 받음)이다.

```java
@RestController
@RequestMapping("/api/users")   // 클래스 전체의 공통 경로
public class UserController {

    @GetMapping                 // 실제 경로: GET /api/users
    public List<User> getUsers() {
        return userService.findAll();
    }

    @GetMapping("/{id}")        // 실제 경로: GET /api/users/{id}
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // @ResponseBody가 자동 적용되어 JSON으로 응답
    }

    @PostMapping                // 실제 경로: POST /api/users
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }
}
```

화면(HTML)까지 렌더링할 페이지는 `@Controller`, 데이터(JSON)만 주고받는 API는 `@RestController`로 만든다. 클래스에 `@RequestMapping`으로 공통 경로를 걸어두면 메서드마다 전체 경로를 반복해서 적지 않아도 된다.

### 매핑이 실제로 갈라지는 지점

같은 URL이라도 HTTP 메서드가 다르면 다른 메서드가 실행되거나, 매칭되는 핸들러가 없으면 **405 Method Not Allowed**가 응답된다. 예를 들어 `/api/users`에 `@GetMapping`만 있는 상태에서 DELETE 요청을 보내면 데이터 대신 405 에러가 온다. Postman으로 같은 URL에 메서드만 바꿔가며 보내보면 이 차이를 직접 확인할 수 있다.

## 한 줄 정리

- 톰캣은 서블릿을 담아 실행하는 컨테이너이고, MVC는 Model·View·Controller로 역할을 나누는 설계 패턴이다. Spring MVC는 여기에 DispatcherServlet(프론트 컨트롤러)을 더해 요청 흐름을 관리하며, REST API에서는 View 계층이 사라지고 자동 직렬화가 그 자리를 대신한다.
- `@Controller`+`@ResponseBody` = `@RestController`. `@RequestMapping`(클래스 공통 경로)과 `@GetMapping` 등(메서드별 HTTP 매핑)으로 경로를 구성하고, `@PathVariable`(경로 값)·`@RequestParam`(쿼리 값)·`@RequestBody`(요청 본문)로 데이터를 받는다.
- 같은 URL이라도 매핑된 HTTP 메서드가 아니면 405 에러가 발생한다. 매핑 어노테이션은 단순 관례가 아니라 실제 요청을 걸러내는 역할을 한다.
