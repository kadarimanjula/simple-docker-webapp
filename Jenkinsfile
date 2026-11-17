pipeline {
  agent any
  
  tools {
      maven 'MVN3'    // name must match a Maven install under Manage  
      jdk    'JDK21'      // name must match a JDK install under Manage
  
  }

  environment {
      DOCKERHUB_CRED_ID = 'dockerhub-credentials'       // Jenkins credential ID (Docker Hub username/token)
      DOCKERHUB_REPO  = 'manjulakadari/simple-docker-webapp'  
      IMAGE_TAG       = "${env.BUILD_NUMBER}"
      CONTAINER_NAME  = 'simple-docker-webapp' 
      APP_PORT        = '8081'  
  }

  options {
      timestamps()
      disableConcurrentBuilds()
  }
  stages {
      stage('Checkout') {
          steps {
             git branch: 'main', url: 'https://github.com/kadarimanjula/simple-docker-webapp.git'
          }
  }
      stage('Build & Test (Maven)') {
          steps {
             sh 'mvn -B clean verify'
          }
        
 }
      stage('SonarQube Analysis') {
         steps {
              withSonarQubeEnv('sonarqube') {   // configured under ManageJenkins > System > SonarQube servers
                  sh '''
                       mvn -B sonar:sonar -Dsonar.projectKey=simple-docker-webapp -Dsonar.projectName=simple-docker-webapp -DskipTests
                      
                  '''
                 }
         }
  }
      stage('Quality Gate') {
          steps {
              timeout(time: 5, unit: 'MINUTES') {
                  waitForQualityGate abortPipeline: true
                 }
         }
  }
       stage('Build Docker Image') {
          steps {
              sh """
                 docker build \
                 -t ${DOCKERHUB_REPO}:${IMAGE_TAG} \
                 -t ${DOCKERHUB_REPO}:latest .
               """
          }
  }
       stage('Push to Docker Hub') {
          steps {
             withCredentials([usernamePassword(
               credentialsId:DOCKERHUB_CRED_ID, 
               usernameVariable: 'DOCKER_USER', 
               passwordVariable:'DOCKER_PASS'
             )]) {
                  sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push manjulakadari:simple-docker-webapp
                      docker push manjulakadari:latest
                      docker logout
                  '''
                 }
          }
  }
       stage('Deploy') {
          steps {
              sh """
                 docker pull simple-docker-webapp:latest
                 docker stop docker-webapp || true
                 docker rm docker-webapp|| true
                 docker run -d --name simple-docker-webapp -p 8081:8080 --restart unless-stopped simple-docker-webapp:latest
              """
          }
  }
       stage('Health Check') {
          steps {
              sh """
                 for i in \$(seq 1 10); do
                     if curl -sf http://localhost:8081/actuator/health; then
                          echo 'App is healthy'
                          exit 0
                     fi
                     echo 'Waiting for app to become healthy...'
                     sleep 5
                 done
                 echo 'Health check failed after retries'
                 exit 1
               """
           }
  }
 }
  post {
     success {
         echo "Built ${DOCKERHUB_REPO}:${IMAGE_TAG}, pushed to Docker Hub, and deployed locally"
     }
      failure {
          echo 'Pipeline failed — check console output above.'
      }
      always {
         sh 'docker image prune -f || true'
      }
  }
} 
