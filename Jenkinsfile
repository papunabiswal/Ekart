pipeline {
    agent any

  tools {
        maven "maven3"
        jdk "jdk17"
    }

  environment {
        ECR_REPO = '904233099353.dkr.ecr.ap-south-1.amazonaws.com/sitv2-ui'
        AWS_REGION = 'ap-south-1'
        IMAGE_TAG = "${BUILD_NUMBER}" // Use BUILD_NUMBER as the tag
        ECS_CLUSTER = 'demo' // Replace with your ECS cluster name
        ECS_SERVICE = 'sitv2-ui' // Replace with your ECS service name
        ECS_TASK_DEF_FAMILY = 'demo' // Replace with your ECS task definition family
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'Dev', credentialsId: 'GitHub-Cred', url: 'https://github.com/papunabiswal/Ekart.git'
            }
        }

      tage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Unit test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
      

      // stage('Install Dependencies') {
      //       steps {
      //           script {
      //               // Install dependencies for Angular application
      //               sh 'npm install --force'
      //           }
      //       }
      //   }

      // stage('Run Unit Tests') {
      //       steps {
      //           script {
      //               // Run unit tests and capture the results
      //               // sh 'ng test --watch=false --code-coverage'
      //             sh 'ng test'
      //           }
      //       }
      //       post {
      //           always {
      //               // Publish test results and coverage report
      //               junit 'coverage/**/*.xml'
      //               archiveArtifacts artifacts: 'coverage/**/*', allowEmptyArchive: true
      //           }
      //       }
      //   }

      stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t ${ECR_REPO}:${IMAGE_TAG} -f docker/Dockerfile .
                    """
                }
            }
        }

      stage('Push to ECR') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
            }
            steps {
                script {
                    // Login to ECR and push the image
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                    docker push ${ECR_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }

      stage('Deploy to ECS') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
            }
            steps {
                script {
                    // Update ECS Task Definition with the new image
                    sh """
                    # Fetch current task definition and update with new image
                    TASK_DEF=\$(aws ecs describe-task-definition --task-definition ${ECS_TASK_DEF_FAMILY} --region ${AWS_REGION})
                    NEW_TASK_DEF=\$(echo \$TASK_DEF | jq --arg IMAGE "${ECR_REPO}:${IMAGE_TAG}" '.taskDefinition | .containerDefinitions[0].image = \$IMAGE')
                    NEW_REVISION=\$(echo \$NEW_TASK_DEF | jq '. | {family: .family, containerDefinitions: .containerDefinitions, volumes: .volumes, taskRoleArn: .taskRoleArn, executionRoleArn: .executionRoleArn, networkMode: .networkMode, requiresCompatibilities: .requiresCompatibilities, cpu: .cpu, memory: .memory}')

                    # Register new task definition revision
                    NEW_TASK_DEF_ARN=\$(aws ecs register-task-definition --region ${AWS_REGION} --cli-input-json "\$NEW_REVISION" | jq -r '.taskDefinition.taskDefinitionArn')

                    # Update ECS service to use the new task definition revision
                    aws ecs update-service --region ${AWS_REGION} --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --task-definition \$NEW_TASK_DEF_ARN
                    """
                }
            }
        }
    }

  post {
        always {
            cleanWs()  // Clean up workspace
            sh 'docker image prune -f'  // Remove dangling Docker images
        }
        success {
            echo "Pipeline completed successfully. Docker image pushed to ECR with tag: ${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed. Please check the logs."
        }

}
}

