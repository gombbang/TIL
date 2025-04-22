# CICD 파이프라인 구축과 NestJS Dockerized 실행

이 문서는 GitHub Actions를 이용한 CICD 파이프라인 구축과 Docker를 사용한 NestJS 애플리케이션의 빌드 및 실행 방법에 대해 설명합니다.

## 1. 프로젝트 초기 설정

### 1.1 Dockerfile 설정

먼저 **Amazon Linux 2**를 기반으로 한 Dockerfile을 생성합니다. 

```dockerfile
FROM amazonlinux:2

# Install Node.js 16.x dependencies only if not already installed
RUN yum update -y && \
    (rpm -q nodejs || curl -sL https://rpm.nodesource.com/setup_16.x | bash - && yum install -y nodejs) && \
    yum clean all

# Set working directory
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install npm dependencies
RUN npm install

# Copy the rest of the application code
COPY . .

# Expose port (update with your app's port)
EXPOSE 3000

# Run the Node.js application
ENTRYPOINT ["npm", "run", "start:prod"]
이 Dockerfile에서는 Node.js 16.x를 설치하고, 애플리케이션의 종속성을 설치한 후 애플리케이션을 실행합니다.
```

### 1.2 package.json의 start:prod 스크립트 수정
start:prod 스크립트에서 실행하는 경로를 dist/src/main으로 수정하여, 빌드된 파일을 실행하도록 변경했습니다.

```
"scripts": {
  "start:prod": "node dist/src/main"
}
```

## 2. GitHub Actions 파이프라인 설정
### 2.1 GitHub Actions Workflow 생성
다음은 GitHub Actions를 사용한 CICD 파이프라인 설정입니다. .github/workflows/ci-cd.yml 파일을 생성하여 파이프라인을 설정합니다.

```
name: CI/CD Pipeline (Test)

on:
  push:
    branches:
      - feature/cicd  # ✅ 이 브랜치에서만 작동

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Build Docker image
        run: |
          docker build -t my-app .

      - name: Debug: Check dist directory in Docker image
        run: |
          echo "🕵️‍♂️ Creating container from image..."
          docker create --name temp-container my-app
          echo "📦 Copying /app/dist to host..."
          docker cp temp-container:/app/dist ./dist-from-container || echo "❌ /app/dist not found"
          echo "🗂️ Listing contents of dist-from-container"
          ls -R ./dist-from-container || echo "❌ dist-from-container is empty or does not exist"
          echo "🧹 Removing temp container..."
          docker rm temp-container

      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Tag and Push Docker image
        run: |
          IMAGE_TAG=$(echo "${GITHUB_SHA}" | cut -c1-7)
          REPO_NAME=${{ secrets.DOCKER_HUB_REPO }}
          echo "🔎 Using IMAGE_TAG: $IMAGE_TAG"
          echo "🔎 Using REPO_NAME: $REPO_NAME"
          docker tag my-app:latest $REPO_NAME:$IMAGE_TAG
          docker push $REPO_NAME:$IMAGE_TAG
```
### 2.2 Workflow 설명
Checkout code: GitHub 리포지토리의 코드를 가져옵니다.

Build Docker image: Docker 이미지를 빌드합니다.

Debug: Docker 컨테이너에서 dist 디렉토리를 확인하여, 올바르게 빌드되었는지 점검합니다.

Login to DockerHub: DockerHub에 로그인합니다.

Tag and Push Docker image: Docker 이미지를 태그하고 DockerHub에 푸시합니다.

## 3. Dockerfile 빌드 및 실행
### 3.1 Docker 이미지 빌드

```
docker build -t my-app .
```
Docker 이미지를 빌드하고, my-app이라는 태그를 지정합니다.

### 3.2 Docker 컨테이너 실행
Docker 컨테이너를 실행하고 로그를 확인하여 애플리케이션이 정상적으로 동작하는지 점검합니다.
````
docker run -p 3000:3000 my-app
```

### 3.3 Docker 로그 확인
로그에서 애플리케이션의 실행 상태를 확인할 수 있습니다.

```
docker logs <container_id>
```

## 4. 문제 해결
### 4.1 MODULE_NOT_FOUND 에러 해결
처음 실행 시 Cannot find module '/app/dist/main' 오류가 발생했으나, 이는 dist 폴더의 빌드가 완료되지 않아서 생긴 문제였습니다. nestjs 애플리케이션에서 빌드된 파일이 dist/src/main.js에 존재하도록 경로를 수정했습니다.

```
"scripts": {
  "start:prod": "node dist/src/main"
}
```

## 5. 결론
이 가이드를 통해 Dockerized NestJS 애플리케이션을 GitHub Actions를 사용하여 자동으로 빌드하고 배포하는 파이프라인을 구축할 수 있었습니다.
