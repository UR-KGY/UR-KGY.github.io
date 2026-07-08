---
title: 로비 UI 정리 및 오류 수정
date: 2026-07-07 21:00:00 +0900
categories: [개발]
tags: [unity,steamworks]
---

## 1. 주요 기능
* **Steamworks 로비 연동**: Facepunch API를 활용하여 방 개설 및 타인 로비 진입 처리 완료.
* **유니티 6 Build Profiles 사용**: 독립된 빌드 설정을 생성하여 대상 플랫폼(`.exe`)으로 결과물 추출 완료.
* **실시간 준비 상태(Ready) 동기화**: 스팀 서버의 `MemberData`를 통해 로비 내부 인원의 준비 여부를 전송 및 실시간 UI 반영 완료.
* **방장 전수 조사 및 시작 권한**: 로비 내부 멤버들의 준비 상태를 실시간 체크하여 모두 완료 시 방장의 시작(Start) 버튼 활성화 완료.

---

## 2. 핵심 원리

### 💡 유니티 빌드 시스템과 에디터 코드 분리
* 유니티는 실제 게임 빌드 시 `UnityEditor` 네임스페이스에 포함된 모든 도구 코드를 강제로 제외함. 
* 에디터 전용 기능이 일반 스크립트에 섞여 있으면 컴파일 에러가 발생하므로, `using` 구문 제거 또는 전처리문(`#if UNITY_EDITOR`)을 통한 코드 격리가 필수적임.

### 🔄 Netcode의 비동기 세션 종료 (`Shutdown`)
* `NetworkManager.Singleton.Shutdown()`은 소켓을 닫고 연결을 정리하는 데 수 프레임 이상 소요되는 **비동기성 작업**임.
* 종료 명령 직후 대기 없이 새 네트워크 세션(`StartClient` 등)을 시작하면, 엔진 내부적으로 기존 리스닝 상태가 유지되어 충돌이 발생함. 따라서 완전히 꺼질 때까지의 프레임 대기 처리가 필요함.

### 📊 LINQ(Language Integrated Query)의 지연 연산
* C# 컴파일러가 LINQ 구문을 내부적으로 반복문(`foreach`)과 조건문 형태로 자동 변환함.
* 선언 시점에는 연산이 실행되지 않다가, `.ToList()`, `.FirstOrDefault()`, 또는 `foreach` 순회 등 실제 알맹이가 필요한 시점에 비로소 루프를 도는 **지연 연산(Deferred Execution)** 원리로 작동함.

---

## 3. 트러블 슈팅 (Troubleshooting)

### ❌ Netcode 중복 구동 경고
* **현상**: `[Netcode] Can't start while listening` 경고 발생.
* **원인**: 방장이 로비 입장 콜백을 동시에 타면서 이미 호스트가 켜진 상태인데 중복으로 `StartClient()`를 요청했거나, `Shutdown()`이 완전히 끝나기 전 다음 줄 코드가 실행됨.
* **해결**: `JoinToServer` 메서드를 `async Task` 기반의 비동기 흐름으로 전환. `Shutdown()` 호출 후 `IsListening`이 `false`가 될 때까지 `await Task.Yield()`로 대기 구조를 적용하여 해결 완료.

### ❌ 타인 화면에 준비(Ready) 표시 갱신 누락
* **현상**: 로비 멤버가 준비를 눌러도 다른 사람 화면의 체크 UI가 바뀌지 않고 `False`로 고정됨.
* **원인**: 스팀의 `OnLobbyMemberDataChanged` 콜백 핸들러 내부에서 스팀 서버가 보내준 실제 데이터(`memberReady`) 대신, 내 로컬 변수(`isReady`)를 매개변수로 잘못 전달함.
* **해결**: `OnMemberReadyChanged?.Invoke(friend, memberReady)`로 매개변수 오타를 수정하여 정상 동기화 완료.

### ❌ 인원 변동 시 기존 유저 Ready 상태 초기화
* **현상**: 새로운 유저가 들어오거나 나갈 때 기존 유저들의 Ready 마크가 해제됨.
* **원인**: 인원 변동 시 로비 UI 슬롯을 통째로 파괴(`Destroy`)하고 새로 생성(`Instantiate`)하면서 프리팹 컴포넌트에 저장되어 있던 로컬 `IsReady` 변수가 유실됨.
* **해결**: 슬롯을 새로 생성하는 시점에 스팀 서버(`lobby.GetMemberData`)로부터 각 유저의 최신 "Ready" 값을 다시 읽어와 재설정하도록 UI 배치 로직 보완 완료.

### ❌ 방장의 Start 버튼 활성화 불가
* **현상**: 모든 인원이 준비를 마쳤음에도 방장의 시작 버튼이 활성화되지 않음.
* **원인**: 방장은 화면에 `Ready` 버튼이 없어 슬롯의 `IsReady`가 항상 `false`임. 이 상태로 전수 조사(`CheckAllMembersReady`) 리스트에 방장 슬롯까지 포함되면서 조건 충족이 불가능했음.
* **해결**: 전수 조사 루프 중 내 스팀 ID(`SteamClient.SteamId`)와 일치하는 **방장 슬롯인 경우 검사를 건너뛰도록(`continue`) 예외 처리**하여 해결 완료.

## TODO : 
* 씬 전환 및 게임 시작 로직 작성 후에 네트워크 프리팹을 수동으로 생성 시도 동기화 테스트  가능하다면 vr 조작 테스트까지 시도