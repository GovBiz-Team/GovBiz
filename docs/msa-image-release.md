# MSA 이미지 발행과 GitOps 연결 경계

## 현재 구현 범위

새 Kubernetes 배포 이미지 저장소는 **GitHub Container Registry(GHCR)**다. 기존 EC2/SSM용 ECR
설정은 역사적 재배포 경로로 남기며 새 워크플로에서 AWS API를 사용하지 않는다.
`develop`의 같은 SHA에서 GovBiz CI·Catalog separation CI·GovBiz Ops CI가 모두 통과하면
`.github/workflows/msa-images.yml`이 네 서비스의 이미지 후보를 발행한다.
`MSA_RELEASE_ENABLED`가 정확히 `true`일 때만 활성화된다.

흐름은 `develop push → 세 CI 성공 → 서비스별 Git archive → GHCR → digest receipt`다.
이후 `검토한 receipt → GovBiz-infra의 대상 values digest 변경 → Git 반영 → Argo CD`가 필요하다.
현재 대상 환경·저장소 간 자동 쓰기·상시 클러스터는 연결하지 않았다.
**이미지 발행 CI와 운영 자동 배포 완료는 서로 다르다.**

| 서비스 | 이미지 저장소 |
| --- | --- |
| Core | `ghcr.io/govbiz-team/govbiz-core-service` |
| Catalog | `ghcr.io/govbiz-team/govbiz-catalog-service` |
| AI | `ghcr.io/govbiz-team/govbiz-ai-service` |
| Ops | `ghcr.io/govbiz-team/govbiz-ops-service` |

## 실제 발행 확인 — 2026-09-20 KST

