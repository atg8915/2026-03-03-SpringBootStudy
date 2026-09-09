# 📘 Spring Boot Day 22 — `SpringRecipeAIProject` Jenkins 배포 파이프라인을 Docker Compose 기반으로 전환

## 0. 핵심 빠른 참조 — 어제 대비 오늘 바뀐 점

| 구분 | 어제(Day21) | 오늘 |
|------|-------------|------|
| 컨테이너 제어 방식 | `docker stop`/`rm`/`pull`/`run`을 단계별로 직접 호출 | `docker compose down`/`pull`/`up -d`로 일괄 제어 |
| 이미지 이름 관리 | Jenkinsfile 각 단계마다 `atg8915/ai-app:latest` 문자열을 반복 기입 | `DOCKER_IMAGE` 환경변수 하나로 선언해 재사용 |
| 파이프라인 결과 처리 | `post` 블록 없음 | `post { success {...} failure {...} }`로 성공/실패 분기 |
| 배포 후 상태 확인 | 없음 | `docker compose ps`로 컨테이너 상태 확인 단계 추가 |
| DockerHub 로그인 명령 | `echo "$DH_PASS" docker login ...`(파이프 누락) | `echo "$DH_PASS" | docker login ...`로 파이프 수정 |

---

## 1. `SpringRecipeAIProject` 작업 내용

### 1-1. Jenkinsfile — `docker run` 방식에서 `docker compose` 방식으로 전환

```groovy
pipeline {
	agent any
	environment {
		APP_DIR = "~/app"
		JAR_NAME = "SpringRecupeAIProject-0.0.1-SNAPSHOT.jar"
		DOCKER_IMAGE = "atg8915/ai-app:latest"
	}
	stages {
		stage("Repository Checkout"){
			steps{ checkout scm }
		}

		stage("Create .env"){
			steps{
				withCredentials([
					string(credentialsId: 'post-url', variable: 'POST_URL'),
					string(credentialsId: 'gen-key', variable: 'GEN_KEY')
				]){
					sh '''
					   echo "SPRING_PROFILES_ACTIVE=prod" > .env
					   echo "POST_URL=${POST_URL}" >> .env
					   echo "GEN_KEY=${GEN_KEY}" >> .env
					   chmod 600 .env
					   '''
				}
			}
		}

		stage("Gradle Permission"){ steps{ sh 'chmod +x gradlew' } }
		stage("Gradlew Build"){ steps { sh './gradlew clean build -x test' } }
		stage("Docker Build"){ steps{ sh 'docker build -t ${DOCKER_IMAGE} .' } }

		stage("DockerHub Login") {
			steps{
				withCredentials([usernamePassword(
					credentialsId: 'dockerhub_info',
					usernameVariable: 'DH_USER',
					passwordVariable: 'DH_PASS'
				)]){
					sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
				}
			}
		}

		stage("DockerHub Push") { steps { sh 'docker push ${DOCKER_IMAGE}' } }
		stage("Docker Compose DOWN") { steps { sh 'docker compose down || true' } }
		stage("Docker Compose Pull"){ steps { sh 'docker compose pull' } }
		stage("Docker Compose Up"){ steps{ sh 'docker compose up -d' } }
		stage("Container Check"){ steps{ sh 'docker compose ps' } }
	}
}
post {
	success {
		echo 'Docker Compose 배포 성공'
	}
	failure {
		echo 'Docker Compose 배포 실패'
		sh 'docker compose ps || true'
	}
}
```
- 기존에는 `docker stop ai-app || true` → `docker rm ai-app || true` → `docker pull <이미지>` → `docker run -d --name ai-app ...`처럼 컨테이너 하나를 수동으로 내리고 받고 올리는 4단계를 일일이 호출함. 오늘부터는 `docker-compose.yml`에 컨테이너 설정을 몰아넣고 `docker compose down/pull/up -d` 3단계로 줄임
- `DOCKER_IMAGE = "atg8915/ai-app:latest"`를 `environment` 블록에 선언해두고 `Docker Build`/`DockerHub Push` 단계에서 `${DOCKER_IMAGE}`로 재사용 — 이미지 이름을 바꿀 때 한 곳만 고치면 되는 구조
- `Container Check` 단계(`docker compose ps`)를 새로 추가해 배포 직후 컨테이너가 실제로 떠 있는지 파이프라인 안에서 확인하게 함

