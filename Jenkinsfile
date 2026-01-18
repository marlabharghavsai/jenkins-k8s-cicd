pipeline {
  agent none

  environment {
    APP_NAME = "demo"
    REGISTRY = "localhost:30083"
    IMAGE_NAME = "${REGISTRY}/docker-hosted/${APP_NAME}"
    SONARQUBE_SERVER = "sonarqube"
  }

  stages {

    stage('Checkout') {
      agent { label 'maven' }
      steps {
        checkout scm
      }
    }

    stage('Build & Test') {
      agent { label 'maven' }
      steps {
        container('maven') {
          dir('app/demo') {
            sh '''
              chmod +x mvnw
              ./mvnw clean test
            '''
          }
        }
      }
    }

    stage('SonarQube Analysis') {
      agent { label 'maven' }
      steps {
        container('maven') {
          withSonarQubeEnv("${SONARQUBE_SERVER}") {
            dir('app/demo') {
              sh '''
                chmod +x mvnw
                ./mvnw verify \
                  org.sonarsource.scanner.maven:sonar-maven-plugin:3.11.0.3922:sonar \
                  -Dsonar.projectKey=demo \
                  -Dsonar.projectName=demo
              '''
            }
          }
        }
      }
    }

    stage('Quality Gate') {
      agent none
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Build & Push Docker Image') {
      agent { label 'docker' }
      steps {
        container('docker') {
          withCredentials([usernamePassword(
            credentialsId: 'nexus-creds',
            usernameVariable: 'NEXUS_USER',
            passwordVariable: 'NEXUS_PASS'
          )]) {
            dir('app/demo') {
              sh '''
                docker login ${REGISTRY} -u $NEXUS_USER -p $NEXUS_PASS
                docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                docker push ${IMAGE_NAME}:${BUILD_NUMBER}
              '''
            }
          }
        }
      }
    }

  }

  post {
    success {
      echo "✅ Pipeline completed successfully"
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