- 소스 기준점: `c27e2b579a14c2b87eb0974cd3f176b195ad8d6b`.
- [GovBiz CI](https://github.com/GovBiz-Team/GovBiz/actions/runs/35457058416),
  [Catalog CI](https://github.com/GovBiz-Team/GovBiz/actions/runs/35457058337),
  [Ops CI](https://github.com/GovBiz-Team/GovBiz/actions/runs/35457058421)가 모두 성공했다.
- [최초 발행 run](https://github.com/GovBiz-Team/GovBiz/actions/runs/35457860821)의 네 publish job이
  모두 성공했고 각 `msa-image-<service>` artifact에 선택할 digest가 기록되어 있다.
- 최초 발행은 `MSA_RELEASE_ENABLED=true`, `msa-release` Environment의 `develop` 전용 branch policy로 실행했다.
- 최초 공개 발행 때 익명 manifest의 SHA-256 일치 및 모든 레이어의 HEAD 200을 확인했다.
  이후 사용자의 공개 결정 철회에 따라 **네 패키지를 삭제·재발행 없이 Private으로 전환**했고,
  새로운 익명 접근 요청은 네 패키지 모두 HTTP 401로 거절되는 것을 확인했다.
- 현재 `MSA_RELEASE_ENABLED=false`로 자동 발행을 중지했으며 조직의 공개 패키지 생성 허용도 껐다.
  기존 이미지·digest·발행 이력과 develop 전용 Environment 정책은 유지했다.
- [패키지 목록](https://github.com/orgs/GovBiz-Team/packages?repo_name=GovBiz)은 권한 있는 계정으로 확인한다.
  전체 레이어 다운로드·비공개 전환 후 인증된 pull·상시 Kubernetes 배포는 아직 검증하지 않았다.
  이전에 다운로드한 외부 사본을 비공개 전환으로 회수할 수는 없다. 소스 저장소 공개 범위는 바꾸지 않았다.

## 발행 권한과 공개 범위

- Actions의 `GITHUB_TOKEN`에 `contents: read`, `actions: read`, 발행 job에만 `packages: write`를 준다.
  별도 장기 PAT나 AWS 액세스 키·OIDC 역할은 필요 없다.
- `msa-release` Environment는 develop만 허용하도록 설정한다. 최초 발행 시 정책과 저장소 연결을 확인한다.
- 현재 중지된 Repository Variable을 승인 후 `MSA_RELEASE_ENABLED=true`로 설정하면 후속 성공 CI에서 자동 발행된다.
  이미 테스트가 완료된 SHA는 Actions의 **MSA image candidates → Run workflow → develop**으로 재시도한다.
- 조직에서 Actions의 패키지 생성 권한이 허용되어야 한다. 기존 동일 이름 패키지가 있으면
  GovBiz 저장소의 Actions 접근 권한을 확인한다. 권한 오류를 우회하기 위해 광범위 PAT를 추가하지 않는다.
- 현재 사용자 선택은 **비공개 이미지**다. 기존 네 패키지는 Private을 유지하고 조직의 공개 패키지
  생성 허용을 켜지 않는다. 소스 저장소의 공개 범위와 이미지 패키지의 공개 범위는 별개다.
  워크플로는 패키지 공개 설정을 임의 변경하지 않는다.

이미지에는 컴파일된 코드·Python 소스·의존성이 들어간다. 비공개여도 비밀값은 런타임 Secret으로만 주입한다.
Kubernetes에는 별도로 `read:packages` 권한의 pull 인증이 필요하다. 해당 namespace의 Secret과
Helm `imagePullSecrets` 참조를 준비하고 인증된 pull을 검증해야 한다. 아직 상시 클러스터 인증은 연결하지 않았다.
Actions의 단기 GITHUB_TOKEN을 상시 클러스터 비밀값으로 복사하지 않는다.

## 재실행·부분 실패·오래된 커밋

- PR·fork·다른 브랜치·실패/누락 CI는 발행하지 않는다. 동일 SHA의 최신 실행/시도를 검사한다.
- 다른 커밋이 develop에 올라왔으면 이전 후보 발행을 중지한다. 빌드 전·업로드 전·receipt 저장 전에 검사한다.
- 서비스 디렉터리 tree, 발행 도구 디렉터리 tree, `linux/amd64`로 입력 키를 만든다.
  `src-<64자리 입력 키>`가 있으면 실제 digest의 source·입력 키 label·플랫폼을 확인해 재사용한다.
  다른 서비스의 소스가 바뀌지 않으면 해당 서비스는 재빌드하지 않는다.
- 서비스 README·테스트 변경도 보수적으로 해당 서비스 재빌드를 유발한다. 발행 도구 변경은 모두 재빌드한다.
- GHCR 태그 자체는 mutable이다. 이 CI는 기존 태그를 덮어쓰지 않지만 registry가 불변 태그를 강제한다고
  주장하지 않는다. 배포는 항상 `repository@sha256:...` digest로 고정한다.
- 외부 base tag 업데이트만으로 재빌드되지는 않는다. 보안 재빌드는 추적 중인 Dockerfile의
  base 버전/digest 등 빌드 입력을 명시적으로 갱신한다.
- `git archive`로 추적 파일만 빌드한다. 호스트 `.env`·캐시·미추적 파일은 보내지 않는다.
  추적 파일에 이미 들어간 비밀값은 이 도구가 정화하는 것이 아니므로 커밋 전 검토가 필요하다.
- 로그인 토큰은 stdin으로만 전달하고 임시 Docker 설정을 사용한다. 기존 사용자 로그인을 변경하지 않는다.
- 병렬 발행 일부 실패 시 일부 이미지만 남을 수 있다. 자동 삭제/배포하지 않는다.
  동일 소스 재실행은 검증된 기존 이미지를 재사용한다. 인증·네트워크 오류를 이미지 없음으로 숨기지 않는다.

## 결과와 버전 선택

성공한 전체 run의 `msa-image-<service>` artifact에는 JSON receipt가 있다.
이미지 식별자·digest·입력 키·검증 소스 SHA만 저장하며 인증 비밀값은 없다.
`verifiedRevision`은 테스트한 소스 기준점이다. 재사용 이미지의 OCI revision label은 같은 service tree를
최초 빌드한 이전 SHA일 수 있으므로 두 값을 같은 의미로 보고하지 않는다.

receipt는 **서명된 provenance가 아니다**. 정확한 저장소·성공한 전체 workflow run·소스 SHA의 artifact인지
확인한다. 임의 JSON을 검증 성공의 증거로 사용하지 않는다.
GovBiz-infra의 `scripts/promote_image.py`는 검토한 receipt로 기존 비로컬 values의 digest만 바꾸는 도구다.
기본은 diff 미리보기이며 stale digest·registry 변경·local fixture 변경을 거절한다.

실제 대상 values, DB·Secret·TLS·Ingress, Argo Application과 자동 동기화 정책은 환경별로 별도 확정한다.
GovBiz CI에 infra 쓰기 PAT를 임의 등록하지 않는다. 로컬 Git 파일 수정이나 이미지 upload만으로
실제 Kubernetes 새 이미지 배포를 완료했다고 표시하지 않는다.

## 검증

```bash
python3 -B -m unittest discover -s infrastructure/release -p 'test_*.py'
```

정확한 SHA·CI gate, 기존 이미지 메타데이터, 오류 거절, Git archive 비밀 파일 제외,
실패 시 logout·receipt 미생성과 선택 digest를 검증한다. CI에도 같은 테스트를 연결했다.
Dockerfile·업무 코드·스키마·로컬 실행 환경은 바꾸지 않았다.
오프라인 테스트는 실제 GHCR upload·익명 pull·Kubernetes 배포 성공을 대신하지 않는다.

공식 근거: [workflow_run 보안과 트리거](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#workflow_run),
[GHCR 인증·기본 공개 범위·digest pull](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry),
[Docker 원격 이미지 메타데이터](https://docs.docker.com/reference/cli/docker/buildx/imagetools/inspect/).
