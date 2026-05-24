---
title: Jekyll 블로그 첫 시작과 환경 구성
date: 2026-05-23 22:00:00 +0900
categories: [다이어리, 시작]
tags: [jekyll, 블로그]
---


## 개발 일지: MQ 대시보드 리팩토링 및 디버깅 기록



---

### **1. 초기 상황 및 문제 진단**

*   **관찰**: Spring Boot 백엔드 애플리케이션 시작 시 MySQL 데이터베이스 연결 실패(`Communications link failure`).
*   **내 분석**: `application.yml`에 설정된 MySQL 서버에 연결이 거부됨. 사용자로부터 HeidiSQL 파싱 기능이 더 이상 사용되지 않는다는 정보를 얻어, 현재 대시보드 기능에 SQL 데이터베이스가 필수적이지 않을 수 있다고 판단.
*   **결정**: 핵심 MQ 대시보드 기능 구현을 위해 MySQL 연결을 일시적으로 비활성화하기로 함.

---

### **2. 백엔드 설정 조정 (MySQL 연결)**

*   **수정 내용**: `backend/Kgy_Springboot/src/main/resources/application.yml` 파일에서 `spring.datasource` 및 `spring.jpa` 섹션을 주석 처리.
*   **수정 이유**: 백엔드 애플리케이션이 MySQL 연결을 시도하지 않도록 하여 시작 실패 오류를 해결하기 위함.
*   **트러블슈팅**:
    *   **문제**: 초기 주석 처리 후 `ParserException` (YAML 구문 오류) 발생.
    *   **내 분석**: 최상위 `spring:` 블록 전체를 주석 처리하면서 YAML 구조가 깨짐.
    *   **해결 방법**: `spring:` 블록 자체는 유지하고, 그 하위의 `datasource`와 `jpa` 섹션만 주석 처리하여 올바른 YAML 구조를 유지.

---

### **3. 백엔드 리팩토링: 메시지 전송을 위한 동적 MQ 타입 전환**

*   **목표**: 프론트엔드에서 MQ 타입을 선택하면 백엔드의 메시지 전송 MQ(Redis, RabbitMQ, NATS)가 동적으로 전환되도록 구현.
*   **내 접근 방식**: 정적인 `application.yml` 설정 대신, 동적 서비스와 Spring Cloud의 `@RefreshScope`를 활용.
*   **스크립트 생성/수정 및 이유**:
    *   **`backend/Kgy_Springboot/src/main/java/kgy/mq/ActiveMqTypeService.java` 생성**:
        *   **생성 이유**: 현재 활성화된 MQ 타입을 중앙에서 관리하기 위함. 이 서비스는 `@Component`이자 `@RefreshScope`로, 상태 변경 시 의존하는 빈들의 재초기화를 트리거할 수 있음.
        *   **내용**: `activeMqType` 필드를 포함하며, `application.yml`의 `mq.type`으로 초기화되고 getter/setter를 가짐.
    *   **`backend/Kgy_Springboot/src/main/java/kgy/mq/MasterConfig.java` 수정**:
        *   **수정 이유**: `currentMqProducer` 빈이 `Environment` 속성 대신 `ActiveMqTypeService`를 통해 동적으로 활성 MQ 타입을 가져오도록 하기 위함.
        *   **변경 내용**: `ActiveMqTypeService`를 주입. `currentMqProducer` 메서드에서 `mqType`을 `activeMqTypeService.getActiveMqType()`에서 가져오도록 변경. `showCurrentMqInfo` 로그도 서비스에서 값을 가져오도록 수정.
    *   **`backend/Kgy_Springboot/src/main/java/kgy/mq/MqController.java` 수정**:
        *   **수정 이유**: 프론트엔드에서 활성 MQ 타입을 변경하도록 요청할 수 있는 API 엔드포인트를 제공하기 위함.
        *   **변경 내용**: `ActiveMqTypeService`와 `ContextRefresher`를 주입. `POST /api/mq/setActiveType` 엔드포인트 추가. 이 엔드포인트는 `activeMqTypeService.setActiveMqType()`을 호출하여 타입을 업데이트하고, `contextRefresher.refresh()`를 호출하여 `@RefreshScope` 빈들의 재초기화를 트리거함.

---

### **4. 프론트엔드 디버깅: 흰 화면 및 `TypeError`**

