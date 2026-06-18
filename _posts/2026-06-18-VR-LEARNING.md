---
title: vr 멀티 게임 제작 2일차
date: 2026-06-18 20:00:00 +0900
categories: [다이어리,개발]
tags: [jekyll, 블로그]
---

## 학습내용 :
https://learn.unity.com/course/create-with-vr/unit/vr-basics/tutorial/1-2-vr-locomotion?version=6.2
https://learn.unity.com/course/create-with-vr/unit/vr-basics/tutorial/1-3-grabbable-gameobjects?version=6.2

플레이의 주체가 되는 XR Origin (샘플 내 오브젝트 이름) 구조

⚙️ XR Origin (VR) (최상위 VR 플레이어 리그)
     ├── 📁 Camera Offset (HMD 및 컨트롤러 위치 보정용 가이드)
     │   ├── 🎥 Main Camera (헤드 트래킹 및 시야 담당)
     │   ├── 🎮 Left Hand (왼손 인터랙터 / Force Grab 옵션 제어)
     │   └── 🎮 Right Hand (오른손 인터랙터 / Force Grab 옵션 제어)
     └── 🏃 Locomotion (이동 및 회전 기능 통합 오브젝트)  (캐릭터의 이동 및 제어 부분 기능을 이곳에 넣어두고 활성, 비활성화로 관리 )
         ├── 🛠️ Locomotion System (최상위 이동 중재 컨트롤 타워)
         ├── 🛠️ Teleportation Provider (순간이동 처리 담당)
         ├── 🛠️ Snap Turn Provider (시점 끊어 돌리기 - 멀미 방지용, 기본 활성화)
         └── 🛠️ Continuous Move Provider (조이스틱 지속 이동 - 필요 시 토글)


1. XR Device Simulator 조작법 (헤드셋이 없을시 사용)
하드웨어 연결 없이 키보드와 마우스로 VR 타겟 시스템을 테스트하는 기능임.

🎮 제어 대상 선택 (Hold 또는 Toggle)
Space 바 (Hold): 오른손 컨트롤러(Right Hand) 제어 권한 획득.

Left Shift (Hold): 왼손 컨트롤러(Left Hand) 제어 권한 획득.

마우스 우클릭 (Hold): 헤드셋(HMD/시야) 제어 모드로 전환.

T / Y 키: 각각 왼손 / 오른손 컨트롤러 선택 상태 고정(Toggle). 해제는 Esc.

🏃 위치 및 회전 조작
W, A, S, D: 제어 대상의 전/후/좌/우 수평 이동.

Q, E: 제어 대상의 수직 하강 / 상승.

마우스 휠 스크롤: 선택된 손(컨트롤러)의 앞/뒤 거리(Depth) 조절. 물건 조준 및 파지에 필수적임.

마우스 이동 (우클릭 또는 손 선택 상태): 대상의 상/하/좌/우 회전(Rotation) 제어.

🕹️ 컨트롤러 버튼 매핑 (손 선택 상태에서 입력)
마우스 좌클릭: 트리거(Trigger) 버튼 클릭 (UI 선택, 발사 등).

G 키: 그립(Grip) 버튼 클릭 (물건 쥐기 및 잡기).

숫자 1 / 숫자 2: Primary Button(A/X) / Secondary Button(B/Y) 입력.

M 키: 메뉴(Menu) 버튼 입력.

키보드 방향키 (↑, ↓, ←, →): 조이스틱(Joystick) 방향 패드 조작 (텔레포트 조준 및 회전 연동).

2. VR 로코모션(Locomotion) 시스템 (Unit 1.2)
Locomotion System: 이동/회전 컴포넌트 간 신호 충돌을 방지하고 중재하는 최상위 컨트롤 타워 컴포넌트임.

Teleportation Area: 자유롭게 이동 가능한 넓은 지형/바닥 면적을 지정함. 레이저가 닿은 정확한 지점으로 순간이동함.

Teleportation Anchor: 특정 타겟 지점으로 착지 위치를 유도함. 좌표뿐만 아니라 이동 후 바라볼 시선 방향(Rotation)까지 강제 고정함.

Snap Turn: 조이스틱 입력 시 시야를 일정 각도(예: 45도)씩 순간적으로 끊어서 회전시킴. VR 멀미 방지에 가장 효과적이며 기본 세팅으로 권장됨.

Continuous Move / Turn: 일반 게임처럼 부드럽게 걷고 회전하는 방식임. 몰입감은 높으나 뇌의 인지부조화로 인해 심한 VR 멀미를 유발할 수 있어 옵션 선택형으로 제공함.

조준점(Reticle) 노출 원리: 인풋 키 입력 조건과 레이저(Ray) 발사가 충족된 상태에서, 충돌한 오브젝트가 Teleportation 특성(Area 또는 Anchor)을 가지고 있을 때만 최종적으로 조준점(Reticle Prefab)이 표면에 시각화됨.

하이어라키 구조화 장점: Locomotion 빈 오브젝트 하나만 SetActive(false) 처리하면 순간이동, 걷기, 회전 기능이 한 방에 차단되어 UI 오픈이나 컷신 재생 시 유지보수가 극도로 용이함.

3. 오브젝트 상호작용 및 그랩(Grab) 시스템 (Unit 1.3)
XR Grab Interactable: 아이템에 부착하여 쥐거나 던질 수 있는 속성을 부여함. 부착 시 물리 연산을 위한 Rigidbody가 자동 생성됨.

Force Grab: 멀리 떨어진 물체를 레이저로 조준하고 그립을 누르면 손안으로 즉시 끌어당겨 잡히는 원격 그랩 기능임. (양손 XR Ray Interactor에서 옵션 제어 가능)

Attach Transform (그립 위치 보정): 무기, 도구 등 파지법 구분이 필요한 물체의 어색한 겹침 현상을 해결함. 물체 하위에 빈 게임 오브젝트를 만들어 손이 위치할 기준점(오프셋)을 설정한 뒤 슬롯에 등록함.

Jitter(떨림) 방지: 물체를 들고 움직일 때 발생하는 미세한 물리적 떨림을 제어하기 위해 컴포넌트 내 Smooth Position / Rotation 속성을 활성화함.

벽 관통 버그 방지: 고속으로 던져진 물체가 충돌 처리를 무시하고 벽을 뚫는 현상을 막기 위해, Rigidbody의 Collision Detection 설정을 Continuous Dynamic으로 변경함.

4. 프로그래밍 디자인 패턴 응용 (물체 특성 부여)
마커 인터페이스(Marker Interface) 패턴: 내용물이 비어있는 인터페이스(예: IFlammable)를 MonoBehaviour를 상속받은 클래스에 다중 구현하는 방식임.

장점: 레이저나 물리 충돌 감지 시 TryGetComponent<IFlammable>을 통해 물체의 구체적인 정체(나무 상자, 기름통 등)를 몰라도 해당 특성의 유무를 논리적으로 즉시 판별하여 상호작용 처리가 가능함. 씬 컴포넌트 구조화와 결합 시 코드 결합도를 크게 낮춤. 


## 트러블 슈팅:
* github.io 의 포스트가 안올라가서 문제 분석 (md 파일의 날짜형식과 내용의 날짜형식이 불일치하여 오류 발생) 해결완료

## 이어서 진행할 것 : 
* VR 키트 제공 기능 학습 
* 넷코드 학습 진행
* 개발 역할 분담 토의
