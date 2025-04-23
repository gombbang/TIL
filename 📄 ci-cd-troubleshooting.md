# CI/CD 구축 및 문제 해결 과정 정리

이 문서는 `GitHub Actions`, `Docker`, `NestJS`를 활용한 CI/CD 파이프라인 구축 중 겪은 문제와 해결 과정을 정리한 기록입니다. 같은 문제를 겪는 개발자에게 도움이 되길 바랍니다.

---

## ✅ 1단계: Docker 이미지 빌드

- 문제: Dockerfile에서 `dist/main` 파일을 찾을 수 없다는 에러  
- 원인: `nest build`가 실행되지 않아 `dist` 디렉토리가 없었음
- 해결:
  - `npm run build` 추가
  - `Dockerfile` 수정: `COPY . .` 전에 `npm run build` 실행 필요

**수정한 Dockerfile 예시**
```Dockerfile
# build
RUN npm run build
```
🔍 2단계: 빌드 결과 디버깅
dist 디렉토리의 존재 여부 확인을 위해 GitHub Actions에 디버그 step 추가

```
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
```
🛠️ 3단계: NestJS 시작 시 모듈 에러
문제: JwtService를 AuthService에서 찾을 수 없다는 NestJS 런타임 에러

원인: JwtModule이 AuthModule에 import되지 않음

해결:

기존
```
@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```
수정

```
import { JwtModule } from '@nestjs/jwt';

@Module({
  imports: [
    TypeOrmModule.forFeature([User]),
    JwtModule.register({
      secret: process.env.JWT_SECRET || 'defaultSecret',
      signOptions: { expiresIn: '1h' },
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService],
})
export class AuthModule {}
```
.env 파일에 JWT_SECRET 설정 필요

💡 기타 팁
GitHub Actions에서 Docker 컨테이너 내부 파일 구조를 디버깅하고 싶다면 docker cp 명령어를 활용

NestJS는 DI(Dependency Injection)를 사용하는 프레임워크이므로 의존 모듈을 빠짐없이 등록해야 함

✅ 진행 상태
 CI/CD GitHub Actions 구성

 Docker 이미지 빌드

 NestJS 애플리케이션 dist 정상 생성 확인

 JwtService 모듈 문제 해결

 실제 애플리케이션 정상 실행 및 테스트

📌 다음 단계: .env를 통해 프로덕션 설정 정리 & 실제 배포 환경과의 통합 테스트

---

### 🔧 저장 방법

1. VS Code 또는 에디터에서 새 파일 생성
2. 이름: `ci-cd-troubleshooting.md`
3. 위 내용 붙여넣기
4. 저장 후 Git에 커밋

```bash
git add ci-cd-troubleshooting.md
git commit -m "docs: add CI/CD troubleshooting guide"
git push origin feature/cicd
```
