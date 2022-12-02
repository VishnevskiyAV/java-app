pipeline {
  agent { label 'master' }
  options {
    buildDiscarder(logRotator(numToKeepStr: '2'))
  }
  environment {
    AWS_ECR_REGION = 'eu-central-1'
    AWS_ECS_SERVICE = 'staging'
    AWS_ECS_TASK_DEFINITION = 'staging'
    AWS_ECS_COMPATIBILITY = 'EC2'
    AWS_ECS_CPU = '256'
    AWS_ECS_MEMORY = '512'
    AWS_ECS_CLUSTER = 'Jenkins-ECS-cluster'
    AWS_ECS_TASK_DEFINITION_PATH = './ecs/container-definition-update-image.json'
    }

  stages {
    stage('Tooling versions') {
      steps {
        sh '''
          docker --version
          docker compose version
          aws --version
        '''
      }
    }
    stage('Build Docker Image') {
        steps {
            withCredentials([string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
              echo "My secret text is '${AWS_ECR_URL}'"
                script {
                    docker.build("${AWS_ECR_URL}:${BUILD_NUMBER} .")
                }
            }
        }
    }
    stage('Push image to ECR') {
        steps {
            withCredentials([string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
                withAWS(region: "${AWS_ECR_REGION}", credentials: 'aws-creds') {
                    script {
                        def login = ecrLogin()
                        sh('#!/bin/sh -e\n' + "${login}") // hide logging
                        docker.image("${AWS_ECR_URL}:${BUILD_NUMBER}").push()
                    }
                }
            }
        }
    }
    stage('Deploy in ECS') {
      steps {
        withCredentials([string(credentialsId: 'AWS_EXECUTION_ROL_SECRET', variable: 'AWS_ECS_EXECUTION_ROL'),string(credentialsId: 'AWS_REPOSITORY_URL_SECRET', variable: 'AWS_ECR_URL')]) {
          script {
            updateContainerDefinitionJsonWithImageVersion()
            sh("/usr/local/bin/aws ecs register-task-definition --region ${AWS_ECR_REGION} --family ${AWS_ECS_TASK_DEFINITION} --execution-role-arn ${AWS_ECS_EXECUTION_ROL} --requires-compatibilities ${AWS_ECS_COMPATIBILITY}  --cpu ${AWS_ECS_CPU} --memory ${AWS_ECS_MEMORY} --container-definitions file://${AWS_ECS_TASK_DEFINITION_PATH}")
            def taskRevision = sh(script: "/usr/local/bin/aws ecs describe-task-definition --task-definition ${AWS_ECS_TASK_DEFINITION} | egrep \"revision\" | tr \"/\" \" \" | awk '{print \$2}' | sed 's/\"\$//'", returnStdout: true)
            sh("/usr/local/bin/aws ecs update-service --cluster ${AWS_ECS_CLUSTER} --service ${AWS_ECS_SERVICE} --task-definition ${AWS_ECS_TASK_DEFINITION}:${taskRevision}")
          }
        }
      }
    }
  }
  post {
    always {
        echo 'Completed'
     }
    }
}