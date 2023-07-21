### CI/CD

![ci-cd](https://user-images.githubusercontent.com/98242564/218456353-d969a6bc-9ae0-4678-ab63-47aee338c61f.png)

#### 1.프로젝트 commit and push to remote repository

![push](https://user-images.githubusercontent.com/98242564/218466542-7dbfa9f6-9056-4b53-a246-2e1d57a15271.png)

#### 2. jenkins는 jar 파일 build후 Docker image를 생성 Docker Hub 로  image Push

![jenkins-run](https://user-images.githubusercontent.com/98242564/218466672-2269e228-bbd4-4fb2-b880-6badde47cd97.png)

![jenkins-complete](https://user-images.githubusercontent.com/98242564/218466689-7a25727e-f703-4ce6-b34b-62eefc85d8fd.png)

#### jenkins pipeline

<pre>
<code>
pipeline {
    agent any

    environment {
        imagename = "boripapa/bad-request"
        registryCredential = 'bad-request-docker'
        dockerImage = ''
    }

    stages {
        stage('Prepare') {
          steps {
            echo 'Clonning Repository'
            git url: "git@github.com:ImBoriPapa/bad-request.git",
              branch: 'main',
              credentialsId: 'bad-request-git'
            }
            post {
             success { 
               echo 'Successfully Cloned Repository'
             }
           	 failure {
               error 'pipeline stops here Prepare..check logs'
             }
          }
        }

        stage('Bulid Gradle') {
          steps {
            echo 'Bulid Gradle!'
            dir('.'){
                sh './gradlew clean build'
            }
          }
          post {
            failure {
              error 'pipeline stops here Bulid Gradle..check logs'
            }
          }
        }
        
        stage('Bulid Docker') {
          steps {
            echo 'Bulid Docker!'
            script {
                dockerImage = docker.build imagename
            }
          }
          post {
            failure {
              error 'This pipeline stops here...Bulid Docker..check logs'
            }
          }
        }

        stage('Push Docker') {
          steps {
            echo 'Push Docker!'
            script {
                docker.withRegistry( 'https://registry.hub.docker.com', registryCredential) {
                    dockerImage.push() 
                }
            }
          }
          post {
            failure {
              error 'This pipeline stops here Docker Push...check logs'
            }
          }
        }
        
        stage('Docker Run') {
            steps {
                echo 'Pull Docker Image & Docker Image Run'
                sshagent (credentials: ['bad-request-aws']) {
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@xx.xxx.xxx.xxx 'docker pull boripapa/bad-request'" 
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@xx.xxx.xxx.xxx 'docker rm -f jenkins'"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@xx.xxx.xxx.xxx 'docker run -d --name jenkins -p 8080:8080  -v /home/ec2-user/yml:/home boripapa/bad-request'"
                }
            }
        }
    }
    
}
</code>
</pre>

#### 3. 운영 서버에서 도커이미지를 pull -> jar파일 실행

<pre>
<code>
FROM openjdk:11-jdk

# JAR_FILE 변수 정의 -> 기본적으로 jar file이 2개이기 때문에 이름을 특정해야함
ARG JAR_FILE=./build/libs/bad-request-1.0.0-PROD.jar

# JAR 파일 메인 디렉토리에 복사
COPY ${JAR_FILE} bad-request.jar

# 시스템 진입점 정의
ENTRYPOINT ["java","-Dspring.profiles.active=prod","-jar","/bad-request.jar"]
</code>
</pre>
