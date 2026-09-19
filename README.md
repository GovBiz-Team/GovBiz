# 🏛️ GovBiz

**LLM 기반 정부지원사업 탐색·신청 관리 플랫폼**

이 저장소는 React·Core API·Catalog Service·AI Service와 Django 기반 Ops를 함께 관리하는 **애플리케이션 모노레포**입니다.
Ops 소스는 [`backend/ops-service`](backend/ops-service)에 있으며, 상태 확인 API·전용 MySQL·Gunicorn 실행 이미지를 갖추고 있습니다.
소스를 통합해도 서비스 프로세스·의존성·DB 책임은 분리합니다.

사용자 클라이언트는 [`frontend/`](frontend/README.md) 아래의 웹(`frontend/web/`)과 React Native 앱(`frontend/mobile/`)으로 나눠 관리합니다.
[모바일 실행·기능 안내](frontend/mobile/README.md) · [공통 코드·workspace 관리](docs/mobile-monorepo.md)

전체 로컬 실행은 루트 `compose.yaml`을 사용합니다. Core·Catalog·AI·Ops를 포함하며,
Core는 Catalog의 내부 HTTP snapshot을 읽습니다. `.env`에 32자 이상의 `CATALOG_INTERNAL_TOKEN`을
설정해야 합니다. 기존 embedded 수집 경로는 전환 호환용 `infrastructure/compose.yaml`에 남겨 둡니다.
[통합 개발·이전 안내](docs/ops-monorepo-migration.md)를 먼저 확인하세요.
Kubernetes·Helm 배포 설정과 로컬 Argo CD GitOps 검증은 별도 [GovBiz-infra](https://github.com/GovBiz-Team/GovBiz-infra)에서 관리합니다.

Compose 서비스·내부 DNS는 `core-service`, `catalog-service`, `ai-service`, `ops-service`이며 Ops DB 컨테이너는 `ops-mysql`입니다.
Kubernetes Deployment·Service도 네 서비스명으로 통일합니다. 기존 로컬 컨테이너·데이터는 자동으로 변경하지 않습니다.
Compose가 생성하는 이름은 `<프로젝트명>-core-service-1`, `<프로젝트명>-ops-service-1` 형식입니다.
고정 `container_name`이나 이전 이름의 DNS 별칭은 두지 않습니다.

### 현재 구현·배포 상태

| 구분 | 상태 |
|---|---|
| 운영 환경 | 현재 운영 환경 없음. EC2 Compose·CodeBuild 설정은 재배포용 템플릿이며 자동 실행하지 않음 |
| Ops 로컬 개발 | 별도 Django 프로세스·MySQL, 상태 확인 API, 독립 테스트·컨테이너 검증 구현 |
| Core 공고 기능 분리 | 별도 Catalog 프로세스·MySQL과 인증된 HTTP 복제 경로 구현. 루트 Compose의 기본 경로이며 기존 데이터 이전·운영 배포는 별도 |
| Kubernetes | kind에서 Core·Catalog·AI·Ops와 독립 DB 실행, HTTP 복제·교차 DB 접근 거절·AI 단독 설정 롤아웃·Catalog 장애·테스트 데이터 복구 검증 완료 |
| 로컬 GitOps | Argo CD Core가 원격 Git 설정을 동기화하고 AI만 자동 변경·복귀하는 것 검증 완료. 상시 클러스터·이미지 digest 자동 반영은 미연결 |
| 이미지 릴리스 | 네 서비스 GHCR 첫 발행 성공 후 비공개 전환·익명 접근 차단 확인. 자동 발행은 일시 중지했으며 상시 배포와 비공개 pull 인증 연결은 별도 |
| 다음 단계 | Ops 관리자 인증·LLMOps 업무 기능, NetworkPolicy·의존성 readiness·장기 작업 종료 검증, 이미지 릴리스 CI와 상시 배포 환경 연결 |

**전체 MSA나 Kubernetes 운영 전환이 완료된 상태는 아닙니다.** 검증용 클러스터는 테스트 후 삭제했습니다.
[실제 검증 기록](https://github.com/GovBiz-Team/GovBiz-infra/blob/develop/docs/msa-validation-20260920.md)에서
완료 범위와 미검증 항목을 확인할 수 있습니다.

검증 이미지는 `infrastructure/scripts/build-msa-images.py`로 순차 빌드합니다.
[이미지 릴리스 CI와 활성화 조건](docs/msa-image-release.md)은 로컬 검증 빌드와 별개이며,
업로드 성공만으로 실제 배포가 완료됐다고 판단하지 않습니다.
[로컬 MSA·Helm·GitOps 실행 방법](https://github.com/GovBiz-Team/GovBiz-infra/blob/develop/docs/msa-local.md)을 따르며,
실제 `.env`·기존 DB·유료 AI를 사용하지 않는 임시 환경에서 검증합니다.

공고 분리 모드에서는 `Catalog → Catalog MySQL`이 원본 수집·색인을 소유하고,
`Core → 내부 HTTP API → 검증 → Core MySQL 조회용 복제본`으로 기존 관심 공고·파트너 모집 참조를 보존합니다.
전환 호환성을 위해 기존 Core 수집 구현도 남아 있으나, 분리 모드에서는 실행되지 않습니다.
[분리 범위·검증·운영 전환 조건](docs/catalog-service-extraction.md)을 참고하세요.

## 문서 안내

3차 프로젝트의 팀 소개부터 기능·아키텍처·평가 결과·회고까지는
[3차 프로젝트 README](docs/third-project/README.md)에서 별도로 관리합니다.
이 메인 README는 현재 모노레포 구성과 구현·배포 상태를 안내합니다.

- [3차 프로젝트 README](docs/third-project/README.md): 기존 프로젝트 소개와 결과·회고
- [전체 문서 목록](docs/README.md) · [기술 README](docs/technical-readme.md): 설계·API·실행·검증 안내
- [웹](frontend/web/README.md) · [모바일](frontend/mobile/README.md): 클라이언트 개발·실행 안내
- [Core](backend/core-service/README.md) · [Catalog](backend/catalog-service/README.md) · [AI](backend/ai-service/README.md) · [Ops](backend/ops-service/README.md): 서비스별 책임·실행·검증 안내
