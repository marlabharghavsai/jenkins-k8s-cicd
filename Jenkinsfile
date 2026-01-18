pipeline {
  agent none

  environment {
    APP_NAME = "demo"
    REGISTRY = "localhost:30083"
    IMAGE_NAME = "${REGISTRY}/docker-hosted/${APP_NAME}"
    SONARQUBE_SERVER = "sonarqube"
    K8S_NAMESPACE = "demo"
  }

  stages {

    /* =======================
       CHECKOUT
       ======================= */
    stage('Checkout') {
      agent { label 'maven' }
      steps {
        checkout scm
      }
    }

    /* =======================
       BUILD & UNIT TEST
       ======================= */
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

    /* =======================
       SONARQUBE ANALYSIS
       ======================= */
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

    /* =======================
       QUALITY GATE
       ======================= */
    stage('Quality Gate') {
      agent none
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    /* =======================
       BUILD & PUSH IMAGE
       ======================= */
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

    /* =======================
       SECURITY SCAN (TRIVY)
       ======================= */
    stage('Security Scan') {
      agent { label 'docker' }
      steps {
        container('docker') {
          sh '''
            docker run --rm \
              -v /var/run/docker.sock:/var/run/docker.sock \
              aquasec/trivy:latest image \
              --exit-code 1 \
              --severity CRITICAL \
              ${IMAGE_NAME}:${BUILD_NUMBER}
          '''
        }
      }
    }

    /* =======================
       DEPLOY TO STAGING
       ======================= */
    stage('Deploy to Staging') {
      agent { label 'docker' }
      steps {
        container('docker') {
          sh '''
            kubectl set image deployment/demo-staging \
              demo=${IMAGE_NAME}:${BUILD_NUMBER} \
              -n ${K8S_NAMESPACE}

            kubectl rollout status deployment/demo-staging \
              -n ${K8S_NAMESPACE}
          '''
        }
      }
    }

    /* =======================
       MANUAL APPROVAL
       ======================= */
    stage('Approve Production') {
      steps {
        input message: 'Approve deployment to PRODUCTION?', ok: 'Deploy'
      }
    }

    /* =======================
       BLUE-GREEN DEPLOY (PROD)
       ======================= */
    stage('Deploy to Production (Blue-Green)') {
      agent { label 'docker' }
      steps {
        lock('production-deploy') {
          container('docker') {
            sh '''
              echo "Deploying NEW version to GREEN environment"

              kubectl set image deployment/demo-green \
                demo=${IMAGE_NAME}:${BUILD_NUMBER} \
                -n ${K8S_NAMESPACE}

              kubectl rollout status deployment/demo-green \
                -n ${K8S_NAMESPACE}

              echo "Switching traffic to GREEN"

              kubectl patch svc demo-prod -n ${K8S_NAMESPACE} \
                -p '{"spec":{"selector":{"app":"demo","version":"green"}}}'
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline completed successfully (Enterprise CI/CD)"
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
