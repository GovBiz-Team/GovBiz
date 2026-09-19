# 🏛️ GovBiz

**LLM 기반 정부지원사업 탐색·신청 관리 플랫폼**

이 저장소는 React·Core API·Catalog Service·AI Service와 Django 기반 Ops를 함께 관리하는 **애플리케이션 모노레포**입니다.
Ops 소스는 [`backend/ops-service`](backend/ops-service)에 있으며, 상태 확인 API·전용 MySQL·Gunicorn 실행 이미지를 갖추고 있습니다.
소스를 통합해도 서비스 프로세스·의존성·DB 책임은 분리합니다.

웹(`frontend/`)과 React Native 앱(`mobile/`)을 함께 관리합니다.
[모바일 실행·기능 안내](mobile/README.md) · [공통 코드·workspace 관리](docs/mobile-monorepo.md)

전체 로컬 실행은 루트 `compose.yaml`, 기존 웹·Core·AI 실행은 `infrastructure/compose.yaml`을 사용합니다.
[통합 개발·이전 안내](docs/ops-monorepo-migration.md)를 먼저 확인하세요.
Kubernetes 배포 설정·검증과 향후 Argo CD 연결은 별도 [GovBiz-infra](https://github.com/GovBiz-Team/GovBiz-infra)에서 관리합니다.

### 현재 구현·배포 상태

| 구분 | 상태 |
|---|---|
| 기존 웹 서비스 | Vercel + AWS EC2 Compose 기반 배포 유지. 이번 변경으로 운영 서버·데이터·배포 연결을 변경하지 않음 |
| Ops 로컬 개발 | 별도 Django 프로세스·MySQL, 상태 확인 API, 독립 테스트·컨테이너 검증 구현 |
| Core 공고 기능 분리 | 별도 Catalog 프로세스·MySQL과 인증된 HTTP 복제 경로 구현. 선택형 로컬 Compose로 전환하며 기존 AWS에는 적용하지 않음 |
| Kubernetes 1단계 | kind에서 Ops + 검증용 MySQL 실행, DB 장애·PVC 보존·Pod 복구·이미지 롤백 검증 완료 |
| 다음 단계 | Argo CD GitOps, Core·Catalog·AI의 Kubernetes 이전, Ops 관리자 인증·LLMOps 업무 기능, AWS Kubernetes 운영 전환 |

**전체 MSA나 Kubernetes 운영 전환이 완료된 상태는 아닙니다.** 검증용 클러스터는 테스트 후 삭제했습니다.
[실제 검증 기록](https://github.com/GovBiz-Team/GovBiz-infra/blob/develop/docs/kubernetes-validation-20260919.md)에서
완료 범위와 미검증 항목을 확인할 수 있습니다.

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
- [웹](frontend/README.md) · [모바일](mobile/README.md): 클라이언트 개발·실행 안내
- [Core](backend/core-service/README.md) · [Catalog](backend/catalog-service/README.md) · [AI](backend/ai-service/README.md) · [Ops](backend/ops-service/README.md): 서비스별 책임·실행·검증 안내
