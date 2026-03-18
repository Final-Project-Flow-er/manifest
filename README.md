# CI/CD 및 시스템 아키텍처 설계

<br>

# 1️⃣ 개요 (System Overview)

> **"Infrastructure as Code(IaC) 및 GitOps 아키텍처를 통한 완전 자동화 배포 환경 구축"**
> 

본 문서는 Chain-G 프로젝트의 안정적인 서비스 운영과 고도화된 배포 자동화를 위해 설계된 CI/CD 파이프라인 및 클러스터 인프라 아키텍처를 다룹니다.

단순한 빌드 자동화를 넘어, Git Repository를 인프라 운영의 단일 진실 공급원(Single Source of Truth)으로 삼는 GitOps 모델을 지향합니다. 이를 통해 인프라의 가시성을 확보하고, 개발자가 코드 커밋만으로 빌드부터 배포, 실시간 모니터링 알림까지 통합적으로 제어할 수 있는 환경을 구축하였습니다.

<br>

# 2️⃣ 전체 아키텍처 (Deployment Architecture)

### 📦 배포 프로세스 흐름

<img width="800" alt="Web App Reference Architecture drawio (2)" src="https://github.com/user-attachments/assets/3fddac84-8e11-4833-b73a-97bc43d423a2" />

1. **CI Stage** : 개발자가 코드를 Merge하면 **Jenkins**가 Webhook을 수신하여 즉시 빌드를 시작합니다.
2. **Containerization** : 빌드된 바이너리는 Docker 이미지화되어 **AWS ECR**에 안전하게 푸시됩니다.
3. **GitOps Trigger** : Jenkins 파이프라인은 배포용 **Manifest Repository**의 이미지 태그를 최신 빌드 번호(예: `#70`)로 자동 업데이트합니다.
4. **CD Stage** : 클러스터 내의 **ArgoCD**가 매니페스트의 변경 사항을 감지하고, **AWS EKS**의 현재 상태를 동기화(Sync)합니다.
5. **Monitoring** : 모든 빌드 및 배포 상태는 **Discord** 채널을 통해 팀원들에게 실시간 공유됩니다.

<br>

# 3️⃣ 기술 스택 및 구축 근거 (Tech Stack)

| 분류 | 선택 스택 | 도입 근거 및 기술적 성과 |
| --- | --- | --- |
| **CI Engine** | **Jenkins** | 파이프라인 스크립트(Groovy)를 통한 복잡한 멀티 스테이지 제어 및 Credentials 보안 관리 최적화 |
| **CD / GitOps** | **ArgoCD** | 선언적(Declarative) 방식의 배포 관리로 설정값 변조 방지 및 장애 발생 시 즉각적인 Git 롤백 지원 |
| **Orchestration** | **AWS EKS** | 엔터프라이즈급 Kubernetes 클러스터 운영을 위한 고가용성(High Availability) 확보 및 무중단 배포(Rolling Update) 구현. |
| **Registry** | **Docker Hub** | 범용적인 컨테이너 저장소 활용 및 빌드 번호 기반의 이미지 버전 관리(Versioning) 체계 수립 |
| **Networking** | **AWS ALB** | Application Load Balancer를 활용한 L7 계층의 트래픽 라우팅 및 SSL/TLS 인증서를 통한 보안 통신(HTTPS) 강화 |

<br>

# 4️⃣ 주요 구현 기능 (Key Features)

## 4.1 상세 파이프라인 설계 (Pipeline Detail)

백엔드와 프론트엔드의 특성에 맞춘 최적화된 빌드/배포 프로세스를 구축하였습니다.

### ⚙️ Backend Pipeline (Spring Boot)

<img width="1057" height="388" alt="통테2" src="https://github.com/user-attachments/assets/b4d06e86-1bd8-4329-be13-7b782191b1b1" />

- **Preparation**: GitHub `dev` 브랜치로부터 최신 소스코드를 Clone 합니다.
- **Source Build**: `Gradle`을 사용하여 프로젝트를 Clean & Build 합니다. (빌드 속도 향상을 위해 테스트 단계 제외 옵션 활용)
- **Dockerizing**: 생성된 `.jar` 파일을 기반으로 빌드 번호를 태그로 지정하여 Docker 이미지를 생성하고 AWS ECR에 Push 합니다.
- **Manifest Sync**: 백엔드 전용 매니페스트 파일(`k8s/backend/deployment.yml`) 내 이미지 경로를 `sed` 명령어로 최신화하여 커밋합니다.

<details>
  <summary>backend 파이프라인 코드</summary>

