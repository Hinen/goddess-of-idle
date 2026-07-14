# GODDESS OF IDLE Spine 애니메이션 및 Spring Rig 기술 명세

> 문서 상태: 구현 기준안 1.0<br>
> 작성 기준일: 2026-07-14<br>
> 대상: Unity 6.3 (`6000.3.19f1`), spine-unity 4.3 계열<br>
> 목적: 원본 영상이나 NIKKE 추출 에셋을 보지 않은 작업자가 오리지널 캐릭터용 Spine 리그, 애니메이션, Unity 제어, 2차 물리를 구현할 수 있게 한다.

---

## 0. 이 문서의 결론

이 프로젝트가 구현할 것은 **오리지널 Spine 4.3 캐릭터 리그 + NIKKE 발표에서 설명한 구조를 참고한 커스텀 Unity Spring Rig**다.

핵심 결정은 다음과 같다.

1. 원화, 파츠, 본 이름, 리그, 애니메이션 데이터는 모두 새로 만든다. NIKKE 추출 파일을 제작 기준이나 납품물로 사용하지 않는다.
2. 기본 포즈와 전투 동작은 Spine에서 제작한다.
3. 머리카락, 의상, 가슴, 스트랩 같은 2차 동작은 Unity에서 **가상 시뮬레이션 본**을 계산하고, 그 결과를 렌더용 Spine 본에 되쓴다.
4. 주 솔버는 고정 시간 간격의 RK4이며 C# Job System과 Burst 적용을 전제로 설계한다.
5. Spine 4.3 내장 Physics constraint는 이 커스텀 솔버와 다른 시스템이다. **같은 본 또는 같은 변환 속성을 두 솔버가 동시에 소유하면 안 된다.**
6. 입력 한 번마다 사격 애니메이션을 처음부터 재생하지 않는다. 공격 큐와 무기 cadence가 실제 탄 발사를 결정하며, 애니메이션은 지속 루프와 이벤트, 절차적 반동 impulse로 연결한다.
7. 캐릭터는 `SkeletonAnimation + SkeletonRenderer`를 사용한다. `SkeletonGraphic`은 월드 캐릭터가 아니라 UI 안에서 렌더해야 할 때만 사용한다.
8. custom-only subtree의 물리 적용 순서는 `애니메이션 로컬 포즈 → 첫 world transform → 가상 본 시뮬레이션 → Spine 로컬 변환 기록 → 두 번째 world transform → LateUpdate 렌더`로 고정한다. built-in Physics와 혼용하면 첫/둘째 pass가 `Physics.Pose → callback → Physics.Update`가 되므로 §15.2의 경계를 따른다.

이 문서에서 “NIKKE식”이라는 표현은 시각적 방향과 공개 발표에서 설명한 기술 원리를 뜻한다. NIKKE의 실제 내부 본 이름, 전체 리그, 수치, 소스 코드가 공개됐다는 뜻이 아니다.

---

## 1. 근거 수준과 용어

### 1.1 근거 수준

| 표기 | 의미 |
| --- | --- |
| **확인** | 로컬 프로젝트, 공식 문서, 발표자 정정문, 공식 발표 영상에서 직접 확인 |
| **공개 관찰** | 공개 영상 또는 제3자 뷰어에서 보이지만 내부 구현을 확정할 수 없음 |
| **본 프로젝트 결정** | 공개 자료를 토대로 이 프로젝트에 맞게 새로 정한 구현 규칙 |
| **튜닝 시작값** | 프로파일링·아트 검수 전에 사용하는 초기값이며 공식 NIKKE 값이 아님 |
| **미확인** | 공개 근거가 없어 추정하지 않기로 한 사항 |

### 1.2 용어

| 용어 | 이 문서에서의 뜻 |
| --- | --- |
| 렌더 본 | Spine skeleton 안에 있으며 attachment를 변형하거나 slot을 움직이는 실제 본 |
| 드라이버 본 | 애니메이터가 키를 넣는 Spine 본. 접두사 `drv_` |
| 물리 출력 본 | 커스텀 솔버가 최종 로컬 변환을 기록하는 Spine 본. 접두사 `phys_` |
| 가상 시뮬레이션 본 | Unity `NativeArray`에만 존재하는 질점/방향 데이터. Spine 본 트리에 별도 뼈로 만들 필요 없음 |
| 앵커 | 애니메이션을 따라가며 물리 체인의 시작점을 제공하는 비시뮬레이션 본 |
| Spring Rig | 앵커, 가상 체인, 보조 스프링, 제한, 충돌체, 렌더 본 매핑을 합친 커스텀 물리 리그 |
| Spine Physics | Spine 4.2부터 추가된 공식 Physics constraint 기능 |
| cadence | 무기의 실제 발사 간격과 탄 소비 규칙 |

---

## 2. 현재 프로젝트 기준선과 즉시 수정해야 할 전제

### 2.1 로컬에서 확인한 상태

- 게임은 하단 고정형 투명 데스크톱 오버레이이며 3명의 캐릭터가 상주한다. 역할은 지원형 B1, 마우스 B2, 키보드 B3이다. 자세한 게임 규칙은 [GAME_DESIGN.md](./GAME_DESIGN.md)를 따른다.
- 최소 상태는 `idle`, `cover`, `fire`, `reload`, `burst`, `hit`, `defeat`다.
- 게임 기획은 입력마다 fire 애니메이션을 재시작하지 말라고 명시한다.
- Unity 버전은 [ProjectVersion.txt](./ProjectSettings/ProjectVersion.txt)의 `6000.3.19f1`이다.
- [manifest.json](./Packages/manifest.json)은 Spine 패키지 3개를 `#4.3` 브랜치로 지정한다.
- [packages-lock.json](./Packages/packages-lock.json)의 잠긴 커밋은 `8a238d76a808b325ba41824b438048677c414e14`다. 해당 커밋의 패키지 메타데이터 기준 버전은 spine-unity `4.3.96`, spine-csharp `4.3.39`, URP shaders `4.3.25`다.
- `packages-lock.json`에 보이는 `4.3.36`은 spine-unity가 요구하는 최소 spine-csharp 의존성 문자열이지, 설치된 spine-unity 버전이 아니다.
- 현재 저장소에는 실제 캐릭터 Spine 에셋, 캐릭터 제어 C# 코드, Spring Rig 코드가 없다.

### 2.2 overlay와 관련된 별도 결손

Spine 구현만으로 투명 데스크톱 오버레이가 완성되지 않는다. 현재 설정에는 다음 충돌이 있다.

- `ProjectSettings.asset`: `runInBackground: 0`
- `ProjectSettings.asset`: `preserveFramebufferAlpha: 0`
- URP asset: post-process alpha output 비활성
- topmost, click-through, 전역 입력, 다중 모니터, DPI 보정용 네이티브 창 계층이 없음

Spring Rig 작업자는 이 결손을 물리 코드 안에서 우회하지 않는다. 별도의 `DesktopOverlayWindow` 계층이 창 alpha, 포커스, click-through, work area, DPI를 책임져야 한다.

### 2.3 버전 고정 규칙

- 이 명세의 weighted mesh, IK, advanced skin/constraint authoring은 **Spine Professional**을 전제로 한다. Spine Essential에는 weighted mesh와 IK, Physics constraint가 없다. 설치 여부와 별개로 edition 및 유효 라이선스를 시작 전에 확인한다.
- 제작에 사용하는 Spine Editor의 **major.minor는 반드시 4.3**이어야 한다.
- 런타임의 patch 버전은 잠긴 Git commit으로 재현한다.
- `#4.3` 부동 브랜치를 release 파이프라인에서 그대로 resolve하지 않는다. manifest와 lockfile을 함께 버전 관리한다.
- 4.2 또는 4.4 Editor에서 export한 데이터와 4.3 Runtime을 혼용하지 않는다.
- major.minor 업그레이드 때는 모든 `.spine` 원본을 새 Editor로 열고 전체 재export, 회귀 테스트를 수행한다.

