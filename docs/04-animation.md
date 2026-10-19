# 애니메이션 통합 — `PlayerAnimationController`

[← README](../README.md)

예측·리컨사일 환경에 캐릭터 애니메이션을 붙인 작업입니다. 원칙은 하나입니다.

> **이동 상태는 각 머신이 예측 결과에서 직접 만들고, 서버가 주도하는 이벤트만 네트워크로 보낸다.**

`PlayerAnimationController`는 팀원과 함께 고친 파일입니다. 이 문서는 제가 작성한 부분만 다루고, 팀원이 만든 구조를 쓴 곳은 따로 밝혔습니다. PR 번호는 비공개 팀 저장소 기준입니다.

---

## 1. 왜 애니메이션을 동기화하지 않는가

플레이어 이동은 `PlayerController.ReplicateMove`에서 계산되고, 이 메서드는 소유자 예측·리플레이·서버·관측자 모든 곳에서 실행됩니다. 즉 각 머신은 이미 같은 이동 결과를 갖고 있습니다. 여기에 애니메이터 상태를 따로 동기화하면 같은 정보를 두 번 보내는 셈이고, 트리거는 두 번 재생됩니다.

| 신호 | 경로 | 이유 | PR |
| --- | --- | --- | --- |
| 속도, 접지 | `UpdateLocomotion` — `ReplicateMove`에서 모든 머신이 호출 | 리플레이 중 여러 번 불려도 마지막 틱 값으로 수렴하는 값만 씀 | #14 |
| 점프 | `PlayJump` — 로컬 Animator 트리거 | 점프 입력은 예측으로 이미 모든 머신에 전달됨. 트리거는 재생을 다시 시작시키므로 **호출부가 리플레이 재발동을 막음** | #14, 호출부 가드 #78 |
| 서버가 판단하는 이벤트 | `ServerPlayTrigger` → `ObserversRpc` | 서버만 아는 결과는 한 번 전달 | #14, #53 |
| 역할 전환 | `ApplyRole` | 술래·도망자·유령 리그를 켜고 끄고 Animator를 다시 바인딩 | #47 |

---

## 2. 역할별 리그

플레이어 프리팹 하나에 역할별 캐릭터 모델을 모두 두고, 현재 역할의 리그만 켭니다. 역할이 바뀌어도 `NetworkObject`를 다시 스폰하지 않으므로 소유권·카메라·이름표 연결이 유지됩니다. 스켈레톤이 없는 모델(정적 유령 메시)은 Animator를 비워 두고, 모든 재생 메서드가 이 경우를 그냥 건너뜁니다 (PR #47).

역할 모델들은 하나의 표시 전용 부모 아래에 있습니다. 이 구조는 팀원이 잡기 슬로우 연출을 위해 만들었습니다. 이 부모만 움직이면 예측·충돌 상태를 건드리지 않고 표시 위치를 보정할 수 있어서, [맵 장애물의 원격 탑승자 보정](01-map-obstacle.md#59-연출은-시뮬레이션과-분리)도 여기에 `SetPlatformOffset`을 추가해 같은 경로로 씁니다 (PR #88).

---

## 3. 에셋 파이프라인

캐릭터 모델의 클립을 Animator Controller + Override Controller 구조에 연결하면서 처리한 일입니다.

| 문제 | 처리 |
| --- | --- |
| 모델에서 가져온 클립을 컨트롤러에 연결하기 전에 이름을 정리해야 함 | 선택한 모델의 클립 이름을 파일 이름으로 일괄 변경하는 에디터 도구 `AnimationClipRenamer` 작성. 이름을 바꾸면 클립의 파일 ID가 바뀌어 기존 참조가 깨지므로, 컨트롤러에 연결하기 전에 실행하도록 정함 (PR #17) |
| 술래가 공격한 뒤 상체가 T자 포즈로 고정됨 | 상체 레이어에 Locomotion 블렌드 트리 상태를 추가하고, 달릴 때도 상체는 Walk를 재생하게 함. 코드가 아니라 Animator Controller 수정 (PR #65) |

---

## 4. 트러블슈팅

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| 호스트에서 잡힌 도망자 모델이 디스폰 직전 한순간 다시 보임 | 팀원이 만든 "예측 잡기 시 도망자 렌더러 숨김"이 `OnDisable`에서 숨김을 풀었는데, 호스트에서는 서버가 플레이어를 비활성화한 **뒤** 틱 스무더가 분리해 둔 그래픽을 다시 붙임 | 복원 시점을 풀에서 재활성화되는 `OnEnable`로 이동 (PR #98) |

```csharp
// PlayerAnimationController — PR #98 시점. OnEnable과 두 주석이 제 수정이고, 숨김·복원 로직은 팀원 코드입니다.
// Restores renderers a despawn left hidden, before the pooled player can be seen again.
private void OnEnable()
{
    RestorePredictedCatchRenderers();
}

// Hidden renderers stay hidden until OnEnable. On a host the server disables this player before the
// tick smoother re-attaches its detached graphics, so restoring here would flash the caught model.
private void OnDisable()
{
    FinishCatchSlow();
    _hasCatchSlowAttack = false;
    _predictedCatchHides.Clear();
}
```
