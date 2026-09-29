# GitHub Actions 공통 구성

NuGet 캐시, Docker BuildKit 캐시와 CI 실행 원칙을 조직 저장소에서 재사용한다. 기존 저장소에 자동으로 워크플로를 추가하거나 운영 배포를 실행하지 않는다. 각 저장소가 필요한 action을 채택하고, 새 저장소는 Actions의 조직 템플릿에서 시작한다.

## 도입 방법

- `actions/setup-dotnet-cached`: 체크아웃 이후 SDK 버전 또는 global.json 경로를 지정한다. NuGet 패키지만 캐시하며 restore/build/test는 정상 수행한다. NUGET_PACKAGES를 변경한 저장소는 packages-path도 맞춘다.
- `actions/build-container-cached`: 이미지와 아키텍처별 scope를 지정한다. 기본은 push=false이다. 레지스트리 로그인과 배포 승인은 호출 저장소가 처리한다. Docker 데몬/Buildx를 지원하는 러너에서 사용한다.
- 각 호출은 검토한 이 저장소의 **40자리 commit SHA**에 고정한다. 외부 변경이 운영 워크플로에 즉시 전파되지 않게 한다.
- 이미 영속적인 자체 러너 캐시나 별도 검증 이미지 재사용이 있는 경우 기존 구성을 유지할 수 있다.

## 실행과 결과물 재사용 원칙

1. PR 검증은 pull_request, 병합 후 검증/배포는 기본 브랜치 push로 실행한다. 같은 기능 브랜치에 push와 PR 검증을 겹쳐 등록하지 않는다. PR merge SHA 검증과 기본 브랜치의 실제 커밋 검증은 서로 다른 검증이다.
2. PR concurrency는 workflow + PR 번호로 묶어 오래된 검증을 취소한다. 운영 배포는 별도 그룹으로 직렬화하고 진행 중 배포를 취소하지 않는다. 이벤트별 조건과 기존 필수 검사 이름은 유지한다.
3. 동일 실행에서는 build/test/publish를 한 번 수행하고 needs + upload/download-artifact로 배포에 전달한다. 가능하면 검증한 이미지의 digest를 그대로 배포한다. 릴리스용 실제 SQL·복구·권한 검증은 캐시 성공으로 대체하지 않는다.
4. 다른 실행의 결과물을 가져오는 경우 승인한 정확한 commit SHA, 원본 저장소, 신뢰한 기본 브랜치·워크플로, 성공 결과, 실행 ID를 모두 확인한다. PR/포크 결과물을 운영 권한으로 실행하지 않는다. workflow_run만 추가해 미검증 결과물을 승격하지 않는다.
5. 캐시는 성능 보조다. 캐시가 없는 상태에서도 정상 빌드돼야 한다. 비밀키·토큰·설정 파일·DB·개인정보는 캐시에 넣지 않는다. 오래된 캐시와 큰 산출물은 보관 기간을 제한한다.
6. 정적 파일 배포·예약 알림·문서 전용 저장소에는 불필요한 빌드나 패키지 캐시를 추가하지 않는다. 보관 저장소는 재활성화할 때 적용한다.

## 템플릿

템플릿을 추가한 후 저장소의 브랜치, SDK, 프로젝트 및 lockfile 경로를 확인한다. 템플릿 자체로 조직 전체에 자동 강제되는 것은 아니다. 기존 저장소의 배포 승인·OIDC·환경·러너 라벨은 저장소별 PR로 변경한다.

자료: [워크플로 재사용](https://docs.github.com/en/actions/concepts/workflows-and-actions/reusing-workflow-configurations), [의존성 캐시](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching), [Docker 캐시](https://docs.docker.com/build/ci/github-actions/cache/).