### 1-2. `docker-compose.yml` 신규 작성

```yaml
version: "3.8"
services:
  app:
   build: .
   image: atg8915/ai-app:latest
   container_name: ai-app
   ports:
     - "9090:9090"
   restart: always
   env_file:
         - .env
```
- `build: .`와 `image:`를 함께 지정 — 로컬에서 직접 빌드할 수도, 지정한 이미지명으로 태깅할 수도 있는 구성
- `restart: always`로 컨테이너가 죽었을 때 자동 재시작되도록 설정(기존 `docker run` 방식에는 재시작 정책이 없었음)
- `env_file: - .env`로 Jenkins가 생성한 `.env`를 컨테이너에 그대로 주입 — `docker run --env-file .env` 방식과 동일한 역할을 compose 설정으로 옮긴 것

### 1-3. `post { success / failure }` 블록으로 배포 결과 분기 처리

- `post` 블록은 `stages` 전체가 끝난 뒤 성공/실패 여부에 따라 추가 동작을 실행하는 영역
- `success`에는 배포 성공 로그만 출력, `failure`에는 실패 로그 출력 후 `docker compose ps || true`로 실패 시점의 컨테이너 상태를 바로 확인할 수 있게 함
- `|| true`를 여기서도 붙인 이유는 동일 — 이미 실패한 빌드에서 상태 확인 명령마저 에러를 내며 파이프라인을 완전히 끊지 않게 하려는 것

### 1-4. DockerHub 로그인 명령의 파이프(`|`) 누락 수정

```bash
# 수정 전 — 파이프 없이 나열되어 있어 비밀번호가 표준입력으로 전달되지 않음
echo "$DH_PASS" docker login -u "$DH_USER" --password-stdin

# 수정 후
echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
```
- `--password-stdin`은 표준입력(stdin)으로 비밀번호를 받는 옵션인데, `|`(파이프)가 빠지면 `echo`와 `docker login`이 그냥 순서대로 나열된 별개의 커맨드로 취급됨
- 그 결과 `docker login`에 비밀번호가 전달되지 않아 로그인이 되지 않음 — 파이프 하나 있고 없고로 동작이 완전히 달라지는 경우

---

## 2. 비교표 — Docker 배포 제어 방식 (어제 vs 오늘)

| 구분 | 어제(Day21) | 오늘 |
|------|-------------|------|
| 컨테이너 중지 | `docker stop ai-app \|\| true` | `docker compose down \|\| true` |
| 최신 이미지 받기 | `docker pull atg8915/ai-app:latest` | `docker compose pull` |
| 컨테이너 기동 | `docker run -d --name ai-app -p 9090:9090 --env-file .env atg8915/ai-app:latest` | `docker compose up -d` |
| 재시작 정책 | 없음 | `docker-compose.yml`의 `restart: always` |
| 배포 후 상태 확인 | 없음 | `docker compose ps`(Container Check 단계) |
| 결과 후처리 | 없음 | `post { success / failure }` |

---

## 3. 다시 만들 때 체크리스트

```text
[Jenkins → Docker Compose 배포 전환]
① 여러 docker 명령(stop/rm/pull/run)을 쓰던 컨테이너는 docker-compose.yml로 설정을 모아 down/pull/up -d 3단계로 축소
② docker-compose.yml의 env_file로 Jenkins가 만든 .env를 그대로 주입(--env-file과 동일 역할)
③ restart: always로 컨테이너 비정상 종료 시 자동 재시작 설정
④ 이미지명은 Jenkinsfile의 environment 블록에 변수(DOCKER_IMAGE)로 선언해 각 단계에서 재사용
⑤ docker compose ps로 배포 직후 컨테이너 상태를 확인하는 단계를 파이프라인에 포함

[post 블록]
⑥ post { success {...} failure {...} }로 빌드 성공/실패에 따른 후처리(로그 출력, 상태 확인)를 분리

[디버깅 포인트]
⑦ echo "$변수" | command --password-stdin 형태에서 파이프(|)가 빠지면 표준입력으로 값이 전달되지 않아 로그인이 조용히 실패할 수 있음 — 파이프 유무를 항상 확인
```