```jsx
pipeline {
    agent any
    tools { jdk 'openjdk17' }
    environment {
        ECR_REGISTRY = '479699168320.dkr.ecr.ap-northeast-2.amazonaws.com'
        IMAGE_NAME = 'chaing-backend'
        FULL_IMAGE_URL = "${env.ECR_REGISTRY}/${env.IMAGE_NAME}"
        SOURCE_GITHUB_URL = 'git@github.com:Final-Project-Flow-er/backend.git'
        DISCORD_URL = credentials('DISCORD_WEBHOOK_URL')
    }
    stages {
        stage('Preparation') {
            steps {
                git branch: 'dev', 
                    url: "${env.SOURCE_GITHUB_URL}", 
                    credentialsId: 'ssh-jenkins-github--key' 
            }
        }
        stage('Source Build & Test') {
            steps {
                script {
                    sh "chmod +x ./gradlew"
                    sh "./gradlew clean build -x test"
                }
            }
        }
        stage('Push to AWS ECR') {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'aws-credentials-id', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=ap-northeast-2

                            aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}

                            docker build -t ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER} .
                            docker tag ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER} ${env.FULL_IMAGE_URL}:latest

                            docker push ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER}
                            docker push ${env.FULL_IMAGE_URL}:latest
                        """
                    }
                    sh "docker image prune -f"
                }
            }
        }
        stage('Update Manifest') {
            steps {
                script {
                    sh "rm -rf temp-manifest"
                    sh "git clone -b main git@github.com:Final-Project-Flow-er/manifest.git temp-manifest"
                    
                    dir('temp-manifest') {
                        sh "sed -i 's|image: .*chaing-backend:.*|image: ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER}|g' k8s/backend/deployment.yml"
                        
                        sh "git config user.email 'kseo_o@naver.com'"
                        sh "git config user.name 'tjddms'"
                        sh "git add ."
                        sh "git commit -m 'Update backend image to build #${env.BUILD_NUMBER}'"
                        
                        sshagent(['ssh-jenkins-github--key']) {
                            sh "git push origin main"
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                
                discordSend(
                    title: "백엔드 빌드 성공! 🎉",
                    description: """
                        **빌드 성공!**
                        **제목:** #${env.BUILD_NUMBER}
                        **결과:** ✅ SUCCESS
                        **실행 시간:** ${duration}
                        **링크:** [빌드 결과 보기](${env.BUILD_URL})
                    """,
                    result: 'SUCCESS',
                    link: env.BUILD_URL,
                    webhookURL: env.DISCORD_URL
                )
            }
        }
        failure {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                
                discordSend(
                    title: "백엔드 빌드 실패! 😭",
                    description: """
                        **빌드 실패!**
                        **제목:** #${env.BUILD_NUMBER}
                        **결과:** ❌ FAILURE
                        **실행 시간:** ${duration}
                        **링크:** [로그 확인하기](${env.BUILD_URL}console)
                    """,
                    result: 'FAILURE',
                    link: env.BUILD_URL,
                    webhookURL: env.DISCORD_URL
                )
            }
        }
    }
}
```

</details>


### 🎨 Frontend Pipeline (React/Vite)

<img width="1058" height="398" alt="통합테스트2" src="https://github.com/user-attachments/assets/4cb98fdb-0ba2-4d72-9836-5186ccfcc178" />

- **Preparation**: Node.js 환경(v20)에서 소스코드를 준비합니다.
- **Source Build**: `npm install` 후 환경 변수(`.env`)를 주입하고 `npm run build`를 통해 정적 리소스를 생성합니다.
- **Dockerizing**: 빌드된 결과물을 포함하여 프론트엔드 전용 Docker 이미지를 빌드 및 Push 합니다.
- **Manifest Sync**: 프론트엔드용 매니페스트(`k8s/frontend/deployment.yml`)를 업데이트하여 배포 자동화 트리거를 발생시킵니다.

<details>
  <summary>frontend 파이프라인 코드</summary>

