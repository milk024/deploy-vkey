# deploy-vkey

> 포트폴리오 공개용 스냅샷입니다. 실제 IP·도메인·조직명은 예시값(`10.0.0.x`, `example.ac.kr`, `myorg`)으로 바꿨고 비밀값은 포함하지 않습니다. 전체 소개는 [k8s-manager](https://github.com/milk024/k8s-manager) 를 보세요.

`vkey-system` 네임스페이스 서비스의 **배포 상태 기준 저장소**. ArgoCD 가 이 저장소를 읽어 클러스터에 동기화한다.

| 앱 | 경로 | 소스 저장소 | 네임스페이스 |
|---|---|---|---|
| vkey-server | `apps/vkey-server` | `myorg/vkey-server` | vkey-system |
| vkey-web | `apps/vkey-web` | `myorg/vkey-web` | vkey-system |

## 흐름

```
소스 저장소 main push → CI 빌드·이미지 push → 이 저장소 apps/<앱>/values.yaml 의 image.tag 갱신 커밋(vkey-ci)
                                             → ArgoCD 가 감지해 동기화
```

## 규칙

- **main 직접 push 는 CI 계정 `vkey-ci`(image.tag 갱신)만.** 그 외 변경은 PR + `vkey-maintainers` 1명 승인.
- **비밀값을 커밋하지 않는다.** 아래는 클러스터에서 별도 관리하며 ArgoCD 대상이 아니다.
  - Secret `vkey-app-secret`, `vkey-postgres-secret`, `vkey-registry-secret`
  - `vkey-postgres`, `vkey-pgadmin`, PVC(`vkey-logs-pvc`), `vkey-gateway`, `vkey-http-to-https-redirect`
- 클러스터에서 `kubectl edit` / `helm upgrade` 로 직접 바꾸지 않는다. ArgoCD 가 이 저장소 기준으로 되돌린다.

## vkey-app-secret 을 바꿀 때

예전에는 vkey-server 저장소의 `k8s/vkey-app-secret.yaml` 을 CI 가 적용했지만, 이제 **Secret 은 저장소에 두지 않는다.**

1. 클러스터에서 Secret 갱신 (권한 있는 사람: 인프라 담당)
   ```bash
   kubectl apply -n vkey-system -f vkey-app-secret.yaml   # 파일은 저장소 밖에 보관
   ```
2. 이 저장소 `apps/vkey-server/values.yaml` 의 `appSecretRevision` 값을 바꿔 PR → 병합되면 vkey-server Pod 가 새 값으로 재시작

## ArgoCD

- `argocd/project-vkey.yaml` — AppProject `vkey` (이 저장소만 읽고 `vkey-system` 에만 배포, 클러스터 범위·RBAC 리소스 금지)
- `argocd/app-*.yaml` — Application, 자동 동기화(prune + selfHeal)
- 접속: `kubectl port-forward -n argocd svc/argocd-server 8080:443` → https://localhost:8080

## 이력

- 2026-11-12: 소스 저장소 `k8s/` 차트 + 실제 클러스터 값(helm release)에서 이전. 이전 시점 `kubectl diff` = 0.
  vkey-server 의 Secret 파일 해시 어노테이션은 `appSecretRevision` 값으로 대체(처음 값 = 기존 해시, 재시작 없음).
