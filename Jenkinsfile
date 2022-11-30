pipeline {
  agent { label 'master' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '2'))
  }
  environment {
    DOCKERHUB_CREDENTIALS= credentials('docker-pass')
    AWS_CREDS = credentials('aws-creds')
  }
  stages {
    stage('Tooling versions') {
      steps {
        sh '''
          docker --version
          docker compose version
        '''
      }
    }
    stage('Build Docker Image'){
      steps{
        sh 'docker context use default'
        sh 'docker build -t vishnevskiyav/java-maven-app:$BUILD_NUMBER .'   
      }
    }
    stage('Login to Docker Hub') {      	
      steps{                       	
        sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'                		   
      }           
    }
    stage('Docker Container Artifactory'){
      steps{
        sh 'docker context use default'
        sh 'docker push vishnevskiyav/java-maven-app:$BUILD_NUMBER'
      } 
    }
    stage('Deploy') {
      steps {
        sh 'docker context create myecscontext --from-env'
        sh 'docker context use myecscontext'
        sh 'docker compose up'
        sh 'docker compose ps --format json'
      }
    }
  }
  post {
    always {
      sh 'docker context use default'
      sh 'docker logout'
    }
  }
}