```jsx
pipeline {
    agent any
    tools { nodejs 'node-20' }
    environment {
        ECR_REGISTRY = '479699168320.dkr.ecr.ap-northeast-2.amazonaws.com'
        IMAGE_NAME = 'chaing-frontend'
        FULL_IMAGE_URL = "${env.ECR_REGISTRY}/${env.IMAGE_NAME}"
        SOURCE_GITHUB_URL = 'git@github.com:Final-Project-Flow-er/frontend.git'
        DISCORD_URL = credentials('DISCORD_WEBHOOK_URL')
    }

    stages {
        stage('Preparation') {
            steps {
                git branch: 'dev', 
                    url: "${env.SOURCE_GITHUB_URL}", 
                    credentialsId: 'ssh-jenkins-github--key' 
            }
        }

        stage('Source Build') {
            steps {
                script {
                    sh "npm install"
                    sh "echo 'VITE_API_BASE_URL=http://api.chaing-flower.com/api/v1' > .env"
                    sh "npm run build"
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                script {
                    withCredentials([aws(credentialsId: 'aws-credentials-id', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh """
                            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
                            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
                            export AWS_DEFAULT_REGION=ap-northeast-2

                            aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin ${env.ECR_REGISTRY}
                            
                            docker build -t ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER} .
                            docker tag ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER} ${env.FULL_IMAGE_URL}:latest
                            
                            docker push ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER}
                            docker push ${env.FULL_IMAGE_URL}:latest
                        """
                    }
                    sh "docker image prune -f"
                }
            }
        }
        
        stage('Update Manifest') {
            steps {
                script {
                    sh "rm -rf temp-manifest"
                    sh "git clone -b main git@github.com:Final-Project-Flow-er/manifest.git temp-manifest"
                    
                    dir('temp-manifest') {
                        sh "sed -i 's|image: .*chaing-frontend:.*|image: ${env.FULL_IMAGE_URL}:${env.BUILD_NUMBER}|g' k8s/frontend/deployment.yml"
                        
                        sh "git config user.email 'kseo_o@naver.com'"
                        sh "git config user.name 'tjddms'"
                        sh "git add ."
                        sh "git commit -m 'Update frontend image to build #${env.BUILD_NUMBER}'"
                        
                        sshagent(['ssh-jenkins-github--key']) {
                            sh "git push origin main"
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                
                discordSend(
                    title: "프론트엔드 빌드 성공! 🎉",
                    description: """
                        **빌드 성공!**
                        **제목:** #${env.BUILD_NUMBER}
                        **결과:** ✅ SUCCESS
                        **실행 시간:** ${duration}
                        **링크:** [빌드 결과 보기](${env.BUILD_URL})
                    """,
                    result: 'SUCCESS',
                    link: env.BUILD_URL,
                    webhookURL: env.DISCORD_URL
                )
            }
        }
        failure {
            script {
                def duration = currentBuild.durationString.replace(' and counting', '')
                
                discordSend(
                    title: "프론트엔드 빌드 실패! 😭",
                    description: """
                        **빌드 실패!**
                        **제목:** #${env.BUILD_NUMBER}
                        **결과:** ❌ FAILURE
                        **실행 시간:** ${duration}
                        **링크:** [로그 확인하기](${env.BUILD_URL}console)
                    """,
                    result: 'FAILURE',
                    link: env.BUILD_URL,
                    webhookURL: env.DISCORD_URL
                )
            }
        }
    }
}
```

</details>

## 4.2 자동화된 이미지 태그 업데이트

- **구현:** Jenkins 빌드 시마다 매니페스트 리포지토리의 YAML 내 이미지 태그를 자동 수정 (`Update backend image to build #[BUILD_NUMBER]`)
- **효과:** 수동 개입 없는 즉시 배포 및 Git 커밋 로그를 통한 배포 이력 추적 가능

<img width="800" alt="통테4" src="https://github.com/user-attachments/assets/c0cf552b-7ef3-48e3-b671-6a3de609610c" />
<img width="800" alt="통합테스트6" src="https://github.com/user-attachments/assets/3452e036-e41f-48af-90d4-495e85d9a4e5" />

## 4.3 ArgoCD 기반의 무중단 배포

- **구현:** ArgoCD와 EKS 클러스터를 연동하여 Git의 설정과 클러스터 상태를 상시 동기화
- **효과:** 인프라 설정의 일관성 유지 및 장애 발생 시 Git 롤백을 통한 즉각 대응 가능

<img width="1919" height="827" alt="image" src="https://github.com/user-attachments/assets/2b67339a-9703-4c77-90cd-71883fea8c79" />

## 4.4 실시간 알림 시스템

- **구현:** Discord Webhook 연동을 통해 빌드 결과(Success/Failure)를 채널로 전송
- **효과:** 장애 발생 시 가시성 확보 및 팀원 간 실시간 커뮤니케이션 효율 증대

<img width="255" height="185" alt="1" src="https://github.com/user-attachments/assets/09257aea-ad8f-43c2-902f-fce38fb44838" />
<img width="247" height="186" alt="2" src="https://github.com/user-attachments/assets/d1f05298-8305-49a2-8a09-17859bd28752" />  

<br>

# 5️⃣ 인프라 설정 및 보안 (Infrastructure & Security)

클러스터의 안정성과 보안을 극대화하기 위해 다음과 같은 클라우드 환경 설정을 적용하였습니다.

### **🖥️ Compute (AWS EKS)**

- **Managed Node Groups**: AWS에서 관리하는 노드 그룹을 사용하여 인프라 운영 부담을 줄이고 안정성을 확보했습니다.
- **Auto-Scaling**: 트래픽 증량에 따라 유연하게 대응할 수 있는 클러스터 확장 기반을 마련했습니다.

### **🌐 Network & Routing**

- **AWS ALB Controller**: Kubernetes Ingress와 연동하여 외부 트래픽을 효율적으로 제어합니다.
- **ACM (Certificate Manager)**: SSL 인증서를 적용하여 모든 구간에 HTTPS 암호화 통신을 의무화했습니다.

### **🔐 Security & Policy**

- **RBAC / IAM Role**: 최소 권한 원칙에 따라 Jenkins 및 ArgoCD에 필요한 최소한의 AWS 권한(IAM)을 부여했습니다.
- **Private Subnet Deployment**: 주요 리소스를 프라이빗 서브넷에 배치하고, 외부 노출을 차단하여 보안 계층을 강화했습니다.
