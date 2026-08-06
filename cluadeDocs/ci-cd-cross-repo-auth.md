# CI/CD 크로스레포 트리거 인증 방식 비교 — Fine-grained PAT vs GitHub App

## 배경: 왜 필요한가

레포가 `backend`/`frontend`/`Infra`로 분리되면서, backend/frontend가 이미지를 빌드·푸시한 뒤 실제 배포(`kubectl apply`)는 yaml이 있는 `Infra` 레포의 워크플로우가 담당하게 된다. 이걸 `repository_dispatch` 이벤트로 연결하려는데, GitHub Actions 기본 제공 `GITHUB_TOKEN`은 **자기 레포에만 유효**해서 다른 레포(Infra)를 건드릴 권한이 없다. 그래서 별도의 인증 수단이 필요하다.

두 가지 선택지: **Fine-grained Personal Access Token(PAT)** 또는 **GitHub App**.

---

## 옵션 1: Fine-grained PAT

### 어떻게 동작하나

1. GitHub 계정(예: 조직 Owner인 본인) → Settings → Developer settings → Fine-grained tokens에서 발급
2. Repository access를 "Only select repositories" → `Infra` 레포 하나만 선택
3. Permissions에서 "Contents: Read and write"만 부여 (repository_dispatch 발생에 필요한 최소 권한)
4. 발급된 토큰 값을 `backend`/`frontend` 레포 각각의 Repository secrets(예: `INFRA_DISPATCH_TOKEN`)에 저장
5. 워크플로우에서 `peter-evans/repository-dispatch` action(또는 `gh api repos/qKet/Infra/dispatches`)으로 이 토큰을 사용해 이벤트 전송

### 장점

- 설정이 단순하다 — 토큰 하나 발급해서 시크릿 두 곳(backend, frontend)에 붙여넣으면 끝. 별도 앱 등록/설치 절차가 없음.
- GitHub Actions 문서/커뮤니티에 흔히 나오는 표준 패턴이라 레퍼런스가 많음.
- 지금처럼 소규모 팀에서 빠르게 굴러가게 만들기 좋음.

### 단점

- **토큰이 특정 개인 계정에 종속된다.** 지금 `qKet` 조직 Owner가 1명뿐인데(GitHub이 "최소 2명 권장"이라고 경고한 것도 같은 맥락), 그 계정이 조직을 나가거나 정지되거나 비번/2FA 문제가 생기면 토큰도 같이 무효화되어 CI가 그 순간 멈춘다.
- 만료를 따로 설정 안 하면 사실상 장기 유효 자격증명이 시크릿 스토어에 계속 남아있는 셈이라 유출 시 위험 범위가 크다. (반대로 만료를 짧게 잡으면 주기적으로 재발급/재배포해야 하는 운영 부담이 생김)
- 권한을 "Infra 레포만"으로 좁혀도, 그 토큰 값 자체는 backend/frontend 레포 시크릿에 각각 복사되어 존재한다 — 시크릿이 여러 곳에 흩어질수록 유출 지점도 늘어난다.
- 감사 로그에 "누가 이 배포를 트리거했는지"가 사람 계정 이름으로 남아서, 실제로는 자동화가 한 일인데 사람이 직접 한 것처럼 보인다.

---

## 옵션 2: GitHub App

### 어떻게 동작하나

1. `qKet` 조직 Settings → Developer settings → GitHub Apps에서 앱 생성 (예: `qket-ci-bot`)
2. Permissions에서 Contents: Read and write만 부여 (그 외 전부 No access)
3. 이 앱을 `qKet` 조직에 설치하고, 설치 대상 레포를 `Infra` 하나로 제한
4. App ID + Private key(.pem)를 생성해서 backend/frontend 레포 시크릿에 저장
5. 워크플로우에서 `actions/create-github-app-token` action으로 **실행할 때마다 단명 토큰(기본 만료 1시간)**을 새로 발급받아 사용

### 장점

- 특정 사람 계정과 완전히 분리된 "봇 정체성" — Owner가 바뀌거나 조직을 나가도 CI에 영향 없음.
- 매 실행마다 새로 발급되는 단명 토큰이라, 시크릿으로 실제 저장돼있는 건 Private key뿐. 이게 유출돼도 그 자체로 바로 API 호출이 되는 게 아니라 installation token 발급 절차를 한 번 더 거쳐야 해서 PAT보다 유출 시 피해가 작다.
- 권한을 레포 단위뿐 아니라 API 카테고리 단위로 세밀하게 제한 가능. 조직 감사 로그에 "앱 이름"으로 명확히 남아서 사람 활동과 자동화가 구분됨.
- 나중에 레포가 늘어나거나(예: 별도 CD 레포 추가) 다른 자동화를 붙일 때도 같은 앱을 재사용하기 쉽다.

### 단점

- 초기 설정이 PAT보다 손이 더 간다 (앱 생성 → 권한 설정 → 조직에 설치 → private key 발급/보관 → 워크플로우에 토큰 발급 스텝 추가).
- Private key(.pem)를 시크릿으로 관리해야 해서, 일반 문자열 토큰보다 다루기가 조금 번거롭다 (개행 처리 등에서 실수하기 쉬움).
- 팀 규모(멤버 4명)에 비해 다소 과한 엔지니어링으로 느껴질 수 있음. 다만 인프라를 조직 단위로 이미 옮긴 상황이라, 자동화를 개인 계정에서 분리해두는 방향 자체는 장기적으로 맞다.

---

## 비교 요약

| | Fine-grained PAT | GitHub App |
|---|---|---|
| 설정 난이도 | 낮음 | 중간 |
| 정체성 | 특정 개인 계정에 종속 | 독립된 봇 정체성 |
| 토큰 수명 | 사실상 장기(수동 만료 설정 안 하면) | 매 실행마다 단명(~1시간) |
| Owner 이탈/계정 문제 시 영향 | CI 중단 위험 있음 | 없음 |
| 감사 로그 가독성 | 사람 이름으로 남음 | 앱 이름으로 명확히 구분됨 |
| 레포가 늘어날 때 확장성 | 토큰 재발급·재배포 필요 | 설치 대상 레포만 추가하면 됨 |
| 유출 시 피해 범위 | 큼 (토큰 자체가 바로 유효) | 작음 (private key만으로는 즉시 사용 불가) |

## 판단 기준

- **지금 당장 빠르게 굴러가는 게 우선**이면 PAT로 시작해도 무방. 다만 나중에 App으로 옮기려면 워크플로우 파일(토큰 발급 스텝)을 다시 만져야 한다.
- **처음부터 제대로 두고 싶다**면 GitHub App — 어차피 CI/CD를 완전히 새로 설계하는 시점이라, 크로스레포 트리거 로직을 두 번 만지지 않아도 된다는 이점이 있다.
