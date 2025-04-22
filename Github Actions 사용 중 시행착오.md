# CI/CD Pipeline for Docker Deployment

이 프로젝트는 GitHub Actions를 사용한 **CI/CD 파이프라인**으로, **Docker 이미지**를 빌드하고, 이를 **DockerHub**에 푸시한 후, **EC2 서버**에 배포하는 자동화된 흐름을 설정하는 과정입니다.

## 목차

- [소개](#소개)
- [CI/CD 파이프라인 설정](#cicd-파이프라인-설정)
  - [1. 초기 CI/CD 파이프라인 설정](#1-초기-cicd-파이프라인-설정)
  - [2. Docker 이미지 푸시 문제 해결](#2-docker-이미지-푸시-문제-해결)
  - [3. EC2 배포 문제 해결](#3-ec2-배포-문제-해결)
  - [4. EC2 환경 변수 전달 문제 해결](#4-ec2-환경-변수-전달-문제-해결)
  - [5. 디버깅을 위한 echo 명령어 추가](#5-디버깅을-위한-echo-명령어-추가)
  - [6. EC2 배포 스크립트 개선](#6-ec2-배포-스크립트-개선)
  - [7. 최종 CI/CD 파이프라인 완성](#7-최종-cicd-파이프라인-완성)
- [결론](#결론)

---

## 소개

이 프로젝트는 **GitHub Actions**를 사용하여 **CI/CD 파이프라인**을 설정하는 과정을 다룹니다. 이 파이프라인은 **Docker 이미지 빌드, DockerHub 푸시, EC2 서버 배포**를 자동으로 처리합니다. 각 단계를 설정하고 발생한 문제들을 해결해 나가면서 자동화된 배포 시스템을 구현합니다.

---

## CI/CD 파이프라인 설정

### 1. 초기 CI/CD 파이프라인 설정

처음에 GitHub Actions를 설정하여 `push` 및 `pull_request` 이벤트가 발생할 때마다 **Docker 이미지 빌드** 및 **DockerHub 푸시** 작업을 자동으로 실행하도록 파이프라인을 설정했습니다.

```yaml
on:
  push:
    branches:
      - main
  pull_request:
    types: [closed]
```

### 2. Docker 이미지 푸시 문제 해결
Docker 이미지를 DockerHub에 푸시하는 과정에서 레포지토리 이름이 잘못 설정되어 문제가 발생했습니다. 이를 해결하기 위해 GitHub Secrets에 저장된 username, password, repository 값을 사용하여 docker login 명령어로 DockerHub에 로그인하고 이미지를 푸시하도록 수정했습니다.

### 3. EC2 배포 문제 해결
EC2 서버에서 Docker 이미지를 pull하려고 할 때 이미지가 없다는 오류가 발생했습니다. 이를 해결하기 위해 DockerHub에서 이미지를 올바르게 pull하도록 스크립트를 수정했습니다.

### 4. EC2 환경 변수 전달 문제 해결
EC2로 SSH 접속 시 환경 변수를 제대로 전달하지 못하는 문제가 있었습니다. 이를 해결하기 위해 envs 대신 env를 사용하여 DOCKER_HUB_REPO와 GITHUB_SHA 값을 EC2 서버로 전달하도록 수정했습니다.

### 5. 디버깅을 위한 echo 명령어 추가
각각의 변수인 REPO_NAME과 IMAGE_TAG가 제대로 설정되었는지 확인하기 위해 echo 명령어로 디버깅 출력을 추가했습니다. 이를 통해 값이 정확히 전달되었는지 확인할 수 있었습니다.

```echo "🔎 Using IMAGE_TAG: $IMAGE_TAG"
echo "🔎 Using REPO_NAME: $REPO_NAME"
```

### 6. EC2 배포 스크립트 개선
EC2에서 Docker 이미지를 가져오고 실행하는 과정에서 기존 컨테이너와 이미지를 제거하고 최신 이미지를 pull하고 실행하는 스크립트를 작성했습니다.

```sudo docker stop $(docker ps -a -q) || true
sudo docker rm $(docker ps -a -q) || true
sudo docker rmi -f $(docker images -q) || true
sudo docker pull $REPO_NAME:$IMAGE_TAG
sudo docker run -d --name my-app -p 3000:3000 $REPO_NAME:$IMAGE_TAG
```

### 7. 최종 CI/CD 파이프라인 완성
최종적으로 GitHub Actions에서 Docker 이미지를 빌드하고 DockerHub에 푸시한 후, EC2 서버에 배포하는 전체 파이프라인을 완성했습니다. 이 파이프라인은 자동으로 Docker 이미지를 빌드하고 배포하는 작업을 수행하며, 필요한 환경 변수와 DockerHub 레포지토리 정보를 관리합니다.

## 결론
이 프로젝트에서는 CI/CD 파이프라인을 설정하면서 발생한 문제들을 차근차근 해결해 나갔습니다. GitHub Actions를 사용하여 Docker 이미지 빌드, DockerHub 푸시, EC2 배포까지 자동화하는 시스템을 성공적으로 구축했습니다. CI/CD 파이프라인을 설정하면서 겪은 문제들을 해결한 경험은 향후 자동화된 배포 작업에 큰 도움이 될 것입니다.
