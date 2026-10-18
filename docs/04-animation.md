<!-- 게시 전 확인: PlayerAnimationController는 팀원과 공동 작업 파일. 아래 각 항목이 본인 커밋인지 git log -L로 확인 후 남길 것. §3 PR #65의 해결 방식(상체 레이어 가중치 코드 제어)도 커밋으로 확인할 것. -->

# 애니메이션 통합 — `PlayerAnimationController`

[← README](../README.md)

예측·리컨사일 환경에 캐릭터 애니메이션을 붙인 작업입니다. 원칙은 하나입니다.

> **이동 상태는 각 머신이 예측 결과에서 직접 만들고, 서버가 주도하는 이벤트만 네트워크로 보낸다.**

---

## 1. 왜 애니메이션을 동기화하지 않는가

플레이어 이동은 `PlayerController.ReplicateMove`에서 계산되고, 이 메서드는 소유자 예측·리플레이·서버·관측자 모든 곳에서 실행됩니다. 즉 각 머신은 이미 같은 이동 결과를 갖고 있습니다. 여기에 애니메이터 상태를 따로 동기화하면 같은 정보를 두 번 보내는 셈이고, 트리거는 두 번 재생됩니다.

| 신호 | 경로 | 이유 |
| --- | --- | --- |
| 속도, 접지 | `UpdateLocomotion` — `ReplicateMove`에서 모든 머신이 호출 | 리플레이 중 여러 번 불려도 마지막 틱 값으로 수렴하는 값만 씀 |
| 점프 | `PlayJump` — 로컬 Animator 트리거 | 점프 입력은 예측으로 이미 모든 머신에 전달됨. 트리거는 재생을 다시 시작시키므로 **호출부가 리플레이 재발동을 막음** |
| 서버가 판단하는 이벤트 | `ServerPlayTrigger` → `ObserversRpc` | 서버만 아는 결과는 한 번 전달 |
| 역할 전환 | `ApplyRole` | 술래·도망자·유령 리그를 켜고 끄고 Animator를 다시 바인딩 |

---

## 2. 역할별 리그

플레이어 프리팹 하나에 역할별 캐릭터 모델을 모두 두고, 현재 역할의 리그만 켭니다. 역할이 바뀌어도 `NetworkObject`를 다시 스폰하지 않으므로 소유권·카메라·이름표 연결이 유지됩니다. 스켈레톤이 없는 모델(정적 유령 메시)은 Animator를 비워 두고, 모든 재생 메서드가 이 경우를 그냥 건너뜁니다.

모든 역할 모델은 하나의 표시 전용 부모 아래에 둡니다. 이 부모만 움직이면 예측·충돌 상태를 건드리지 않고 표시 위치를 보정할 수 있고, [맵 장애물의 원격 탑승자 보정](01-map-obstacle.md#59-연출은-시뮬레이션과-분리)이 이 지점(`SetPlatformOffset`)을 씁니다.

---

## 3. 에셋 파이프라인

아트팀이 Meshy AI로 만든 FBX를 Animator Controller + Override Controller 구조에 연결하면서 생긴 문제들을 처리했습니다.

| 문제 | 처리 |
| --- | --- |
| 여러 FBX에서 가져온 클립 이름이 모두 같아(`metarig\|metarig\|Scene`) Override Controller에서 구분 불가 | 클립 이름을 일괄 변경하는 에디터 도구 `AnimationClipRenamer` 작성 |
| 폭탄·배트 같은 프롭이 캐릭터 메시에 합쳐져 본 스케일이 누적됨 | 프롭을 캐릭터와 분리하는 방향으로 결정 |
| 술래 상체 레이어가 빈 상태에서 T자 포즈가 됨 | 상체 레이어 가중치를 코드에서 제어 (PR #65) |

---

## 4. 트러블슈팅

| 문제 | 원인 | 해결 |
| --- | --- | --- |
| 호스트에서 잡힌 도망자 모델이 디스폰 직전 한순간 다시 보임 | 숨긴 렌더러를 `OnDisable`에서 복원했는데, 호스트에서는 서버가 플레이어를 비활성화한 **뒤** 틱 스무더가 분리해 둔 그래픽을 다시 붙임 | 복원 시점을 풀에서 재활성화되는 `OnEnable`로 이동 |

```csharp
// PlayerAnimationController — 복원 시점
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
