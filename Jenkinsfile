pipeline {
  agent any

  tools {
    maven 'M3'
    jdk 'JDK21'
  }

  environment {
    // Format imposé : nomprenom_classe_nomProjet, en minuscules
    IMAGE = 'nabliaous_classe_gestionprojets'
  }

  stages {
    stage('Maven') {
      steps {
        dir('backend') {
          sh 'mvn clean install'
          sh 'mvn package -DskipTests'
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
        }
      }
    }

    stage('SonarQube') {
      steps {
        dir('backend') {
          withSonarQubeEnv('sonarqube') {
            sh 'mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=gestion-projets'
          }
        }
      }
    }

    stage('Docker build') {
      steps {
        sh 'docker compose build app'
      }
    }

    stage('Deploy') {
      steps {
        sh 'docker compose up -d'
        sh 'docker compose ps'
      }
    }
  }
}