*   **관찰**: 백엔드 변경 후 프론트엔드 대시보드가 흰 화면으로 표시됨.
*   **내 분석**: 이는 일반적으로 JavaScript 오류 또는 API 호출 실패를 나타냄.
*   **트러블슈팅 - 1단계 (`ReferenceError: selectedMqType is not defined`)**:
    *   **문제**: 브라우저 콘솔에서 `App.jsx`의 `selectedMqType`에 대한 `ReferenceError` 보고.
    *   **내 분석**: 코드상으로는 `useState`로 명확히 선언되어 있었으므로, 브라우저 또는 개발 서버의 캐싱 문제로 판단.
    *   **해결 시도**: 강력한 캐시 클리어링 ( `node_modules`, `dist`, `.vite` 폴더 삭제 후 `npm install`, `npm run dev` 재시작, 브라우저 강제 새로고침) 수행. 이 과정에서 `console.log` 위치로 인한 `Cannot access 'selectedMqType' before initialization` 오류도 발생했으나, `console.log` 제거 및 `useEffect` 의존성 배열 복원으로 해결.
    *   **결과**: `ReferenceError`는 해결되었으나, 여전히 흰 화면.

*   **트러블슈팅 - 2단계 (`TypeError: Cannot read properties of undefined (reading 'toString')`)**:
    *   **문제**: `fmt` 함수에서 `undefined` 값에 대해 `toString()`을 호출하려 할 때 `TypeError` 발생. `MqCard` 컴포넌트에서 `data.transactionCount`가 `undefined`였기 때문.
    *   **내 분석**: `App.jsx`의 `dummyData`에는 `transactionCount: 0`이 존재했으므로, 초기 렌더링 시에는 문제가 없어야 함. 오류는 `fetchMqData`가 상태를 업데이트하려 할 때 발생하고 있었음.
    *   **근본 원인 파악**: `fetchMqData`가 호출하는 백엔드 API(`GET /api/mq/metrics/{type}`)의 응답이 JSON 데이터가 아닌 프론트엔드의 `index.html` 콘텐츠였음. 이는 `vite.config.js`에 `/api/mq` 경로에 대한 프록시 설정이 누락되었기 때문.
    *   **해결 방법**: `frontend/dashboard-react/vite.config.js`에 `/api/mq` 경로를 `http://localhost:8080` (백엔드 주소)로 프록시하는 규칙을 추가.
    *   **결과**: 프론트엔드 화면이 정상적으로 표시되기 시작.

---

### **5. 현재 상태 및 다음 할 일**

*   **현재 상태**: 프론트엔드 대시보드 화면은 정상적으로 표시되지만, `TypeError: Cannot read properties of undefined (reading 'toString')` 오류가 여전히 발생하고 있음. 이는 백엔드 API(`GET /api/mq/metrics/{type}`)의 응답에서 `transactionCount` 값이 `undefined` 또는 `null`로 오고 있음을 시사함.
*   **내 분석**: `App.jsx`의 `fetchMqData` 함수에 `console.log`를 추가하여 백엔드 API로부터 수신되는 `res` (전체 Axios 응답) 및 `res.data` (파싱된 JSON)의 정확한 내용을 확인하는 중. 이 로그를 통해 백엔드가 예상하는 `MqMetricsResponse` 형식으로 데이터를 반환하고 있는지 검증할 예정.

---

### **6. 향후 계획**

1.  **백엔드 API 응답 분석**: `fetchMqData`의 `console.log` 출력을 면밀히 검토하여 `data.transactionCount`가 `undefined`인 정확한 원인 파악.
2.  **백엔드 API 검증**: 응답 내용을 바탕으로 백엔드의 `MqController.getMqMetrics` 메서드가 `meterRegistry`로부터 `transactionCount`를 올바르게 가져와 반환하고 있는지 디버깅.
3.  **프론트엔드 동적 전송 구현**: 메트릭 표시가 안정화되면, MQ 타입 버튼 클릭 시 `POST /api/mq/setActiveType`을 호출하는 프론트엔드 로직 구현.
4.  **프론트엔드 메시지 전송 기능 구현**: 메시지 전송을 위한 UI 요소를 구현하고, 백엔드의 현재 활성 `mqProducer`를 사용하도록 연동.
