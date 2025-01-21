## Advanced Jenkins 사용 ① - Pipeline

### 1. Delivery Pipeline 사용
___
- Pipeline 생성
  - first-Project(echo) -> Second-Project(git, maven build) -> Third-Project(git, maven build, tomcat 배포)
  - 3가지 item 을 연쇄적으로 실행(빌드 후 조치의 Build other projects 이용)
  - Delivery Pipeline plugin 설치


- My-First-Project 에 빌드 후 조치 Build other projects 에 My-Second-Project 추가
- My-Second-Project 에 빌드 후 조치 Build other projects 에 My-Third-Project 추가


- 새로운 뷰에 My-First-Pipeline 생성(Delivery Pipeline View Type 선택)
- Components -> Initial Job : My-First-Project

<br>

### 2. Jenkins Pipeline 스크립트 사용
___
- Declarative , Scripted(groovy + DSL) 두가지 방식
- 시작 시 유효성 검사 유무, 특정 Stage 실행 가능 여부, 제어문, Option 등의 차이가 있다.


- My-First-Pipeline Item 생성
- Pipeline -> Script
```yaml
pipeline {
    agent any # agent 지정
    stages {
        stage('Compile') {
            steps {
                echo "Compiled successfully!";
            }
        }

        stage('JUnit') {
            steps {
                echo "JUnit passed successfully!";
            }
        }

        stage('Code Analysis') {
            steps {
                echo "Code Analysis completed successfully!";
            }
        }

        stage('Deploy') {
            steps {
                echo "Deployed successfully!";
            }
        }
    }
}
```
- Script 추가 -> Post 스크립트가 끝난 결과에 따른 메시지 확인
```yaml
post {
      always {
        echo "This will always run"
      }
      success {
        echo "This will run when the run finished successfully"
      }
      failure {
        echo "This will run if failed"
      }
      unstable {
        echo "This will run when the run was marked as unstable"
      }
      changed {
        echo "This will run when the state of the pipeline has changed"
      }
    }
```

<br>

### 3. Jenkins Pipeline - Pipeline Syntax 사용
___
- 외부 Git 에 저장된 Script 실행
- Pipeline Syntax 사용 하여 script 생성
```yaml
pipeline {
    agent any
    stages {
        stage('Git clone') {
            steps {
                git 'https://github.com/joneconsulting/jenkins_pipeline_script';
            }
        }

        stage('Compile') {
            steps {
                echo "Compiled successfully!";
                sh './build.sh'
            }
        }

        stage('JUnit') {
            steps {
                echo "JUnit passed successfully!";
                sh './unit.sh'
            }
        }

        stage('Code Analysis') {
            steps {
                echo "Code Analysis completed successfully!";
                sh './quality.sh'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deployed successfully!";
                sh './deploy.sh'
            }
        }
    }
}
```

<br>

### 4. Jenkins Pipeline - Maven build pipeline
___

```yaml
pipeline {
    agent any
    tools { 
      maven 'Maven3.9.9'
    }
    stages {
        stage('github clone') {
            steps {
                git branch: 'main', url: 'https://github.com/xognstl/cicd-web-project.git'; 
            }
        }
        
        stage('build') {
            steps {
                sh '''
                    echo build start
                    mvn clean compile package -DskipTests=true
                '''
            }
        }
    }
}
```