Spine은 Editor와 Runtime의 major.minor 일치를 요구한다. 현재 공식 호환표에서 spine-unity 4.3은 Unity 6000.4까지를 포함하므로 Unity 6000.3은 범위 안이지만, 잠긴 commit 조합은 실제 Player build로 별도 검증한다. 자세한 원칙은 [Spine Versioning Guide](https://esotericsoftware.com/spine-versioning)와 [spine-unity Installation/Compatibility](https://esotericsoftware.com/spine-unity-installation)를 따른다.

---

## 3. 공개 NIKKE 자료에서 확인된 것과 확인되지 않은 것

### 3.1 확인된 사실

1. NDC22 발표 보도에 따르면 개발팀은 초기 Live2D 방향에서 기술 R&D 후 Spine으로 전환했다. 본 구조로 신체 부위를 추적하고 상호작용하며 3D 총기를 부착하기 쉬운 점이 슈터 연출에 적합하다고 설명했다. 출처: [Inven NDC22 발표 정리](https://www.inven.co.kr/webzine/news/?news=272724), [NDC22 발표 영상](https://www.youtube.com/watch?v=Bn_x75Wm9p8).
2. Unity Wave 2022 발표는 Spine 애니메이션 위에 커스텀 헤어·신체 물리를 합성하는 구조를 설명한다. 출처: [Unity Korea 발표 영상](https://www.youtube.com/watch?v=-zyp6qNTms0).
3. 발표자는 실제 Spine 본과 1:1로 대응하는 가상 스프링 본을 따로 두고, 애니메이션과 물리가 같은 본을 동시에 제어하지 않게 했다고 설명한다.
4. 체인의 **개념적 의존성**은 부모→자식이며, 상단의 비물리 본이 움직이는 앵커가 된다. 다만 발표에서 설명한 실제 Job 구현은 한 번의 physics update에서 물리 본들을 동시에 계산했다. 이 때문에 생기는 계층 오차는 계산을 충분히 자주 수행하면 줄어든다고 설명한다.
5. 스프링 복원력, 감쇠력, 보조 스프링, 길이 고정, 움직임 제한, 애니메이션-물리 혼합 기능이 설명된다.
6. 발표 뒤 게시된 발표자 정정문에 따르면 발표 슬라이드의 적분기 설명은 잘못되었다. 발표 당시 설명 대상인 기존 구현은 Euler-Cromer였고, 발표 뒤 NIKKE 구현을 RK4로 교체했다. C# Job System과 Burst를 사용했으며 발표자의 해당 프로젝트 프로파일링에서는 두 방식의 비용 차이가 거의 없었다고 한다. 측정 기기, 빌드, 본 수, 표본 수치가 공개되지 않았으므로 이를 일반적인 RK4 성능 우위나 무비용 주장으로 확대하지 않는다. 출처: [발표자 정정문](https://ijemin.com/blog/%EC%9C%A0%EB%8B%88%ED%8B%B0-%EC%9B%A8%EC%9D%B4%EB%B8%8C-%EC%8A%B9%EB%A6%AC%EC%9D%98-%EC%97%AC%EC%8B%A0-%EB%8B%88%EC%BC%80-%EC%8A%A4%ED%8C%8C%EC%9D%B8-%EB%AC%BC%EB%A6%AC-%EA%B5%AC%ED%98%84-%EB%B0%9C/).
7. Spine 자체 Physics constraint는 4.2 beta changelog에 2023-11-11 추가됐고 4.2 stable은 2024-04-16, 공개 출시 공지는 2024-04-27이다. 따라서 2022년 발표 구현은 **공개 배포된 Spine Physics constraint를 사용한 것이 아니다**. 이 연대기는 공개되지 않은 내부 prototype의 존재 여부까지 증명하지는 않는다. 출처: [Spine changelog archive](https://esotericsoftware.com/spine-changelog/archive), [Spine 4.2 발표](https://esotericsoftware.com/blog/Spine-4.2-The-physics-revolution), [4.2 출시 공지](https://esotericsoftware.com/forum/d/26069-spine-42-has-been-released).

### 3.2 공개 관찰이지만 공식 내부 사양으로 취급하지 않을 것

- 공개 제3자 뷰어에는 캐릭터별 `stand`, `aim`, `cover`, `skillcut` 번들과 `idle` 계열 애니메이션이 보인다.
- 이 뷰어는 일반적인 spine-webgl 로더이며 NIKKE Spring Rig 소스가 아니다.
- 추출 에셋을 살펴보면 렌더 구조에 관한 힌트는 얻을 수 있지만, 제작 시 사용된 원본 `.spine`, 명명 규칙, 물리 authoring 데이터, Unity 매핑 코드는 알 수 없다.
- 서로 mirror 관계인 저장소는 독립된 복수 근거로 계산하지 않는다.

### 3.3 미확인 사항

- NIKKE의 실제 전체 본 트리와 본별 수치
- 캐릭터별 정확한 stiffness, damping, mass, collision radius
- 현재 서비스 버전이 2022년과 동일한 solver를 유지하는지 여부
- 내부 authoring tool과 serialization format
- job batch/dependency 구성, fixed timestep, substep 제한의 실제 값
- 공개 발표 구현에 별도 collision solver가 있었는지 여부
- 애니메이션 track 배치와 이벤트 이름

따라서 아래 본 트리와 수치는 **본 프로젝트를 위한 오리지널 제안**이며 NIKKE 내부 구조의 복제가 아니다.

---

## 4. 원본 영상 없이 이해하는 목표 동작과 알고리즘

### 4.1 영상의 핵심 타임라인

| 구간 | 구현자가 알아야 할 내용 |
| --- | --- |
| 01:30–02:04 | Spine 키 애니메이션과 편집 가능한 2D 물리를 결합해 머리카락·신체 흔들림을 자동 생성 |
| 03:01–04:06 | 변위 반대 방향의 Hooke 복원력 `-k(x-rest)` |
| 05:40–07:25 | 부모가 애니메이션으로 움직이면 자식 물리 본은 world space에서 뒤늦게 따라온다. 체인의 개념적 의존성은 부모→자식 |
| 07:27–08:28 | 체인 최상단의 비물리 애니메이션 본이 앵커 역할 |
| 08:34–09:16 | 애니메이터가 물리 본을 표시하고 연속 구간을 체인으로 묶는 authoring 흐름 |
| 09:25–10:59 | 렌더 Spine 본과 별도의 가상 Spring Bone을 1:1 매핑. 애니메이션과 물리의 동시 소유 방지 |
| 11:04–13:34 | 애니메이션 적용 → 첫 world 갱신 → 가상 체인 계산 → Spine local에 결과 쓰기 → world 재갱신 → 렌더 |
| 14:48–16:34 | setup/rest 목표와 체인 앵커를 scene tool로 시각화 |
| 16:44–17:57 | 속도 반대 방향 감쇠력 |
| 18:01–19:04 | 주변 보조 스프링으로 지나친 늘어짐을 억제하고 탄성을 높임 |
| 19:12–19:47 | animation/physics blend, 고정 위치, 물리 무시, 길이 고정, 신장/이동 제한 |
| 20:46–21:39 | C# Job System + Burst. 직렬 부모→자식 대신 한 physics update에서 모든 물리 본을 동시에 계산하며, 이 계산을 충분히 자주 하면 계층 오차가 줄어든다고 설명 |
| 21:45–23:43 | Job에는 값 형식 임시 데이터(`SpringBoneAccess`)만 전달하고, Spine/managed 객체와 복사 경계를 둠 |
| 24:10–26:18 | `Update`에서 완료 결과를 렌더 본에 되쓰고, `FixedUpdate`에서 이전 Job 완료·복사·다음 입력 준비·schedule을 수행하는 비동기 handoff 설명 |
| 27:03–30:08 | `NativeArray`와 delta time을 받는 chain job 안에서 변위, 복원력, 감쇠, 가속도, 속도, 위치를 계산 |
| 31:39–32:08 | 체인이 계단처럼 꺾이는 현상을 줄이기 위해 자식 물리 이동에 연동해 부모를 보정했다고 보고. 상세 식은 공개되지 않음 |
| 32:18–34:06 | 보이지 않는 animation-only ghost skeleton 하나와 별도로 유지한 physics-only spring-chain/skeleton 결과를 additive, average, override로 혼합 |
| 34:08 이후 적분기 슬라이드 | 발표 설명에는 오류가 있었고 발표자 정정문이 우선한다. 기존 구현은 Euler-Cromer였으며 발표 뒤 NIKKE 구현을 RK4로 교체 |

### 4.2 공개 발표의 역사적 실행 구조와 본 프로젝트 구조

발표의 구현과 이 프로젝트가 채택할 단순 기준 구현을 구분한다.

**2022 발표에서 설명한 역사적 구현**

```text
Update:
  완료된 물리 결과를 SpringBone에서 SkeletonBone local pose로 복사
  → 렌더

FixedUpdate:
  이전에 schedule한 Job이 남아 있으면 Complete
  → 결과를 임시/렌더 측 데이터로 복사
  → 현재 animation world pose와 물리 파라미터를 값 형식 access 배열에 복사
  → 다음 SpringChainJob schedule

매 FixedUpdate의 SpringChainJob:
  모든 물리 본을 현재 parent/setup 입력에 대해 동시에 계산
  → 다음 FixedUpdate에서 새 입력으로 새 Job을 schedule
```

이 설명은 비동기 handoff와 **update당 동시 계산** 구조를 보여준다. 발표자는 이 처리를 충분히 자주 수행하면 부모→자식 계층 오차가 줄어든다고 설명한다. 근거가 확인되는 계산 단위는 FixedUpdate마다 schedule되는 새 Job이며, 정확한 fixed timestep, dependency/batch 구성, frame latency는 공개되지 않았다.

**본 프로젝트의 단순·결정적 기준 구현**

```text
입력/AI가 전투 상태와 recoil impulse를 갱신
    ↓
Spine AnimationState.Update(deltaTime)
    ↓
AnimationState.Apply(skeleton)                 // 렌더 본 local pose
    ↓
Skeleton.UpdateWorldTransform                  // 첫 world pose
    ↓
앵커/목표 위치를 virtual spring chain에 복사
    ↓
fixed-step RK4 simulation, serial parent → child
    ↓
virtual 결과를 대응 Spine 본의 local 회전/이동으로 변환
    ↓
Skeleton.UpdateWorldTransform                  // 두 번째 world pose
    ↓
SkeletonRenderer.LateUpdate                    // mesh/material 생성 및 렌더
```

이 직렬 순서는 공개 구현의 정확한 복제가 아니라 vertical slice에서 검증하기 쉬운 **본 프로젝트 결정**이다. 성능 측정 뒤 update당 동시 계산 방식으로 바꿀 수 있으나, 두 방식의 결과가 같다고 가정하지 않는다.

순서를 바꾸면 다음 문제가 난다.

- 물리를 애니메이션보다 먼저 적용: 같은 프레임의 애니메이션이 물리 결과를 덮어씀.
- 두 번째 world 갱신 생략: local 값은 바뀌었지만 attachment가 이전 world pose에 남음.
- 직렬 기준 구현에서 자식부터 계산: 부모의 최신 물리 결과를 반영하지 못해 체인이 끊기거나 진동함. update당 동시 계산 구현은 같은 update의 최신 부모 결과를 보지 않으므로 update frequency별 오차와 안정성을 별도로 검증해야 함.
- 실제 Spine 본을 시뮬레이션 상태 그 자체로 사용: animation apply 때 이전 프레임 물리가 덮이며 feedback 경계가 불명확해짐.

### 4.3 역사적 ghost skeleton 혼합

발표는 보이지 않는 **animation-only ghost skeleton 하나**에 animation 값을 보존하고, 별도로 유지한 **physics-only spring-chain/skeleton 결과**와 합치는 다음 세 혼합 모드를 제공했다고 설명한다. 두 개의 ghost skeleton이 있었다는 뜻이 아니다.

| 모드 | 공개 설명에서의 역할 | 구현 시 주의 |
| --- | --- | --- |
| additive | animation과 physics의 position contribution을 additive 방식으로 결합 | 정확한 기준 pose, 공간, 식은 공개되지 않음 |
| average | 두 결과를 평균/혼합 | 정확한 가중식은 공개되지 않았으므로 단순 0.5 lerp로 단정하지 않음 |
| override | physics 결과가 대상 변환을 대체 | animation과 physics의 중복 소유 경계를 명확히 해야 함 |

§14.10의 `lerp(restTarget, simulatedPosition, physicsWeight)`는 이 역사적 3모드 시스템을 그대로 재현한 것이 아니라, 본 프로젝트가 시작하기 위해 단순화한 출력 혼합안이다.

---

## 5. Spine 원화와 파츠 준비

### 5.1 원화 납품 규칙

- 기준 포즈는 게임 화면에서 가장 많이 보이는 후면 3/4 사격 포즈로 한다.
- 캐릭터마다 `stand/cover/aim`을 별도 skeleton으로 쪼개기보다, 프로토타입은 하나의 skeleton과 skin/animation으로 통합하는 것을 우선한다. atlas 메모리나 draw order 복잡도가 문제일 때만 분리한다.
- 각 움직이는 파츠 아래에는 회전 시 드러날 수 있는 픽셀을 충분히 복원한다. 어깨 뒤 몸통, 머리 뒤 목, 치마 아래 다리, 머리카락 아래 얼굴이 특히 중요하다.
- 좌우가 시각적으로 다른 3/4 포즈이므로 무조건 대칭 복사하지 않는다.
- 총은 2D attachment 또는 별도 3D/2D weapon renderer로 교체할 수 있게 손과 총의 연결점을 분리한다.

### 5.2 권장 레이어 그룹

```text
CHARACTER
├─ back_fx
├─ back_hair
│  ├─ hair_back_01 ... n
│  └─ hair_side_back_l/r
├─ back_accessory
├─ body
│  ├─ leg_back, leg_front
│  ├─ pelvis, torso
│  ├─ chest_l, chest_r
│  ├─ arm_back, arm_front
│  ├─ hand_l, hand_r
│  └─ neck
├─ clothes
│  ├─ coat_back, coat_front
│  ├─ skirt panels
│  ├─ ribbon/strap/cable
│  └─ armor/accessories
├─ weapon
│  ├─ weapon_body
│  ├─ magazine
│  ├─ bolt/slide
│  ├─ muzzle
│  └─ shell
├─ head
│  ├─ face
│  ├─ eye_l/r, pupil_l/r, eyelid_l/r
│  ├─ brow_l/r
│  ├─ mouth variants
│  └─ front_hair pieces
└─ front_fx
```

### 5.3 메쉬 원칙

- rigid 파츠는 region attachment를 우선한다.
- 얼굴, 몸통, 가슴, 긴 머리, 천처럼 굽어야 하는 파츠만 mesh attachment로 만든다.
- 메쉬 외곽은 실루엣을 따르고, 관절과 굴곡 구간에 edge loop를 집중한다.
- 투명 픽셀 안쪽에 불필요한 vertex를 촘촘히 넣지 않는다.
- 한 vertex가 4개를 넘는 본에 가중되지 않게 한다. 대부분 2개, 관절 경계만 3개를 목표로 한다.
- 물리 본 체인을 따라 긴 파츠의 vertex strip을 배치한다. 본 사이에 최소 한 줄 이상의 중간 vertex가 있어야 곡률이 부드럽다.
- 서로 맞닿는 분리 mesh는 Spine 4.3의 weld/weight copy 기능을 사용해 경계 가중치를 맞춘다.

---

## 6. 본 명명 규칙과 전체 계층

### 6.1 접두사

| 접두사 | 용도 | 소유자 |
| --- | --- | --- |
| `drv_` | 애니메이터가 키를 넣는 몸체/앵커 | Spine animation |
| `phys_` | 물리 결과가 기록되는 렌더 본 | Unity Spring Rig |
| `def_` | deformation 전용 보조 본 | Spine animation 또는 constraint |
| `ik_` | IK target/control | Spine constraint/Unity aim |
| `att_` | 총구, 손잡이, 탄피 등 attachment 기준점 | Unity gameplay/VFX |
| `fx_` | VFX 위치 | Unity VFX |
| `ui_` | 머리 위 UI 등 | Unity UI |
| `col_` | 물리 충돌체 authoring marker | Spring Rig authoring |

사이드 표기는 `_l`, `_r`, 중앙은 `_c`, 체인 인덱스는 2자리 `_01` 형식을 쓴다. 이름을 runtime string으로 반복 탐색하지 않고 import 시 index로 resolve한다.

### 6.2 제안 전체 트리

아래는 필수 골격의 기준안이다. 캐릭터 디자인에 없는 파츠는 제거할 수 있지만, 역할과 소유 경계는 유지한다.

```text
root
└─ drv_world
   └─ drv_character
      ├─ drv_body
      │  └─ drv_pelvis
      │     ├─ drv_spine_01
      │     │  └─ drv_spine_02
      │     │     └─ drv_chest
      │     │        ├─ def_torso_l
      │     │        ├─ def_torso_r
      │     │        ├─ drv_neck
      │     │        │  └─ drv_head
      │     │        │     ├─ drv_face
      │     │        │     │  ├─ drv_eye_l
      │     │        │     │  ├─ drv_eye_r
      │     │        │     │  ├─ drv_brow_l
      │     │        │     │  └─ drv_brow_r
      │     │        │     ├─ drv_jaw
      │     │        │     ├─ drv_hair_anchor_back
      │     │        │     │  ├─ phys_hair_back_l_01 → 02 → 03 → 04
      │     │        │     │  ├─ phys_hair_back_c_01 → 02 → 03 → 04
      │     │        │     │  └─ phys_hair_back_r_01 → 02 → 03 → 04
      │     │        │     ├─ drv_hair_anchor_side_l
      │     │        │     │  └─ phys_hair_side_l_01 → 02 → 03
      │     │        │     ├─ drv_hair_anchor_side_r
      │     │        │     │  └─ phys_hair_side_r_01 → 02 → 03
      │     │        │     ├─ drv_hair_anchor_front
      │     │        │     │  ├─ phys_hair_front_l_01 → 02
      │     │        │     │  └─ phys_hair_front_r_01 → 02
      │     │        │     └─ att_head_fx
      │     │        ├─ drv_clavicle_l
      │     │        │  └─ drv_upperarm_l → drv_forearm_l → drv_hand_l
      │     │        │     ├─ ik_hand_l
      │     │        │     └─ att_grip_l
      │     │        ├─ drv_clavicle_r
      │     │        │  └─ drv_upperarm_r → drv_forearm_r → drv_hand_r
      │     │        │     ├─ ik_hand_r
      │     │        │     └─ att_grip_r
      │     │        ├─ drv_chest_anchor_l
      │     │        │  └─ phys_chest_l_01 → phys_chest_l_02
      │     │        ├─ drv_chest_anchor_r
      │     │        │  └─ phys_chest_r_01 → phys_chest_r_02
      │     │        ├─ drv_shoulder_accessory_anchor_l/r
      │     │        └─ drv_back_accessory_anchor
      │     ├─ drv_thigh_l → drv_shin_l → drv_foot_l
      │     ├─ drv_thigh_r → drv_shin_r → drv_foot_r
      │     ├─ drv_coat_anchor_back
      │     │  ├─ phys_coat_back_l_01 → 02 → 03
      │     │  ├─ phys_coat_back_c_01 → 02 → 03
      │     │  └─ phys_coat_back_r_01 → 02 → 03
      │     ├─ drv_skirt_anchor
      │     │  ├─ phys_skirt_l_01 → 02
      │     │  ├─ phys_skirt_c_01 → 02
      │     │  └─ phys_skirt_r_01 → 02
      │     └─ drv_hip_accessory_anchor_l/r
      ├─ drv_aim_root
      │  ├─ ik_aim_target
      │  ├─ ik_elbow_hint_l
      │  └─ ik_elbow_hint_r
      ├─ drv_weapon_root
      │  ├─ drv_weapon_recoil
      │  │  ├─ att_weapon_body
      │  │  ├─ att_grip_primary
      │  │  ├─ att_grip_secondary
      │  │  ├─ att_muzzle
      │  │  ├─ att_shell_eject
      │  │  ├─ att_magazine
      │  │  └─ fx_muzzle
      │  └─ ik_weapon_target
      ├─ drv_cover_root
      │  ├─ att_cover_contact_l
      │  └─ att_cover_contact_r
      ├─ drv_accessory_root
      │  ├─ drv_ribbon_anchor_* → phys_ribbon_*_01 → 02 → 03
      │  ├─ drv_strap_anchor_*  → phys_strap_*_01  → 02 → 03
      │  └─ drv_cable_anchor_*  → phys_cable_*_01  → 02 → 03
      ├─ fx_root
      │  ├─ fx_hit
      │  ├─ fx_burst
      │  └─ fx_defeat
      └─ ui_root
         └─ ui_status_anchor
```

### 6.3 부위별 상세 설계

| 부위 | 본 수 기준 | 애니메이션 소유 | 물리 소유 | 주의점 |
| --- | ---: | --- | --- | --- |
| pelvis–chest | 3 spine + chest | base track | 없음 | 큰 자세와 호흡. 물리가 몸 전체 자세를 결정하지 않음 |
| torso deformation | 좌우 1–2 | base/aim | 없음 | 3/4 포즈의 비대칭 실루엣 유지 |
| arms/hands | 관절당 3 + IK | aim/fire/reload | 없음 | 총 grip과 IK를 우선. 물리 금지 |
| chest | 좌우 anchor + 2 | anchor는 animation | `phys_chest_*` | 좌우 파츠가 겹치거나 서로 관통하지 않게 ellipse limit 사용 |
| back hair | 큰 덩어리당 3–5 | anchor만 animation | 각 chain | 가닥 하나마다 체인을 만들지 말고 실루엣 덩어리 기준 |
| side/front hair | 2–3 | anchor만 animation | 각 chain | 얼굴 collider와 각도 cone 필수 |
| coat/skirt | 패널당 2–4 | anchor만 animation | 각 chain | 좌우 다리·cover collider와 충돌 |
| ribbon/strap/cable | 2–5 | anchor만 animation | 각 chain | fixed-length와 max stretch를 강하게 적용 |
| weapon | root/recoil/markers | fire/reload | procedural recoil만 | Spring Rig에 넣지 않음. gameplay cadence가 소유 |
| face | 눈/눈썹/턱 | expression track | 없음 | blink/말/피격 표정 분리 |

### 6.4 본 소유권 불변식

각 프레임에서 본의 각 변환 속성은 한 시스템만 최종 소유한다.

```text
drv_*          = Spine animation/constraint
phys_*         = custom Spring Rig output
drv_weapon_recoil = procedural recoil controller, animation은 기준값만 제공
ik_*           = aim/reload controller 또는 Spine IK, 둘 중 하나
```

- `phys_*`에는 rotation/translation key를 넣지 않는 것이 원칙이다. 필요하면 setup pose와 물리 mix만 사용한다.
- 연출상 물리를 잠시 끄려면 animation이 동일 본을 덮지 말고 `physicsWeight=0`으로 줄여 rest target을 따라가게 한다.
- Spine Physics constraint가 활성인 본은 custom Spring Rig mapping에서 제외한다.
- transform constraint, IK, path constraint, physics의 평가 순서는 authoring sheet에 명시하고 import validator가 중복 소유를 오류 처리한다.

---

## 7. Slot, draw order, skin, atlas

### 7.1 slot 계층

slot 이름은 attachment 역할을 말해야 하며 본 이름을 그대로 복제하지 않는다.

```text
slot_back_fx
slot_hair_back_*
slot_leg_back
slot_coat_back_*
slot_torso
slot_chest_l/r
slot_arm_back
slot_weapon_back
slot_face
slot_eye_*, slot_brow_*, slot_mouth
slot_hair_front_*
slot_arm_front
slot_weapon_front
slot_coat_front_*
slot_front_fx
```

- draw order가 자주 바뀌는 묶음은 Spine 4.3 draw-order folder로 정리한다.
- 총이 팔 앞/뒤를 오가야 하면 전체 slot을 난잡하게 재정렬하지 말고 `weapon_back`/`weapon_front` attachment 전환 또는 제한된 draw-order key를 쓴다.
- clipping attachment는 큰 범위를 중첩하지 않는다. 지속적인 다중 clipping은 CPU mesh 비용과 디버깅 난도를 높인다.

### 7.2 skin 전략

- `base`: 필수 몸체와 얼굴
- `costume_*`: 의상과 의상 전용 본/constraint
- `weapon_smg`, `weapon_sg`, `weapon_sr`, `weapon_rl`, `weapon_ar`, `weapon_mg`
- `expression_*`: 가능하면 attachment variant로 처리하며 별도 skin은 대량 표정 세트에만 사용
- skin 전용 본과 constraint는 같은 skin 활성 범위를 가져야 한다.

### 7.3 atlas/material

- 한 캐릭터의 상시 전투 파츠는 가능한 한 1 atlas page, 최대 2 page를 목표로 한다.
- PMA(Pre-multiplied Alpha) export와 material shader 설정을 일치시킨다.
- 월드 캐릭터는 조명이 필요하지 않으면 URP용 unlit Spine shader를 사용한다.
- normal/tangent를 요구하지 않는 shader에서는 `Calculate Tangents`, normals 생성을 끈다.
- additive VFX가 PMA material과 함께 batch 가능한지 확인하고, 별도 material 남발로 submesh/draw call이 늘지 않게 한다.
- `Immutable Triangles`는 런타임에 attachment topology와 draw order가 바뀌지 않는 skeleton에만 켠다. 이 프로젝트는 무기/표정/엄폐 전환이 있으므로 캐릭터별 검증 전에는 기본 off다.
- `Use Single Submesh`는 여러 material, separation, slot material override가 필요 없는 캐릭터만 사용한다.

---

## 8. Weight, constraint, aim, weapon

### 8.1 몸체 weight

- pelvis–spine–chest는 회전할 때 복부가 접히는 방향에 edge loop를 배치한다.
- chest mesh는 `drv_chest`의 큰 움직임과 `phys_chest_*`의 국소 변형을 합성한다.
- 가슴 물리는 attachment 전체를 rigid 이동시키지 않고, 바깥/아래쪽 vertex의 `phys_chest_02` 가중치를 높여 내부 접점은 torso에 붙인다.
- 머리카락 뿌리 vertex는 anchor/첫 phys 본에 강하게 붙이고 끝으로 갈수록 다음 본 가중치를 증가시킨다.

### 8.2 aim 구조

- `drv_aim_root`는 조준 방향의 큰 상체 회전을 제공한다.
- 손은 `att_grip_primary/secondary`를 목표로 IK를 적용한다.
- 총의 기준 자세는 `drv_weapon_root`; 순간 반동은 `drv_weapon_recoil`에만 더한다.
- 3/4 고정 화면에서는 무제한 360도 aim을 만들지 않는다. 캐릭터별 허용 yaw/pitch에 해당하는 2D 각도 범위만 지원한다.
- aim 입력은 저역통과 또는 critically damped smoothing을 적용하되, 실제 명중 판정은 시각 IK가 아니라 gameplay target에서 결정한다.

### 8.3 2D 총과 3D 총 호환

- 2D 총: `att_weapon_body` 아래 region/mesh attachment 사용.
- 3D 총: Spine attachment는 숨기고 `att_weapon_body`, muzzle, grip marker의 world transform으로 별도 GameObject를 배치한다.
- 어떤 방식이든 muzzle flash, projectile origin, shell eject는 동일 marker 계약을 사용한다.
- 총기 교체 시 본 트리를 바꾸지 않고 weapon skin + metadata만 교체한다.

---

## 9. 필수 애니메이션 세트

### 9.1 공통 clip

| 이름 | loop | 용도 | 종료 조건 |
| --- | --- | --- | --- |
| `idle` | yes | 비전투/장시간 입력 없음 | combat 진입 |
| `cover_enter` | no | 엄폐 진입 | complete 또는 `cover_hidden` event |
| `cover_idle` | yes | 엄폐 유지 | AI 결정 |
| `aim_enter` | no | 엄폐 해제 및 사격 자세 진입 | complete |
| `aim_idle` | yes | 조준 유지 | cover/reload/defeat |
| `fire_start` | no | 연사 시작의 anticipation | complete |
| `fire_loop` | yes | 연사 중 지속 자세 | cadence queue 종료/탄창 없음 |
| `fire_end` | no | follow-through | complete |
| `fire_single` | no | SR/SG/RL 단발용 | complete |
| `reload` | no | 공통 fallback | `reload_commit` + complete |
| `burst` | no | 개별 버스트 | `burst_commit` 또는 complete |
| `hit_light` | no | 경피격 overlay | complete |
| `hit_heavy` | no | 큰 피격 | complete |
| `defeat` | no | 전투 불능 | game reset |
| `blink` | no | 표정 overlay | complete |

### 9.2 무기별 차이

| 무기 | 기본 형태 | 반동/동작 | 애니메이션 규칙 |
| --- | --- | --- | --- |
| SMG | 연사 | 작고 빠른 impulse | `fire_loop` 유지, shot마다 애니메이션 재시작 금지 |
| AR | 연사/점사 | 중간 impulse | loop + cadence event/절차 반동 |
| MG | 장기 연사 | 누적 sway, spin-up 가능 | start/loop/end 분리, 과열은 후속 범위 |
| SG | 단발 | 큰 recoil, pump/bolt 가능 | `fire_single`, recovery가 다음 cadence보다 길 수 있음 |
| SR | 단발 | 큰 상체 recoil, 긴 재정렬 | 실제 shot 시점 event, aim 복귀 분리 |
| RL | 단발 | 큰 후방 follow-through | muzzle/backblast marker, 긴 recovery |

### 9.3 애니메이션 제작 원칙

- 키 애니메이션은 silhouette, anticipation, weapon handling, body recoil을 만든다.
- 머리카락·천 끝단의 반복 흔들림은 키로 과도하게 만들지 않는다. anchor의 의도적 움직임만 키로 주고 Spring Rig가 follow-through를 만든다.
- 물리 off 상태에서도 포즈가 무너지지 않게 setup pose와 base animation을 완결한다.
- `fire_loop`의 첫/마지막 pose는 loop seam에서 torso/weapon이 튀지 않게 맞춘다.
- 실제 탄 발사와 대미지는 애니메이션 frame rate에 종속하지 않는다.

---

## 10. AnimationState track 설계

### 10.1 track 배치

| Track | 역할 | 예시 |
| ---: | --- | --- |
| 0 | full-body base state | `idle`, `cover_*`, `aim_*`, `reload`, `defeat` |
| 1 | upper-body fire overlay | `fire_start/loop/end`, `fire_single` |
| 2 | reaction/recoil overlay | `hit_light`, 특수 procedural-compatible clip |
| 3 | face | `blink`, expression |
| 4 | burst/cinematic override | `burst`; 필요 시 lower track를 alpha 0 또는 empty mix |

Spine의 높은 track이 낮은 track 위에 적용된다. 각 overlay animation은 자신이 소유할 본만 key해야 한다. 예를 들어 face clip이 chest rotation을 key하면 안 된다.

### 10.2 mix 시작값

| 전환 | mix duration 시작값 |
| --- | ---: |
| idle → aim_enter | 0.12 s |
| aim_idle → cover_enter | 0.10 s |
| cover_idle → aim_enter | 0.08 s |
| aim_idle ↔ fire overlay | 0.04–0.08 s |
| fire → reload | 0.06 s |
| any → hit_light overlay | 0.03 s |
| any → defeat | 0.15 s, 단 gameplay 즉시 정지 |
| any → burst | 0.10 s 또는 캐릭터 연출별 명시 |

이 값은 튜닝 시작값이다. mix 동안 attachment/draw-order 전환이 필요한 clip은 `mixAttachmentThreshold`와 `mixDrawOrderThreshold`를 명시적으로 검증한다.

### 10.3 상태 우선순위

```text
Defeat
  > Burst cinematic lock
  > Heavy hit / forced reaction
  > Reload
  > Cover enter/exit
  > Fire
  > Aim idle
  > Idle
```

- `defeat` 진입 시 모든 다른 track을 clear하고 물리 impulse 입력을 중지한다. Spring Rig는 즉시 reset하지 않고 짧게 settle한 뒤 freeze할 수 있다.
- burst가 전신 연출이면 fire/reload를 interrupt하고 gameplay commit event를 별도로 보장한다.
- reload가 interrupt되면 탄약 commit 전/후를 이벤트 상태로 구분한다.
- `TrackEntry.Complete`는 interrupt된 entry에 발생하지 않을 수 있다. 필수 gameplay 전이는 Complete 하나에만 의존하지 않는다.

---

## 11. 입력 큐, cadence, 연속 사격

### 11.1 불변 규칙

`input event != shot != animation restart`

- 입력은 담당 캐릭터의 bounded attack queue에 토큰을 추가한다.
- `WeaponCadenceController`가 fire interval, ammo, reload, target 유효성을 검사하고 토큰을 실제 shot으로 소비한다.
- 실제 shot이 발생할 때만 damage, ammo 감소, muzzle, shell, recoil impulse, burst gauge가 실행된다.
- `fire_loop`는 queue가 활성인 동안 한 번 시작하고 유지한다.
- 같은 loop가 재생 중이면 `SetAnimation`을 다시 호출하지 않는다.

### 11.2 연사 상태 흐름

```text
첫 queue token
  → aim 확보
  → fire_start 1회
  → fire_loop 유지
  → cadence tick마다 ExecuteShot()
  → queue 비고 grace window 경과
  → fire_end 1회
  → aim_idle 또는 cover
```

- grace window 시작값: SMG/AR/MG는 `1.25 × shotInterval`, 최대 0.12 s.
- 새 토큰이 grace 안에 들어오면 loop를 끊지 않는다.
- queue 최대치와 보관 시간은 gameplay spec이 소유한다. animation layer가 임의로 토큰을 버리지 않는다.
- loop clip 내부의 `fire` event를 실제 shot clock으로 쓰지 않는다. frame drop과 cadence 변화 때문에 gameplay가 clip에 종속되기 때문이다.
- 대신 실제 `ExecuteShot()`이 recoil phase, VFX, audio를 호출한다. clip event는 연출 marker 또는 단발 clip의 동기화 보조로만 사용한다.

### 11.3 단발 무기

- SG/SR/RL은 queue token 하나당 `fire_single` entry를 재생할 수 있다.
- 단, 현재 entry가 recovery 중일 때 새 입력이 들어와도 즉시 restart하지 않고 cadence가 허용하는 다음 shot까지 queue한다.
- shot은 `fire` event 시점 또는 정규화된 fire phase에 맞추되, 이벤트 유실에 대비해 `WeaponCadenceController`가 commit을 보장한다.

---

## 12. Spine 이벤트 계약

| 이벤트 | payload | 소비자 | 규칙 |
| --- | --- | --- | --- |
| `fire_pose` | optional int phase | animation controller | 시각 pose 동기화용, damage 직접 처리 금지 |
| `muzzle_on` | marker name | VFX | 한 프레임 flash 또는 effect spawn |
| `shell_eject` | marker name | VFX/pool | 탄피 없는 무기는 생략 |
| `mag_out` | weapon slot | reload visual | magazine attachment 숨김/분리 |
| `mag_in` | weapon slot | reload visual | 새 magazine 표시 |
| `reload_commit` | none | weapon controller | 탄약 gameplay commit. 중복 호출 idempotent |
| `cover_hidden` | none | visibility/AI | 엄폐 완료 marker |
| `burst_commit` | skill id | burst system | 효과 commit. 캐릭터별 fallback timeout 필요 |
| `hit_peak` | intensity | VFX/camera | 피격 연출 peak |
| `defeat_commit` | none | game over | 패배 상태 시각 확정 |

- 문자열은 editor용이며 runtime은 import 시 `EventData`/hash/id로 캐시한다.
- `Start`, `Interrupt`, `End`, `Dispose`, `Complete`, custom Event를 각각 구분한다.
- gameplay commit 이벤트는 entry별 token으로 중복 실행을 막는다.
- 이벤트 callback 안에서 같은 entry를 무분별하게 clear/set하여 재진입하지 않는다. 상태 변경 요청을 queue하고 callback 종료 후 처리한다.

---

## 13. Spring Rig 데이터 모델

### 13.1 왜 가상 본을 별도로 두는가

렌더 본 자체의 현재 위치를 곧바로 다음 시뮬레이션 상태로 쓰면 AnimationState가 매 프레임 local pose를 다시 적용하는 시점과 충돌한다. 가상 본은 다음을 분리한다.

- 애니메이션이 제시한 현재 목표/rest pose
- 물리가 기억하는 위치와 속도
- 렌더 skeleton에 기록할 최종 결과

Spine tree에는 `phys_*` 렌더 본만 존재한다. `sim_*` 데이터는 Unity의 연속 메모리에 저장한다.

### 13.2 `SpringRigAsset` 정적 데이터

캐릭터 또는 의상별 ScriptableObject에 다음을 저장한다.

```text
RigVersion
SkeletonDataAsset GUID / expected skeleton hash
Chains[]
  id
  anchorBoneName
  outputBoneNames[]
  parentVirtualIndex[]
  restLength[]
  restLocalDirection[]
  mass[]
  stiffness[]
  damping[]
  gravityScale[]
  inertia[]
  animationBlend[]
  fixedLength[]
  maxStretchRatio[]
  angularLimitMin/Max[]
  motionRadius[]
  collisionRadius[]
  solverGroup
AuxiliarySprings[]
  virtualA, virtualB, restDistance, stiffness, damping
Colliders[]
  sourceBoneName, shape, localOffset, radius/capsule endpoints
LODProfiles[]
  distance/visibility rule, frequency, enabled chain mask
```

런타임 시작 시 bone name을 `BoneData.Index`로 resolve하고 이후 문자열 lookup을 금지한다. 누락/중복 mapping은 캐릭터 spawn을 실패시키는 authoring 오류다.

### 13.3 `SpringRigRuntime` 동적 SoA

Job/Burst용 동적 데이터는 구조체 배열보다 SoA를 우선한다.

```text
NativeArray<float2> position
NativeArray<float2> previousPosition
NativeArray<float2> velocity
NativeArray<float2> restTarget
NativeArray<float2> anchorPosition
NativeArray<float>  restLength
NativeArray<float>  massInv
NativeArray<float>  stiffness
NativeArray<float>  damping
NativeArray<float>  blend
NativeArray<int>    parentIndex
NativeArray<int>    outputBoneIndex
NativeArray<byte>   flags
```

렌더 skeleton 객체는 managed이므로 Burst job 안에서 직접 접근하지 않는다. main thread에서 입력 world pose를 NativeArray에 복사하고, job 완료 뒤 결과를 Spine local transform으로 기록한다.

### 13.4 체인 그룹과 dependency

- 서로 독립인 hair/coat/chest chain은 병렬 job으로 실행할 수 있다.
- 한 chain 내부의 개념적 의존성은 부모→자식이다. 기준 구현은 직렬 계산하지만, 공개 NIKKE 발표처럼 매 physics update에서 모든 본을 동시에 계산하는 병렬 방식도 후보에 둔다. 발표는 계산을 충분히 자주 하면 계층 오차가 줄어든다고 설명한다. 두 방식은 수치적으로 동일하지 않다.
- auxiliary spring이 서로 다른 chain을 연결하면 두 chain을 같은 solver island로 묶는다.
- 3캐릭터 전체를 bone 하나씩 job으로 쪼개지 않는다. scheduling overhead를 고려해 `character × island` 또는 여러 작은 chain batch를 측정한다.

### 13.5 Spine 4.2 공식 Physics constraint의 실제 동작

이 절의 “4.2”는 Physics constraint가 공개 도입된 원형을 뜻한다. 현재 프로젝트는 4.3 runtime을 잠그고 있으므로 §13.5.7의 차이까지 함께 적용한다. 공식 기능과 NIKKE 커스텀 Spring Rig는 이름만 비슷할 뿐 상태, 적분, authoring, 실행 경계가 다르다. 속성의 사용자 의미는 [공식 Physics Constraints 가이드](https://esotericsoftware.com/spine-physics-constraints), runtime 세부는 §23의 SHA 고정 source를 기준으로 했다.

#### 13.5.1 Editor authoring 의미

- Physics constraint는 **한 Spine bone의 unconstrained world pose에 물리 offset을 계산해 적용**한다. 별도의 질점 체인이나 ghost skeleton을 자동으로 만들지 않는다.
- 영향 축은 `X`, `Y`, `Rotate`, `Scale X`, `Shear X`이며 필요한 축만 켠다. `Limit`은 physics에 영향을 주는 translation의 최대 속도를 정하며, 최종 offset의 절대 범위를 clamp하는 값은 아니다. 서로 다른 축에 다른 설정이 필요하면 같은 bone에 constraint를 여러 개 둘 수 있다.
- setup slider는 힘이나 질량 자체가 아니라 setup pose에서 해당 물리 영향이 적용되는 정도를 authoring한다. `0`이면 unconstrained pose를 완전히 따라 물리 offset이 없고, `100`이면 bone 이동으로 생기는 물리 offset을 최대로 반영한다. 특히 inertia가 만드는 값은 bone 이동에서 유도된 **unconstrained pose 대비 지연 offset**이며, 이를 물리학의 관성력이나 mass와 같은 값으로 해석하면 안 된다.
- `Inertia`, `Strength`, `Damping`, `Mass`, `Wind`, `Gravity`, `Mix`는 animation timeline으로 key할 수 있다. 각 속성의 `Global` flag는 여러 Physics constraint를 하나의 timeline row로 함께 제어하게 한다. `Mix=0`이면 해당 constraint는 Physics mode 분기와 `lastTime` 갱신 전에 조기 종료한다. 따라서 `Physics.Reset` mode도 이 constraint를 reset하지 못하며, 오랜 뒤 다시 켜면 그 사이 delta를 한꺼번에 보려 할 수 있다. 장시간 비활성 뒤에는 direct `Reset()` 또는 reset timeline을 사용한다.
- 각 constraint의 `FPS`가 내부 고정 step을 정한다. 4.2 가이드는 보통 30–60 FPS를 권장하며, FPS를 바꾸면 단순 품질뿐 아니라 움직임 자체가 달라질 수 있다고 경고한다. export에는 값을 명시하고 버전 기본값에 기대지 않는다.
- 회전, scale, shear 영향은 bone length가 0이면 의미 있는 방향/끝점을 만들지 못하므로 zero-length bone을 피한다. 활성 축이 하나도 없거나 skin bone/constraint 활성 범위가 어긋난 설정도 authoring 오류다.
- `Reference scale`은 authoring 크기와 runtime 크기 사이에서 힘과 회전/scale 계산의 기준을 제공한다. 프로젝트의 pixels-per-unit 또는 root scale을 바꿀 때 같은 숫자가 같은 체감이라고 가정하지 말고 함께 재검증한다.

#### 13.5.2 4.2 runtime 상태

4.2 `PhysicsConstraint`는 constraint마다 다음 상태를 가진다.

```text
unconstrained pose 추적: ux, uy, cx, cy, tx, ty
축별 동적 offset/velocity:
  xOffset, xVelocity
  yOffset, yVelocity
  rotateOffset, rotateVelocity
  scaleOffset, scaleVelocity
fixed-step 상태: remaining, lastTime
제어 상태: reset
setup/runtime 파라미터:
  inertia, strength, damping, massInverse, wind, gravity, mix
```

이는 `position[]/velocity[]`를 가진 N개 virtual point chain이 아니다. constraint 하나가 대상 bone의 translation/rotation/scale/shear 채널 offset을 기억하는 구조다. `SetToSetupPose()`는 파라미터를 setup 값으로 되돌리지만 동적 offset과 velocity까지 지우지 않는다. 동적 상태는 direct `Reset()`으로 지우며, `Physics.Reset` mode는 `mix>0`일 때만 그 경로에 도달한다.

#### 13.5.3 bone 이동 감지와 fixed-step 갱신

4.2 source의 핵심을 축약하면 다음과 같다. 실제 rotation/scale 경로에는 bone geometry와 reference scale 보정이 더 들어가므로 아래 식은 X/Y 계열의 구조를 설명하기 위한 의사식이다.

```text
step = 1 / exportedPhysicsFPS
remaining += frameDelta

// 현재 unconstrained bone world pose가 이전 pose에서 이동한 양을 감지
inertiaInput = (previousWorld - currentWorld) * inertia
axisLimit = data.limit * frameDelta * abs(skeletonAxisScale)
offset += clamp(inertiaInput, -axisLimit, +axisLimit)

while remaining >= step:
  decay = pow(damping, 60 * step)
  massStep = massInverse * step

  xVelocity += (wind    - xOffset * strength) * massStep
  xOffset   += xVelocity * step
  xVelocity *= decay

  yVelocity -= (gravityScaled + yOffset * strength) * massStep
  yOffset   += yVelocity * step
  yVelocity *= decay

  remaining -= step

apply offset * mix to target bone world transform
UpdateAppliedTransform()
```

source가 “속도 갱신 뒤 offset 갱신” 순서로 계산한다는 사실은 확인되지만, 이 문서에서는 공식 명칭이 없는 적분기 이름을 임의로 붙이지 않는다. 남은 시간 `remaining`은 다음 frame으로 이월된다. 4.2 stable에는 고정 step 사이의 render interpolation이 없어서 render rate가 physics FPS보다 높거나 두 주기가 맞지 않으면 같은 물리 pose가 여러 render frame 유지되다가 한 번에 바뀌는 stutter가 날 수 있다.

4.2에서 `Wind`는 world X, `Gravity`는 world Y 방향의 지속 입력이며 skeleton scale과 `yDown`을 반영한다. `Mass`는 가속 변화에 대한 저항을 조절하고, `Strength`는 offset을 unconstrained pose로 복귀시키며, `Damping`은 velocity를 감쇠한다. 이 의미는 §14의 `k`, `c`, virtual point mass와 1:1 호환 계약이 아니다.

#### 13.5.4 update mode, reset, 외부 이동

spine-runtimes의 world transform 갱신은 다음 `Physics` mode를 구분한다.

| mode | 4.2 runtime 의미 |
| --- | --- |
| `None` | physics를 갱신하지도 pose에 적용하지도 않음 |
| `Reset` | `mix != 0`일 때 동적 상태를 reset한 뒤 `Update`처럼 계산·적용. `mix==0`이면 mode 분기 전에 반환 |
| `Update` | delta를 소비해 동적 상태를 전진시키고 결과 적용 |
| `Pose` | 저장된 physics pose를 적용하되 시간을 전진시키지 않음 |

- direct constraint `Reset()`은 accumulator, offset, velocity를 0으로 만들고 `reset=true`로 표시한다. pose 추적값은 즉시 현재 bone에 대입되는 것이 아니라, 다음 유효한 `Update`에서 현재 unconstrained pose를 기준으로 재초기화된다. animation의 physics reset timeline은 direct reset 경로를 호출하므로 `mix==0`에서도 사용할 수 있다.
- `Physics.Reset`은 `PhysicsConstraint.Update(Physics.Reset)`의 mode이므로 `mix==0` 조기 반환 뒤에는 도달하지 않는다. zero-mix constraint를 확실히 초기화하려면 direct `Reset()` 또는 reset timeline을 사용한다.
- skeleton의 `PhysicsTranslate(x,y)`와 `PhysicsRotate(x,y,degrees)`는 constraint의 이전 pose 추적점을 함께 이동/회전시켜, skeleton root나 GameObject가 움직였을 때 그 이동을 inertia 입력으로 상속할 수 있게 한다. 따라서 built-in runtime 입력을 단순히 “불가능”이라고 하면 틀리다.
- 반대로 arbitrary chain point에 gameplay impulse를 넣거나 auxiliary spring topology를 바꾸는 고수준 API는 4.2 기본 모델에 없다. 필요한 경우 공식 파라미터/이동 API를 쓰거나 별도 custom solver가 소유해야 한다.
- teleport, spawn, 장시간 pause 뒤에는 direct `Reset()`; 정상 이동을 관성으로 반영하려면 `PhysicsTranslate/Rotate`; freeze된 결과만 다시 적용하려면 `Pose`를 사용한다. `SetToSetupPose`만 호출하고 reset됐다고 가정하지 않는다.

Editor의 `Simulate`는 현재 skeleton의 physics를 연속 계산한다. 이를 끄고 timeline 위치를 바꾸면 frame 0부터 현재 시점까지 skeleton 내 가장 높은 physics update rate로 fast-forward하며, export도 각 출력 frame을 같은 방식으로 평가한다. `Deterministic`은 Animate mode에서도 frame 0부터 매번 재계산해 특정 timeline 시점의 preview를 얻는 **Editor 전용 동작**이다. runtime/network lockstep 결정성을 보장한다는 뜻이 아니다. deterministic preview는 비싸고 loop 경계에서 jump할 수 있다. runtime에서 같은 효과가 필요하면 reset 후 명시적으로 fast-forward한다.

#### 13.5.5 4.2 기능 경계

- 4.2에는 Physics constraint와 연동된 collider/self-collision 시스템이 없다. 얼굴, 총, 엄폐물 충돌이 필요하면 별도 시스템을 설계한다.
- 4.2 Editor는 물리 결과를 key timeline으로 직접 bake하지 않는다. export `Warm up`은 시작 pose를 안정시키는 준비 과정이며 seamless loop를 보장하는 baking이 아니다. 4.3 Editor의 `Key Constrained` 기능은 §13.5.7처럼 별도 취급하며 4.2 사양으로 소급하지 않는다.
- 공식 Physics는 “game physics 전체”가 아니라 bone secondary motion constraint다. 여러 bone에 constraint를 배치해 chain처럼 보이게 할 수는 있지만 NIKKE의 virtual point chain, auxiliary edge, ghost skeleton, Job/Burst solver와 같은 구조가 자동으로 생기지 않는다.
- 내장 constraint와 custom solver 사이의 일반적인 성능 우열을 뒷받침하는 동일 조건 benchmark는 없다. 콘텐츠, constraint 수, world pass, Job scheduling, 기기별로 Player에서 측정한다.

#### 13.5.6 spine-unity 평가 순서

현재 계열의 `SkeletonAnimation`/`SkeletonRenderer`는 GameObject transform 이동을 감지해 설정된 inheritance 비율에 따라 skeleton의 `PhysicsTranslate`/`PhysicsRotate`로 전달한다. AnimationState를 적용한 뒤 callback 유무에 따라 world pass가 달라진다.

```text
AnimationState.Update / Skeleton.Update
ApplyTransformMovementToPhysics
AnimationState.Apply

UpdateWorld callback 없음:
  Skeleton.UpdateWorldTransform(Physics.Update)

UpdateWorld callback 있음:
  Skeleton.UpdateWorldTransform(Physics.Pose)   // 저장된 built-in pose 적용, 시간 전진 없음
  SkeletonRenderer.UpdateWorld callback        // custom constraint가 world를 읽고 local에 기록
  Skeleton.UpdateWorldTransform(Physics.Update) // built-in state 전진 + 최종 world 재계산
```

따라서 callback은 “physics가 전혀 없는 animation-only pose”를 자동으로 받는 것이 아니라, built-in의 저장 pose가 적용된 첫 pass를 받는다. custom rest target이 정말 animation-only여야 한다면 대상 subtree에서 built-in Physics를 금지하거나 별도 pose 복원 경계를 설계해야 한다. 같은 bone/속성뿐 아니라 한 solver 대상의 **ancestor**를 다른 solver가 움직여도 계층적으로 상호작용한다. 가장 안전한 기본은 서로 다른 subtree이며, 혼용 시 실제 callback 순서를 회귀 테스트한다.

#### 13.5.7 4.2 원형과 이 프로젝트의 잠긴 4.3 차이

| 항목 | Spine 4.2 원형 | 잠긴 4.3 runtime (`8a238d…`) |
| --- | --- | --- |
| fixed-step render smoothing | accumulator step 결과를 그대로 적용; step 사이 interpolation 없음 | `xLag/yLag/rotateLag/scaleLag` 상태로 고정 step 사이 결과를 보간해 높은 render FPS의 stutter를 완화 |
| pose/state 모델 | constraint 내부 setup 파라미터와 동적 state 중심 | setup/applied pose를 분리한 constraint pose 모델 강화 |
| Wind/Gravity 방향 | Wind는 world X, Gravity는 world Y에 고정하고 skeleton scale/`yDown` 반영 | skeleton의 `windX/windY`, `gravityX/gravityY` 방향 벡터로 합성; 기본값은 각각 `(1,0)`, `(0,1)` |
| constraint 결과 key/baking | Physics 결과를 timeline key로 직접 bake하는 기능 없음; `Warm up`은 baking이 아님 | 4.3 Editor의 [`Key Constrained`](https://esotericsoftware.com/blog/Spine-4.3-released)가 constraint 적용 뒤 값을 key로 기록해 physics 동작을 수동 baking 가능 |
| 낮은 update/render rate 반응 | 4.2 forum에서 jitter/stutter 사례 보고 | 4.3 changelog와 release note에서 smoothness/responsiveness 개선 |
| authoring FPS 기본값 | 4.2 export 값을 명시해서 사용 | 4.3 beta 과정에서 Editor 기본값이 바뀐 이력이 있으므로 기본값 의존 금지 |

4.3의 개선은 4.2의 모든 수치 결과와 bit-identical하다는 뜻이 아니다. 이 프로젝트는 4.3 Editor/Runtime으로 export·실행하고, 4.2 설명은 기능의 공개 원형과 차이를 이해하는 기준으로 사용한다.

### 13.6 Spine 4.2 Physics와 NIKKE 공개 커스텀 물리 대칭 비교

| 비교 축 | Spine 4.2 공식 Physics constraint | NIKKE 2022 발표/정정문의 커스텀 물리 | 본 프로젝트 판단 |
| --- | --- | --- | --- |
| 공개 시점 | 4.2 beta에 2023-11-11 추가, 4.2 stable 2024-04-16 | 2022 Unity Wave에서 공개 | 공개된 동일 기능일 수 없음 |
| authoring 위치 | Spine Editor의 Physics constraint | Spine 본 marking + 별도 Unity-side authoring/tool | 주요 chain은 custom asset로 authoring |
| 기본 단위 | constraint 하나가 대상 bone의 transform channel을 보정 | 실제 Spine 본과 매핑된 virtual spring bone/chain | `phys_*`와 NativeArray virtual state 분리 |
| 상태 위치 | constraint 내부 channel별 offset/velocity | Spine 렌더 본과 분리된 spring state | custom state는 Unity SoA에 유지 |
| animation 입력 | unconstrained bone world pose와 그 이동 | animation-only ghost skeleton pose를 virtual chain 입력으로 복사 | 첫 world pose에서 rest target 추출 |
| 출력 | 대상 bone world transform에 offset/mix 적용 | virtual 결과를 대응 Spine bone local pose로 복사 | main thread에서 local writeback |
| 영향 채널 | X/Y/Rotate/Scale X/Shear X 선택 | 공개 발표는 위치 기반 spring bone과 렌더 bone 혼합 중심 | chain별 rotation, 필요한 경우 x/y만 |
| 체인 표현 | constraint를 여러 bone에 배치할 수 있으나 별도 point-chain 모델은 아님 | 연속 물리 본을 chain으로 묶음 | 명시적 parent index와 solver island |
| 계층 계산 | skeleton update cache 순서에서 각 constraint 평가 | 개념은 부모→자식, 실제 Job은 physics update마다 모든 본을 동시 계산 | 우선 직렬, 이후 update당 동시 계산 A/B |
| 힘/복귀 | offset에 strength, damping, mass, wind/gravity 적용 | Hooke 복원력, damping, mass와 외력 설명 | 명시적 force function |
| inertia 의미 | bone 이동에서 유도된 unconstrained-pose 지연 offset | anchor/animation 이동에 대한 virtual state 지연 | character acceleration/impulse를 별도 입력 |
| 적분 | source가 velocity 갱신 후 offset 갱신; 공식 적분기 명칭 단정 안 함 | 발표 대상 기존판 Euler-Cromer, 발표 뒤 RK4로 교체 | fixed-step RK4를 프로젝트 결정으로 채택 |
| 시간 step | constraint별 exported FPS와 accumulator | 정확한 timestep/substep 값 미공개 | 1/60, max 4는 프로젝트 시작값 |
| render 보간 | 4.2에는 fixed step 사이 보간 없음 | 공개 자료에서 정확한 보간 방식 미확인 | 필요 시 accumulator alpha 보간을 별도 구현 |
| auxiliary spring | 기본 topology에 없음 | 주변 보조 스프링을 공개 설명 | 명시적 auxiliary edge 지원 제안 |
| 길이/각도 제한 | channel limit은 있으나 virtual segment projection 모델과 다름 | 길이 고정, stretch/motion 제한 설명 | length projection과 cone/ellipse limit 제안 |
| collision | 4.2 내장 collider 없음 | 공개 발표에서 별도 collision solver 확인 안 됨 | circle/capsule/segment는 프로젝트 제안 |
| animation 혼합 | constraint `Mix` 및 timeline | animation-only ghost와 별도 physics-only 결과를 additive/average/override | 우선 단순 physicsWeight lerp, 필요 시 모드 확장 |
| 외부 이동 입력 | PhysicsTranslate/PhysicsRotate, transform inheritance | animation/world pose를 access data에 복사 | recoil/group impulse API 추가 제안 |
| reset | `Physics.Reset`은 mix>0에서만 도달; direct `Reset()`/reset timeline은 mix==0에도 사용 가능 | 정확한 production reset 정책 미공개 | spawn/teleport/resume 규칙 명시 |
| pause/freeze | None/Pose/Reset/Update mode 조합 | 공개 자료에서 정확한 mode 계약 미확인 | state별 hard reset/sleep/freeze |
| 결정적 preview | Editor Deterministic fast-forward; runtime 결정성 보장 아님 | 동시 병렬 계산과 float/Burst의 lockstep 보장 미공개 | deterministic 요구 시 별도 플랫폼 테스트 |
| baking | 4.2 직접 key baking 없음; warm-up만 제공 | custom runtime 계산 | build-time bake는 현재 범위 밖 |
| 실행 주체 | spine runtime main update/cache | C# Job System + Burst, 값 형식 access 배열 | managed 복사 경계 + Burst job |
| 병렬화 | runtime 내부 구현에 따르며 public Job batch 계약 없음 | 한 Job에서 모든 물리 본 동시 계산, FixedUpdate마다 새 Job schedule | chain/island batching을 프로파일 후 선택 |
| 성능 근거 | custom과의 공정한 공개 benchmark 없음 | 발표자 프로젝트에서 RK4 전환 차이가 거의 없었다는 정성 보고만 있음 | 어느 쪽도 사전 승자 선언 금지 |
| Editor/runtime 일치 | built-in preview가 직접 제공됨 | 별도 Unity preview/tool 필요 | artist preview 비용을 선택 기준에 포함 |
| 적합한 범위 | Editor 완결형 장식 흔들림, 공식 constraint pipeline | gameplay input, custom topology/limits, virtual-to-render handoff | 작은 독립 장식은 built-in 허용 가능 |
| 소유권 위험 | 다른 constraint/custom write와 순서 충돌 | animation과 physics가 같은 bone을 덮을 위험 | bone+property+ancestor/subtree validator 필요 |
| 공개되지 않은 것 | 내부 제품별 튜닝이 아니라 공식 source가 공개됨 | 전체 source, 정확한 본 트리/수치/timestep/job dependency/현재 서비스 상태 | 미확인값을 NIKKE 값으로 표기 금지 |

본 프로젝트의 기본값은 주요 2차 동작에 custom Spring Rig를 쓰는 것이다. Spine Physics를 허용할 경우 `SolverOwnershipManifest`에 constraint 이름, 대상 bone, 속성, ancestor 영향 범위를 등록한다. 공식 파라미터를 아래 custom preset에 그대로 복사하지 않는다.

### 13.7 기존 명세 재검토 결과

| 판정 | 항목 | 반영 내용 |
| --- | --- | --- |
| 유지 | Spine Physics 공개 시점이 NIKKE 2022 발표보다 늦음 | 단, 공개 도입 시점을 말하는 것이며 미공개 내부 prototype 부재까지 증명하지는 않음 |
| 유지 | virtual state와 렌더 Spine bone 분리, Job/Burst, animation/physics 이중 소유 금지 | 발표와 정정문 범위로 한정 |
| 유지·한정 | 발표 뒤 RK4 교체 | 발표 슬라이드 자체가 RK4였다고 쓰지 않고, 기존 Euler-Cromer→발표 뒤 RK4로 정정 |
| 수정 | “실제 구현이 직렬 부모→자식” | 개념적 의존성과 physics update당 전체 본 동시 계산을 분리 |
| 수정 | built-in runtime input이 제한적/부적합이라는 포괄 표현 | Translate/Rotate/reset/pose API는 인정하고, arbitrary virtual topology/impulse API 부재로 범위를 좁힘 |
| 수정 | Spine 4.2와 로컬 4.3을 같은 동작으로 취급 | 4.3 interpolation/pose model 개선을 별도 명시 |
| 수정 | 단일 `physicsWeight`가 NIKKE 혼합의 재현처럼 보임 | animation-only ghost + 별도 physics-only 결과의 역사적 3모드와 프로젝트 단순안을 분리 |
| 추가 | axes, state, fixed-step, mode, reset, reference scale, no-baking, collision 경계 | §§13.5.1–13.5.5에 추가 |
| 추가 | `Physics.Pose → callback → Physics.Update` 및 transform movement forwarding | §13.5.6과 §15.2에 반영 |
| 미확인 유지 | NIKKE 정확한 timestep, job dependency/batch, 파라미터, collision, current live solver | 추정하거나 preset으로 위장하지 않음 |

---

## 14. Spring solver 상세

이 절은 공개 NIKKE 발표의 원리를 참고해 **본 프로젝트가 새로 설계하는 solver**다. 공개된 NIKKE source의 복원본이 아니다. 특히 RK4 force 식, 1/60 step, max substep, projection 순서, collision shape, preset 수치, 단순 blend 식은 모두 프로젝트 제안이며 NIKKE의 알려진 production 값이 아니다.

### 14.1 힘 모델

가상 본 끝점 위치를 `x`, 속도를 `v`, 애니메이션에서 얻은 rest target을 `r`, 질량을 `m`이라 한다.

```text
dx/dt = v
dv/dt = F(x, v, t) / m

Fspring = -k (x - r)
Fdamp   = -c v
Fgravity = m g gravityScale
Fexternal = recoil + wind + characterAcceleration
Faux = Σ[-ka (distance(x, xj) - restDistance) direction]
Ftotal = Fspring + Fdamp + Fgravity + Fexternal + Faux
```

2D skeleton이지만 Unity world transform의 scale/rotation을 반영해 계산 공간을 명시한다. 권장 계산 공간은 캐릭터 root 기준의 2D plane이며, 창 이동이나 월드 원점 변화가 물리 impulse로 오인되지 않게 `drv_character`의 렌더 이동과 물리 상속 정책을 분리한다.

### 14.2 프로젝트 튜닝 단위와 preset 시작값

stiffness와 damping을 감으로만 입력하면 캐릭터 scale이나 mass가 바뀔 때 재튜닝이 어렵다. authoring UI에는 자연 주파수 `f`(Hz)와 감쇠비 `ζ`를 노출하고 내부 계수를 다음처럼 계산한다.

```text
ω = 2πf
k = mω²
c = 2ζmω = 2ζ√(km)
```

`ζ < 1`은 흔들리며 수렴하고, `ζ = 1`은 overshoot가 거의 없는 임계 감쇠, `ζ > 1`은 느리게 복귀한다. 모든 수치는 프로젝트용 시작값이다.

| preset | f root→tip | ζ | mass root→tip | max angle | physicsWeight | 설명 |
| --- | --- | ---: | --- | --- | ---: | --- |
| `hair_soft` | 3.0→2.0 Hz | 0.45 | 1.0→0.65 | 20°→38° | 1.0 | 가벼운 긴 머리, 뚜렷한 overshoot |
| `hair_heavy` | 4.0→2.8 Hz | 0.62 | 1.4→0.9 | 16°→30° | 1.0 | 두꺼운 머리 덩어리 |
| `cloth_coat` | 3.5→2.4 Hz | 0.58 | 1.2→0.8 | 18°→34° | 0.9 | 코트/치마 패널 |
| `chest` | 5.0→4.0 Hz | 0.72 | 1.0→0.8 | 타원 limit 사용 | 0.65–0.85 | 작은 지연, 빠른 안정화 |
| `strap` | 5.5→3.5 Hz | 0.68 | 0.7→0.45 | 12°→28° | 1.0 | fixed length 강제 |
| `rigid_accessory` | 6.0→4.5 Hz | 0.78 | 1.3→1.0 | 8°→18° | 0.8 | 총기 장식, 금속 장신구 |

튜닝 순서:

1. gravity/external force/auxiliary spring을 끄고 `f`로 복귀 속도를 맞춘다.
2. `ζ`로 overshoot 횟수와 settle 시간을 맞춘다.
3. angle/length limit으로 실루엣 파손을 막는다.
4. gravity와 recoil impulse를 추가한다.
5. auxiliary spring으로 면 형태를 보강한다.
6. collision을 마지막에 추가한다. collision으로 잘못된 기본 튜닝을 숨기지 않는다.

### 14.3 RK4 한 substep

상태 `y=(x,v)`, 미분 `f(y,t)=(v,a)`에 대해:

```text
k1 = f(y,                 t)
k2 = f(y + k1 * dt/2,     t + dt/2)
k3 = f(y + k2 * dt/2,     t + dt/2)
k4 = f(y + k3 * dt,       t + dt)
y' = y + dt/6 * (k1 + 2*k2 + 2*k3 + k4)
```

각 stage에서 spring, damping, auxiliary force를 새 상태로 재평가한다. RK4라고 부르면서 force를 첫 stage 한 번만 계산해 4번 가중하는 잘못을 금지한다.

### 14.4 fixed timestep

- NIKKE의 공개 자료에는 실제 fixed timestep과 최대 substep 값이 없다. 아래는 본 프로젝트의 시작안이다.
- 시작값: `fixedDelta = 1/60 s`.
- render delta를 accumulator에 더하고 `while accumulator >= fixedDelta`로 step한다.
- 한 render frame 최대 substep 시작값: 4.
- 초과 누적 시간은 무한 catch-up하지 않는다. 최대 누적을 `4 × fixedDelta`로 clamp하고 진단 counter를 올린다.
- pause/alt-tab 뒤 큰 delta를 그대로 적분하지 않는다. resume 시 reset 또는 짧은 fast-forward 정책을 상태별로 사용한다.
- 30/60/120/144 Hz 렌더에서 동일한 60 Hz 물리 결과가 나오는지 검증한다.

### 14.5 rest target 생성

1. 첫 `UpdateWorldTransform` 후 각 `phys_*` 본의 **animation-only world pose**를 읽는다.
2. 해당 본의 setup length와 world rotation으로 endpoint를 구한다.
3. endpoint를 `restTarget`으로 기록한다.
4. anchor의 프레임 이동량/가속도를 구해 inertia 입력을 만든다.

물리 출력이 다음 프레임 rest target에 feedback되지 않게 animation-only pose를 읽는 시점이 중요하다. callback 통합 방식에서 이전 물리값이 animation apply 후 완전히 덮이는지 회귀 테스트한다.

### 14.6 프로젝트 기준 직렬 parent→child와 길이 투영

아래는 정확성을 먼저 확보하기 위한 프로젝트 기준 구현이다. 발표의 실제 Job처럼 update당 전체 본 동시 계산을 사용하지 않으며, 추후 병렬판과의 A/B 기준선으로 삼는다.

각 본을 적분한 뒤 부모 endpoint `p`에서 자식 위치 `x`까지 벡터를 구한다.

```text
d = x - p
len = |d|
targetLen = restLength
x = p + normalize(d) * clamp(len, minLen, maxLen)
```

- `fixedLength=true`: min=max=`targetLen`.
- stretch 허용: 시작값 `0.98–1.03 × targetLen`.
- 길이 투영 후 속도도 radial/tangential 성분을 재조정해 projection이 에너지를 계속 주입하지 않게 한다.
- zero-length 본은 rotation 기반 chain에 쓰지 않는다.

### 14.7 각도/이동 제한

- angular cone은 animation rest direction 기준 최소/최대 각으로 정의한다.
- chest는 타원형 motion limit과 좌우 중앙면 침범 제한을 사용한다.
- front hair는 얼굴 쪽 각도를 좁게, 바깥쪽은 넓게 허용한다.
- coat/skirt는 위쪽으로 뒤집히지 않도록 pelvis 기준 cone을 둔다.
- hard clamp만 사용하면 떨림이 생길 수 있으므로 경계 근처에는 soft-limit spring 구간을 둔다.

### 14.8 auxiliary spring

긴 머리 패널의 좌우 edge나 같은 덩어리의 병렬 chain 사이를 보조 스프링으로 연결한다.

- 목적: 체인이 고무줄처럼 과도하게 늘어지거나 서로 완전히 독립적으로 흩어지는 현상 방지.
- 보조 연결은 mesh topology를 따라 인접한 지점만 묶는다.
- 교차 연결을 과도하게 만들면 solver island가 커지고 병렬성이 줄어든다.
- 시작값은 주 spring stiffness의 15–35%로 두고 실루엣 유지 정도만 확보한다.

### 14.9 프로젝트 제안 collision

공개 발표에서는 NIKKE의 별도 collision solver를 확인할 수 없다. 아래 shape와 처리 순서는 화면 관통을 줄이기 위한 본 프로젝트 설계다.

지원 shape:

- circle: 얼굴, 어깨, 가슴의 단순 영역
- capsule: 팔, 허벅지, 총열, cover edge
- plane/segment: 화면 바닥 또는 단순 엄폐선

처리 순서:

```text
RK4 integrate
→ parent length/angle constraint
→ collision projection
→ length constraint 재투영
→ velocity correction
```

- collider는 `col_*` marker 또는 ScriptableObject local data에서 생성하고 매 프레임 source bone world pose로 갱신한다.
- collision radius는 시각 mesh보다 약간 작게 시작한다.
- continuous collision은 프로토타입 범위에서 제외하되, 큰 teleport는 reset으로 처리한다.
- self-collision은 비용 대비 효과가 낮으므로 보조 스프링과 chain ordering으로 우선 해결한다.

### 14.10 animation/physics blend

아래 단일 weight 식은 프로젝트의 첫 구현이다. §4.3에서 확인한 animation-only ghost와 별도 physics-only 결과의 additive/average/override 3모드를 그대로 구현한 것이 아니다.

최종 endpoint:

```text
final = lerp(restTarget, simulatedPosition, physicsWeight)
```

- 일반: 1.0
- reload처럼 손/총 간섭이 큰 순간: 관련 strap/hair만 0.5–0.8
- burst 연출 고정 pose: 0으로 즉시 끊지 말고 0.1–0.2 s 동안 감쇠
- defeat: 1을 유지해 관성 settle 후 freeze하거나 캐릭터 연출 데이터로 지정

weight가 0이어도 시뮬레이션을 계속할지, rest로 reset할지는 flag로 구분한다. 오랫동안 숨겨진 chain은 reset 후 sleep한다.

### 14.11 recoil impulse

실제 `ExecuteShot()`에서 다음 impulse를 Spring Rig external force에 넣는다.

```text
impulse = weaponProfile.secondaryMotionImpulse
direction = opposite muzzle direction + authored vertical component
falloff by chain group:
  chest 1.0
  front hair 0.35
  back hair 0.65
  coat/skirt 0.45
  rigid accessories 0.20
```

- shot 간 impulse는 누적 가능하지만 group별 max velocity를 둔다.
- recoil 본의 키 애니메이션과 Spring Rig impulse는 역할이 다르다. 총/팔의 의도적 반동은 animation/procedural recoil, 머리카락·천의 뒤따름은 Spring Rig다.

### 14.12 reset, teleport, visibility

다음 상황에서는 `position=restTarget`, `velocity=0`, accumulator=0으로 hard reset한다.

- skeleton 교체 또는 skin이 물리 본 구성을 바꿈
- 캐릭터 spawn/despawn
- 캐릭터 root가 한 frame에 teleport threshold를 넘음
- scale 부호가 바뀌거나 mirror 전환
- editor/domain reload
- 장시간 비활성 뒤 다시 표시

짧은 cover 전환, 일반 animation change, track mix에는 reset하지 않는다.

### 14.13 Unity에 되쓰기

각 가상 segment의 부모 위치 `p`와 최종 endpoint `x`로 world angle을 계산한다.

1. `worldAngle = atan2(x.y-p.y, x.x-p.x)`
2. Spine parent world transform의 inverse를 사용해 local rotation을 구한다.
3. 본의 setup orientation과 transform inherit mode를 반영한다.
4. output 본의 rotation 또는 필요한 local x/y만 기록한다.
5. 전체 skeleton에 두 번째 `UpdateWorldTransform`을 한 번 호출한다.

음수 scale, shear, nonuniform scale이 있는 부모는 단순 `worldAngle-parentAngle`로 처리하지 않는다. Spine runtime의 world/local 변환 API 또는 2×2 matrix inverse로 변환한다.

---

## 15. Unity 컴포넌트 구조와 업데이트 순서

### 15.1 제안 컴포넌트

| 컴포넌트 | 책임 | 금지 사항 |
| --- | --- | --- |
| `NikkeSpineCharacter` | 캐릭터 구성 root, 의존성 연결, lifecycle | 전투 규칙 직접 구현 |
| `SpineAnimationController` | AnimationState tracks, mixes, states, event relay | 실제 damage/cadence 소유 |
| `WeaponCadenceController` | bounded queue, ammo, shot clock, reload | animation frame에 shot 종속 |
| `SpineEventRelay` | EventData 캐시, typed event 발행 | callback 중 무제한 상태 재진입 |
| `SpringRigAsset` | 정적 authoring 데이터 | 런타임 상태 저장 |
| `SpringRigRuntime` | NativeArray, index mapping, accumulator | Unity object를 Burst job에 전달 |
| `SpringRigSystem` | 모든 visible rig schedule/complete | 본 이름 매 frame 탐색 |
| `SpringRigAuthoring` | scene gizmo, chain/limit/collider 편집 | build에 editor dependency 포함 |
| `SpineVisibilityController` | offscreen/sleep/LOD | gameplay 캐릭터를 임의 despawn |
| `SpineAttachmentBridge` | weapon/VFX/UI marker world transform | 매 프레임 Find 호출 |

이름의 `Nikke`는 법적/브랜드 경계를 고려해 실제 코드에서는 `IdleSpineCharacter`처럼 프로젝트 고유 명칭으로 바꾸는 것이 좋다.

### 15.2 권장 spine-unity callback 경로

spine-unity 4.3은 animation update와 renderer를 분리한다. `SkeletonAnimation`은 AnimationState를 갱신하고 world transform을 계산하며, `SkeletonRenderer.LateUpdate`가 mesh/material을 만든다. 공식 문서: [spine-unity Main Components 4.3](https://esotericsoftware.com/spine-unity-main-components-4.3).

커스텀 world-space constraint에는 다음 경로를 우선 검토한다.

```text
SkeletonAnimation.Update
  AnimationState.Update(delta)
  Skeleton.Update(delta)
  ApplyTransformMovementToPhysics            // GameObject 이동/회전을 built-in Physics에 전달
  AnimationState.Apply(skeleton)
  Skeleton.UpdateWorldTransform(Physics.Pose) // 저장된 built-in physics pose만 적용
  SkeletonRenderer.UpdateWorld callback
    read world anchors
    complete/simulate available jobs
    write local output bones
  Skeleton.UpdateWorldTransform(Physics.Update) // built-in physics 전진 + 두 번째 world 계산
SkeletonRenderer.LateUpdate
```

`UpdateWorld` 구독이 없으면 첫 `Pose` pass 없이 `Physics.Update` 한 번으로 끝난다. 구독하면 정확히 `Physics.Pose → callback → Physics.Update`가 되어 두 번 world transform을 수행한다. 따라서 callback에서 읽는 값에는 저장된 built-in Physics pose가 이미 들어 있을 수 있다. custom rest target을 animation-only로 유지하려면 같은 subtree의 built-in constraint를 배제하거나 명시적으로 pose를 분리한다. 정확한 경계가 필요하지만 비용도 있으므로 3캐릭터 기준 프로파일링한다.

### 15.3 Job scheduling 현실적 순서

같은 `UpdateWorld` callback 안에서 모든 데이터를 복사하고 job을 schedule한 뒤 즉시 complete하면 병렬 이득이 작다. 다음 두 구현은 공개 NIKKE의 정확한 schedule을 복제한 것이 아니라 본 프로젝트 후보이며, §4.2의 역사적 `Update`/`FixedUpdate` handoff와 구분한다.

**A. 단순·안전 기준 구현**

```text
UpdateWorld callback:
  copy animation world pose
  schedule SpringJob
  Complete
  write local bones
return → second world transform
```

먼저 A로 정확성을 확정한다.

**B. 중앙 시스템 batch 구현**

```text
모든 character animation pose 준비
→ SpringRigSystem이 visible rig를 한 번에 schedule
→ 렌더 전 dependency complete
→ main thread writeback
→ 각 skeleton second world transform
```

B는 spine-unity update loop를 명시적으로 제어하거나 custom update component가 필요하다. 3명뿐이므로 A보다 실제로 빠른지 Player 프로파일로 확인된 경우에만 채택한다.

### 15.4 Threaded Animation/Mesh 옵션

spine-unity 4.3에는 threaded animation/mesh generation 경로가 있지만, 캐릭터 3명에서는 scheduling/synchronization 비용이 이득보다 클 수 있다.

- 기본값: off.
- custom Spring Job이 안정화된 뒤 각각 독립적으로 on/off A/B 측정.
- Editor가 아니라 Development Player에서 `SkeletonAnimation.Update`, world transform, mesh generation, jobs wait 시간을 본다.
- GPU skinning Player 설정은 Spine CPU pose/mesh 생성을 대체하지 않는다.

---

## 16. Authoring tool 요구사항

`SpringRigAuthoring` custom editor는 다음을 제공해야 한다.

### 16.1 Scene gizmo

- anchor: 흰색 diamond
- animation rest target: 노란 점선
- simulated point: cyan circle
- 렌더 output endpoint: green line
- auxiliary spring: magenta line
- angular cone: 반투명 부채꼴
- max motion radius: 회색 원
- collider: red outline
- chain index와 solver island label

### 16.2 inspector

- skeleton bone picker; free-text 이름 입력 최소화
- 연속 descendant를 chain으로 자동 수집
- 좌우 mirror 생성 후 개별 override
- chain preset: `hair_soft`, `hair_heavy`, `cloth`, `chest`, `strap`, `rigid_accessory`
- mass/stiffness/damping 곡선을 root→tip gradient로 편집
- animation/physics split preview
- fixed timestep 30/60/120 preview
- impulse test button
- reset/teleport/slow-motion controls
- validation report

### 16.3 import/build validator

빌드를 막는 오류:

- bone mapping 누락 또는 중복
- 같은 본/속성을 custom Spring Rig와 Spine Physics가 동시에 소유
- chain에 cycle 존재
- parent index가 처리 순서를 위반
- zero/negative mass, rest length, timestep
- output bone이 anchor의 ancestor
- physics bone에 의도하지 않은 animation timeline 존재
- Editor/Runtime major.minor 불일치
- skin bone과 constraint 활성 범위 불일치

경고:

- atlas 2 page 초과
- 한 chain 6본 초과
- collider 과다
- auxiliary edge가 solver island를 과도하게 결합
- 예상 max substep 빈번 초과

---

## 17. 성능 예산과 렌더 설정

다음은 공식 NIKKE 수치가 아니라 이 3인 데스크톱 오버레이의 **프로토타입 목표 예산**이다.

### 17.1 Development Player 예산

| 항목 | 60 FPS 목표 | 실패 기준 |
| --- | ---: | ---: |
| 3개 AnimationState + world pose | 합계 ≤ 0.8 ms | 지속 ≥ 1.5 ms |
| Spring Rig schedule+complete+writeback | 합계 ≤ 0.7 ms | 지속 ≥ 1.5 ms |
| Spine mesh generation | 합계 ≤ 0.8 ms | 지속 ≥ 1.5 ms |
| Spine 관련 GC alloc | 0 B/frame | 반복 allocation 발생 |
| draw calls | 캐릭터당 1–2, 총 3–6 | 의도 없이 10 초과 |
| atlas | 캐릭터당 1 page 목표 | 3 page 이상 |
| 물리 본 | 캐릭터당 24–48 시작 | 검증 없이 80 초과 |

저사양 목표 장치에서 GPU/전체 frame budget이 우선이며, 위 숫자는 아트 복잡도에 따라 조정한다.

### 17.2 LOD

하단 상주형이라 카메라 거리는 거의 고정이므로 거리 LOD보다 visibility/focus/power LOD가 중요하다.

| 상태 | animation | spring | render |
| --- | --- | --- | --- |
| 전투 visible/focused | 60 Hz | 60 Hz | target FPS |
| visible/unfocused | 30–60 Hz | 30 Hz 가능 | 30 FPS 가능 |
| 완전 가림/minimized | gameplay clock 유지 | sleep/reset 정책 | 렌더 중지 |
| defeat freeze | 필요한 track만 | settle 후 sleep | 유지 |

물리 step을 30 Hz로 줄여도 render frame 사이 pose interpolation을 적용해 떨림을 줄인다.

### 17.3 URP/overlay 권장

- unlit, shadows off, additional lights off.
- HDR 효과가 필요하지 않으면 HDR off를 A/B 측정해 RGBA8 경로를 검토한다.
- MSAA 1은 투명 2D 캐릭터에 적합한 시작점이다.
- SRP Batcher 유지.
- framebuffer alpha와 URP alpha output을 투명 창 compositor 요구에 맞춰 end-to-end 검증한다.
- `runInBackground=true`는 게임이 비포커스 상태에서도 진행되어야 한다는 기획과 맞지만, 전력/사용자 설정 정책을 함께 둔다.
- frame pacing을 무제한으로 두지 않는다. focused/unfocused FPS cap을 명시한다.

---

## 18. Export와 Unity import 파이프라인

### 18.1 source of truth

```text
ArtSource/<character>/character.psd
SpineSource/<character>/<character>.spine
SpineExport/<character>/<character>.json or .skel
SpineExport/<character>/<character>.atlas.txt
SpineExport/<character>/<character>*.png
Unity/Assets/Characters/<character>/...
```

- `.spine`과 원본 art가 source of truth다.
- 개발 중에는 JSON export를 사용하면 diff와 검증이 쉽다.
- release에서 binary `.skel`로 바꿀 수 있지만 동일 animation test를 통과해야 한다.
- export 설정을 사람 기억에 의존하지 않고 Spine CLI preset/script로 고정한다. 공식 자료: [Spine CLI](https://esotericsoftware.com/spine-command-line-interface), [Spine Export](https://esotericsoftware.com/spine-export).

### 18.2 export gate

1. Spine Editor 4.3 확인.
2. setup pose에서 모든 attachment, skin, draw order 확인.
3. animation name/event 계약 검사.
4. animation-only preview 검사.
5. Physics constraints 사용 목록 export 및 custom ownership과 충돌 검사.
6. PMA와 atlas filter/wrap 설정 확인.
7. headless export.
8. Unity reimport.
9. `SkeletonDataAsset` 오류/누락 검사.
10. 자동 pose snapshot과 runtime smoke scene 실행.

### 18.3 asset 변경 시 호환성

- bone rename/delete는 SpringRigAsset mapping을 깨는 breaking change다.
- animation/event rename은 controller contract의 breaking change다.
- attachment만 바꾸는 skin 업데이트는 비breaking일 수 있으나 topology가 바뀌면 `Immutable Triangles` 가정을 재검증한다.
- skeleton hash/version을 asset에 기록하고 mismatch 시 명확한 오류를 낸다.

---

## 19. QA 매트릭스

### 19.1 애니메이션

- `idle → aim → fire → aim → cover`가 loop seam과 mix pop 없이 연결되는가.
- SMG/AR/MG에 10–20 input/s를 넣어도 fire loop가 매번 restart되지 않는가.
- SR/SG/RL 입력이 cadence보다 빠를 때 queue되고 recovery 중 pose가 튀지 않는가.
- reload interrupt 전/후 ammo commit이 정확한가.
- burst/hit/defeat priority가 모든 base state에서 동일한가.
- interrupt된 TrackEntry에서 Complete가 없어도 state가 stuck하지 않는가.

### 19.2 Spring Rig

- 30, 60, 120, 144 render FPS에서 동일한 settle 시간과 유사한 궤적을 보이는가.
- 200 ms/500 ms artificial stall 뒤 폭발하지 않는가.
- alt-tab, minimize, resume에서 큰 delta가 들어가지 않는가.
- spawn/teleport/skin swap 시 이전 캐릭터의 velocity가 남지 않는가.
- 음수 scale/mirror와 nonuniform scale에서 방향이 뒤집히지 않는가.
- hair가 얼굴/어깨/총/cover를 반복 관통하지 않는가.
- 직렬 기준 solver에서 부모→자식 순서를 끄면 실패하고, update당 동시 계산 solver에서는 update frequency별 계층 오차를 검출하는 회귀 test가 존재하는가.
- `UpdateWorld` callback 뒤 두 번째 world update를 끄면 실패하는 회귀 test가 존재하는가.
- physicsWeight 0↔1 전환이 snap 없이 동작하는가.
- recoil spam에서도 max velocity/angle limit을 넘지 않는가.

### 19.3 렌더/overlay

- 검정/흰색/다색 desktop 배경에서 PMA fringe가 없는가.
- 캐릭터와 HUD 밖의 픽셀이 실제 창 alpha 0인가.
- click-through 영역과 interactive HUD 영역이 분리되는가.
- DPI 100/125/150/200%, 다중 모니터, taskbar 위치 변화에 맞는가.
- 일반 PC 작업 중 키보드/마우스 입력을 가로채지 않는가.
- 비포커스에서도 기획대로 gameplay와 animation이 진행되는가.

### 19.4 프로파일링

- Unity Editor가 아닌 Development Player에서 측정.
- 60초 idle, 60초 최대 연사, 30초 burst/reload 반복.
- main thread, worker wait, GC, draw calls, overdraw, atlas memory를 기록.
- threaded animation/mesh와 custom batching을 각각 on/off 비교.
- 평균뿐 아니라 p95/p99 frame time과 stall 직후 substep clamp count를 본다.

---

## 20. 구현 단계와 완료 조건

### Phase 1: Spine vertical slice

- 오리지널 테스트 캐릭터 1명, `idle/aim/fire/cover/reload/hit/defeat` 제작.
- 필수 본/slot/marker/event 계약 적용.
- Unity 4.3 import와 AnimationState track controller 구현.
- 연사 입력에서 fire loop가 재시작되지 않음.

완료 조건: Spring Rig 없이도 모든 gameplay state가 끊김 없이 동작한다.

### Phase 2: 단일 Spring chain

- back hair 1 chain, anchor + virtual state + RK4.
- animation-only rest target, local writeback, second world update.
- fixed step, reset, physicsWeight.

완료 조건: 30/60/120 FPS와 animation 전환에서 안정적이며 GC 0 B/frame.

### Phase 3: full character Spring Rig

- hair, chest, coat/skirt, strap.
- auxiliary springs, angle/length limit, circle/capsule collision.
- recoil impulse와 authoring gizmo.

완료 조건: 대표 combat sequence에서 관통/폭발/snap이 없고 아트 검수를 통과한다.

### Phase 4: Job/Burst와 3인 통합

- managed/NativeArray 경계 확정.
- chain island batching, Burst compile.
- 3개 캐릭터 동시 최대 연사 Player 프로파일.

완료 조건: §17 목표 예산 또는 합의된 수정 예산을 충족한다.

### Phase 5: overlay production hardening

- transparent window, click-through, background update, multi-monitor/DPI.
- minimize/resume/power LOD.
- build-time export/validation.

완료 조건: §19 전체 QA 매트릭스를 통과하고 법적 배포 검토가 완료된다.

---

## 21. 다음 구현 에이전트용 체크리스트

### 시작 전

- [ ] [GAME_DESIGN.md](./GAME_DESIGN.md)의 attack queue와 자동전투 규칙을 읽는다.
- [ ] Unity `6000.3.19f1`과 잠긴 Spine 패키지 commit을 유지한다.
- [ ] Spine Editor 4.3 라이선스와 major.minor를 확인한다.
- [ ] 사용할 아트가 오리지널이며 추출 NIKKE 에셋이 아님을 확인한다.

### Spine 제작

- [ ] §6 이름과 소유권에 맞게 최소 skeleton을 만든다.
- [ ] `drv_*`와 `phys_*`를 분리한다.
- [ ] aim/grip/muzzle/shell/magazine marker를 만든다.
- [ ] §9 clip과 §12 event를 구현한다.
- [ ] 물리 없는 상태에서 animation-only 결과를 승인받는다.

### Unity

- [ ] `SkeletonAnimation + SkeletonRenderer`로 import한다.
- [ ] base/fire/reaction/face/burst track을 분리한다.
- [ ] 입력 queue와 WeaponCadenceController를 animation에서 분리한다.
- [ ] 같은 fire loop에 `SetAnimation`을 반복 호출하지 않는다.
- [ ] custom Spring Rig의 update 순서와 두 번째 world transform을 보장한다.
- [ ] main thread는 Spine API, Burst job은 blittable NativeArray만 다룬다.
- [ ] 같은 본에 custom solver와 Spine Physics를 중복 적용하지 않는다.

### 검증

- [ ] 30/60/120/144 FPS, stall, alt-tab, minimize/resume.
- [ ] 연속 입력과 queue overflow/expiry.
- [ ] reload/burst/hit/defeat interrupt.
- [ ] teleport/skin swap/mirror/reset.
- [ ] Player profiler, GC 0, draw calls/atlas.
- [ ] 투명 창 alpha와 click-through를 별도 subsystem에서 검증.

---

## 22. 법적·라이선스 경계

이 절은 법률 자문이 아니다. 배포 전 권리자 정책과 관할 법률을 전문가와 다시 확인해야 한다.

- 공개된 기술 발표에서 원리를 학습해 오리지널 코드와 리그를 만드는 것과, 게임에서 추출한 아트/Spine 데이터를 복제·재배포하는 것은 다르다.
- 이 프로젝트에는 NIKKE의 추출 `.skel`, `.atlas`, texture, audio, 캐릭터 데이터, 코드, 정확한 리그 복제물을 포함하지 않는다.
- 프로젝트명, 캐릭터, 아트, 적, UI는 배포 단계에서 오리지널 IP로 치환하는 것이 안전하다.
- 공개된 2차 창작 가이드라인은 팬아트/영상 허용이 곧 플레이 가능한 팬게임 서비스 배포 허용을 뜻하지 않는다. 현 기획의 “NIKKE 팬게임” 배포는 별도 허가 없이는 고위험으로 취급한다.
- Spine Runtime을 제품에 통합·배포하려면 유효한 Spine Editor 라이선스와 배포 요건을 지켜야 한다. 2025-04-05 갱신된 공식 문서는 통합 시 유효한 라이선스, 제품 문서에 Runtimes License 포함 등의 조건을 둔다. 출처: [Spine Editor License Agreement](https://esotericsoftware.com/spine-editor-license), [Spine Runtimes License](https://esotericsoftware.com/spine-runtimes-license).

배포 gate:

- [ ] 모든 시각/음향/캐릭터 IP 권리 확인
- [ ] Spine Editor 라이선스 사용자와 통합 시점 기록
- [ ] 배포물 문서에 요구되는 Spine Runtimes license/copyright 포함
- [ ] 팬게임/상표/게임 서비스 관련 서면 허가 또는 법률 검토

---

## 23. 출처와 접근일

모든 웹 출처 접근일: 2026-07-14.

### 1차 또는 공식 자료

- [Unity Korea, Unity Wave 2022: 승리의 여신 NIKKE Spine 캐릭터 헤어&신체 물리 구현](https://www.youtube.com/watch?v=-zyp6qNTms0)
- [발표자 이재민의 적분기·Job/Burst 정정문](https://ijemin.com/blog/%EC%9C%A0%EB%8B%88%ED%8B%B0-%EC%9B%A8%EC%9D%B4%EB%B8%8C-%EC%8A%B9%EB%A6%AC%EC%9D%98-%EC%97%AC%EC%8B%A0-%EB%8B%88%EC%BC%80-%EC%8A%A4%ED%8C%8C%EC%9D%B8-%EB%AC%BC%EB%A6%AC-%EA%B5%AC%ED%98%84-%EB%B0%9C/)
- [NDC22 SHIFT UP 디렉터 발표 영상](https://www.youtube.com/watch?v=Bn_x75Wm9p8)
- [Spine Physics Constraints](https://esotericsoftware.com/spine-physics-constraints)
- [Spine changelog archive](https://esotericsoftware.com/spine-changelog/archive): 4.2 beta 기능 추가일, 4.2 stable, 4.3 physics 개선 이력
- [Spine IK Constraints](https://esotericsoftware.com/spine-ik-constraints)
- [Spine weighted mesh Professional example](https://esotericsoftware.com/spine-examples-powerup)
- [Spine 4.2: The Physics Revolution](https://esotericsoftware.com/blog/Spine-4.2-The-physics-revolution)
- [Spine 4.2 release notice](https://esotericsoftware.com/forum/d/26069-spine-42-has-been-released)
- [Spine forum: Physics jitters/unstable at higher framerates](https://esotericsoftware.com/forum/d/26512-physics-jittersunstable-at-higher-framerates): 4.2 fixed-step accumulator와 step 사이 interpolation 부재 설명
- [Spine 4.3 released](https://esotericsoftware.com/blog/Spine-4.3-released): 낮은 update/render rate의 physics smoothness·responsiveness 및 pose model 개선
- [spine-unity Main Components 4.3](https://esotericsoftware.com/spine-unity-main-components-4.3)
- [Spine Runtime Architecture](https://esotericsoftware.com/spine-runtime-architecture)
- [Spine Versioning](https://esotericsoftware.com/spine-versioning)
- [Spine CLI](https://esotericsoftware.com/spine-command-line-interface)
- [Spine Export](https://esotericsoftware.com/spine-export)
- [spine-unity installation/Unity compatibility](https://esotericsoftware.com/spine-unity-installation)
- [Spine Editor License Agreement](https://esotericsoftware.com/spine-editor-license)
- [Spine Runtimes License Agreement](https://esotericsoftware.com/spine-runtimes-license)

### SHA 고정 공식 runtime source

Spine 4.2 분석 기준 commit: `b81e5a58ed38704aee4f866f0e0ac672623ce914`.

- [4.2 PhysicsConstraint.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/b81e5a58ed38704aee4f866f0e0ac672623ce914/spine-csharp/src/PhysicsConstraint.cs): 동적 state, fixed-step, force/update/apply, reset, translate/rotate
- [4.2 Skeleton.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/b81e5a58ed38704aee4f866f0e0ac672623ce914/spine-csharp/src/Skeleton.cs): `Physics.None/Reset/Update/Pose`, physics forwarding API
- [4.2 SkeletonJson.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/b81e5a58ed38704aee4f866f0e0ac672623ce914/spine-csharp/src/SkeletonJson.cs): Physics constraint export data와 FPS fallback parsing
- [4.2 Animation.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/b81e5a58ed38704aee4f866f0e0ac672623ce914/spine-csharp/src/Animation.cs): physics parameter/global/reset timeline runtime
- [4.2 spine-unity SkeletonAnimation.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/b81e5a58ed38704aee4f866f0e0ac672623ce914/spine-unity/Assets/Spine/Runtime/spine-unity/Components/SkeletonAnimation.cs): animation apply와 `Pose → callback → Update` world pass

현재 프로젝트 4.3 분석 기준 locked commit: `8a238d76a808b325ba41824b438048677c414e14`.

- [4.3 PhysicsConstraint.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/8a238d76a808b325ba41824b438048677c414e14/spine-csharp/src/PhysicsConstraint.cs): lag interpolation과 pose/state 구조
- [4.3 Skeleton.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/8a238d76a808b325ba41824b438048677c414e14/spine-csharp/src/Skeleton.cs): current Physics mode/API
- [4.3 SkeletonAnimationBase.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/8a238d76a808b325ba41824b438048677c414e14/spine-unity/Assets/Spine/Runtime/spine-unity/Components/Base/SkeletonAnimationBase.cs): transform movement 적용 호출과 animation apply 경계
- [4.3 SkeletonRenderer.Common.cs](https://github.com/EsotericSoftware/spine-runtimes/blob/8a238d76a808b325ba41824b438048677c414e14/spine-unity/Assets/Spine/Runtime/spine-unity/Components/SkeletonRenderer.Common.cs): `PhysicsTranslate/Rotate` forwarding과 `Pose → callback → Update` 구현

### 보조 자료

- [Inven NDC22 발표 정리](https://www.inven.co.kr/webzine/news/?news=272724)
- [GameMeca NDC22 관련 보도](https://www.gamemeca.com/view.php?gid=1681237)
- [SHIFT UP 채용 페이지](https://shiftup.co.kr/recruit/recruit.php?searchkey=Spine&category=0): Unity/C# 및 Spine 관련 인력 수요의 보조 근거일 뿐, 출시 코드 구조의 증거로 사용하지 않음
- [fatboytao/NikkeSpine](https://github.com/fatboytao/NikkeSpine): 공개 제3자 뷰어의 bundle/animation label 관찰에만 사용; 에셋을 내려받거나 재배포하지 않음

### 로컬 자료

- [GAME_DESIGN.md](./GAME_DESIGN.md)
- [Packages/manifest.json](./Packages/manifest.json)
- [Packages/packages-lock.json](./Packages/packages-lock.json)
- [ProjectSettings/ProjectVersion.txt](./ProjectSettings/ProjectVersion.txt)
- [ProjectSettings/ProjectSettings.asset](./ProjectSettings/ProjectSettings.asset)

---

## 24. 남은 의사결정

구현을 막지 않지만 vertical slice 뒤 확정해야 한다.

1. 대표 캐릭터와 첫 무기(SMG/AR 권장).
2. 오리지널 아트의 최종 후면 3/4 포즈와 화면 점유율.
3. 총을 2D attachment로 할지 별도 3D renderer로 할지.
4. 캐릭터별 실제 hair/coat/accessory chain 수.
5. focus를 잃었을 때 animation/render FPS와 전력 정책.
6. burst가 전신 Spine animation인지 별도 cut-in renderer인지.
7. built-in Spine Physics를 완전히 금지할지, custom solver와 분리된 소형 장식에 허용할지.
8. 팬게임 prototype을 외부 배포할지, 오리지널 IP로 전환할지.

첫 구현은 결정을 기다리지 않고 다음 최소값으로 시작할 수 있다: 오리지널 테스트 캐릭터 1명, AR, 2D 총, back-hair 3 chain, chest 2 chain, coat 2 chain, custom Spring Rig only, 60 Hz fixed RK4.
