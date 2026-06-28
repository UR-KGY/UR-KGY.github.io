---
title: Steamworks 로비 생성 학습중
date: 2026-06-28 21:00:00 +0900
categories: [개발, VR]
tags: [unity, steamworks]
---

netcode 와 스팀 로비 생성을 연동을 위해 먼저 steamwork 부분을 학습중
라이브러리의 내용을 전부 이해하는것은 시간적으로 손해가 심하므로 다른 이들의 
사용방법을 보면서 대략적인 기능과 사용방법을 터득하는 것을 목표로함 

https://www.youtube.com/watch?v=oLfQa2Ypeq0&list=PLueG8VSxPj8NY9b3kuuW_ASEKGbeYzwXy&index=2


# 📂 Steamworks 비동기 프로그래밍 및 디버깅 노트

## 1. OnLobbyEntered NullReferenceException 원인
* **현상:** `obj.GetComponentInChildren<Text>().text = SteamClient.Name;` 줄에서 널 에러 발생.
* **오해:** `SteamClient.Name`이 null일 것이라 추측함.
* **진짜 원인:** 생성된 프리팹 내에서 레거시 `Text` 컴포넌트를 찾지 못해 `GetComponentInChildren<Text>()`가 `null`을 반환함. 
* **해결 방법:** * 최신 유니티 버전이라면 `TextMeshProUGUI` 자료형으로 변경.
  * 프리팹 자식이 비활성화 상태라면 `GetComponentInChildren<Text>(true)`로 활성화 여부와 상관없이 찾도록 수정.
  * 프리팹 최상위에 스크립트를 달아 인스펙터에서 직접 UI를 드래그 앤 드롭하는 방식(추천)으로 변경.

---

## 2. 친구 초대(Invite) 미작동 문제
* **현상:** 로그에 상대방 `SteamID`는 정상 출력되나 초대가 가지 않음.
* **원인 1 (실수):** 로비 생성(`CreateLobbyAsync`)을 하지 않은 상태에서 `CurrentLobby.InviteFriend()`를 호출함.
* **원인 2 (환경적 요인):** 스팀 테스트용 기본 AppID인 **480 (Spacewar)**을 사용하는 경우, 스팀 자체 보안 정책으로 인해 친구 초대 메시지 전송이 제한될 수 있음.
* **교훈:** `catch (System.Exception e)` 블록을 비워두면 에러가 침묵하므로, 반드시 `Debug.LogError`를 작성하여 예외를 추적해야 함.

---

## 3. 비동기(Async / Await)와 Task의 개념

### 💡 비동기를 쓰는 이유
* 스팀 서버 이미지 다운로드 등 시간이 걸리는 작업 시 게임이 멈추는 현상(프레임 드랍, 응답 없음)을 방지하기 위함.

### 📦 Task의 용도
* **개념:** 미래에 완료될 비동기 작업의 상태를 담은 **'대기 티켓(영수증)'**.
* **역할:** `Task`가 있어야만 다른 함수에서 `await` 키워드를 사용해 "이 작업이 끝날 때까지 기다렸다가 다음 코드를 실행해라"라는 연계 제어가 가능해짐.
* **`Task.Delay(n)`:** 메인 컴퓨터를 얼리지 않고, 유니티의 정상 루프를 돌리면서 의도적으로 n초 동안 대기 상태를 시뮬레이션할 때 사용함.

---

## 4. async void vs async Task (올바른 사용법)

C# 비동기 함수 설계 시 아래의 공식(표준)을 반드시 따름.

| 반환 타입 | 특징 | 핵심 용도 |
| :--- | :--- | :--- |
| **`async Task`** | 완료 여부 추적 및 대기(`await`) 가능. 에러 발생 시 유니티가 안전하게 캐치함. | **표준.** 직접 만들고 직접 호출하는 모든 비동기 함수 (95%) |
| **`async void`** | 호출 후 추적 불가능(Fire-and-Forget). 내부 에러 발생 시 게임 크래시 위험 존재. | **예외.** 유니티 버튼(OnClick)이나 스팀 이벤트 콜백 시스템에 함수를 '등록'할 때만 사용 (5%) |

### 🛠️ 코드 적용 예시
```csharp
// 1. 다른 곳에서 호출하고 기다려야 하므로 Task 사용 (데이터 반환 시 Task<T>)
public async Task<bool> CreateLobby() 
{
    var result = await SteamMatchmaking.CreateLobbyAsync();
    return result.HasValue;
}

// 2. 유니티/스팀 시스템 규격(void)에 맞추어 등록하는 문고리 함수이므로 void 사용
private async void OnLobbyEntered(Lobby lobby) 
{
    // 내부에서 비동기 메서드 호출 가능
    await DoSomethingAsync(); 
}