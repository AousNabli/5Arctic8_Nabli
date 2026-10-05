pipeline {
  agent any

  tools {
    maven 'M3'
    jdk 'JDK21'
  }

  environment {
    DOCKERHUB_USER = 'VOTRE_USER_DOCKERHUB'
    IMAGE = "${DOCKERHUB_USER}/nabliaous_5arctic8_gestionprojets"
  }

  stages {
    stage('Checkout SCM') {
      steps {
        checkout scm
      }
    }

    stage('Build & Test Backend') {
      steps {
        dir('backend') {
          sh 'mvn clean test'
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
          jacoco execPattern: 'backend/target/jacoco.exec',
                 classPattern: 'backend/target/classes',
                 sourcePattern: 'backend/src/main/java'
        }
      }
    }

    stage('Analyse SonarQube') {
      steps {
        dir('backend') {
          withSonarQubeEnv('sonarqube') {
            withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
              sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.8.0.7211:sonar -Dsonar.projectKey=gestion-projets -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml'
            }
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
          dir('backend') {
            withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
              sh '''
                CE_URL=$(grep '^ceTaskUrl=' target/sonar/report-task.txt | cut -d= -f2-)
                for i in $(seq 1 30); do
                  STATUS=$(curl -s -u "$SONAR_TOKEN:" "$CE_URL" | jq -r '.task.status')
                  echo "Statut de l'analyse : $STATUS"
                  [ "$STATUS" = "SUCCESS" ] && break
                  if [ "$STATUS" = "FAILED" ] || [ "$STATUS" = "CANCELED" ]; then exit 1; fi
                  sleep 5
                done
                ANALYSIS_ID=$(curl -s -u "$SONAR_TOKEN:" "$CE_URL" | jq -r '.task.analysisId')
                GATE=$(curl -s -u "$SONAR_TOKEN:" "http://localhost:9000/api/qualitygates/project_status?analysisId=$ANALYSIS_ID" | jq -r '.projectStatus.status')
                echo "Quality Gate : $GATE"
                [ "$GATE" = "OK" ]
              '''
            }
          }
        }
      }
    }

    stage('Package Backend') {
      steps {
        dir('backend') {
          sh 'mvn package -DskipTests'
        }
      }
    }

    stage('Build Frontend') {
      when {
        expression { fileExists('frontend/package.json') }
      }
      steps {
        dir('frontend') {
          sh '''
            docker run --rm -u $(id -u):$(id -g) -e HOME=/tmp \
              -v "$PWD":/app -w /app node:20 \
              sh -c "if [ -f package-lock.json ]; then npm ci; else npm install; fi && npm run build"
          '''
        }
      }
    }

    stage('Archivage') {
      steps {
        archiveArtifacts artifacts: 'backend/target/*.jar, backend/target/site/jacoco/**, frontend/dist/**',
                         allowEmptyArchive: true, fingerprint: true
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker compose build app'
        sh 'docker tag $IMAGE:latest $IMAGE:$BUILD_NUMBER'
      }
    }

    stage('Docker Hub') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh 'echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin'
          sh 'docker push $IMAGE:latest'
          sh 'docker push $IMAGE:$BUILD_NUMBER'
          sh 'docker logout'
        }
      }
    }

    stage('DEBUG Workspace') {
      steps {
        sh '''
          pwd
          ls -la
          ls -la backend/target | head -15
          docker images | head -8
          docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Status}}"
        '''
      }
    }

    stage('Docker Compose') {
      steps {
        sh 'docker compose up -d'
        sh 'docker compose ps'
      }
    }

    stage('Monitoring') {
      steps {
        sh '''
          wait_for() { for i in $(seq 1 30); do curl -sf "$1" >/dev/null && return 0; sleep 5; done; echo "Indisponible : $1"; return 1; }
          wait_for http://localhost:8089/actuator/health
          wait_for http://localhost:9090/-/ready
          wait_for http://localhost:3000/api/health
          echo "Application, Prometheus et Grafana répondent"
        '''
      }
    }
  }
}
