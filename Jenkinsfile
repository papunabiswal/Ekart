pipeline {
    agent any
    
    tools {
        maven "maven3"
        jdk "jdk17"
    }
    
    environment {
        AWS_ACCOUNT = '904233099353' // Add your AWS account number here
        ECR_REPO = "${AWS_ACCOUNT}.dkr.ecr.ap-south-1.amazonaws.com/ekart-repo"
        AWS_REGION = 'ap-south-1'
        IMAGE_TAG = "${BUILD_NUMBER}" // Use BUILD_NUMBER as the tag
        ECS_CLUSTER = 'demo' // Replace with your ECS cluster name
        ECS_SERVICE = 'demo' // Replace with your ECS service name
        ECS_TASK_DEF_FAMILY = 'demo' // Replace with your ECS task definition family
        SCANNER_HOME= tool 'sonar'
    }

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'circleci-project-setup', credentialsId: 'GitHub-Cred', url: 'https://github.com/papunabiswal/Ekart.git'
            }
        }
        
        stage('Compile') {
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
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    // sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=Ekart -Dsonar.projectName=ekart"
                    sh "mvn sonar:sonar"
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                    docker build -t ${ECR_REPO}:${IMAGE_TAG} -f Dockerfile .
                    """
                }
            }
        }
        
            
        stage('Tag Docker Image') {
            steps {
                sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}"
            }
        }
 
        stage('Push to ECR') {
            steps {
                withAWS(credentials: 'Aws-Cred', region: 'ap-south-1') {
                    script {
                        sh """
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}
                        docker push ${ECR_REPO}:${IMAGE_TAG}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to ECS') {
            steps {
                withAWS(credentials: 'Aws-Cred', region: 'ap-south-1') {
                    script {
                        // Update ECS Task Definition with the new image
                        sh """
                        sudo apt-get update && sudo apt-get install -y jq
 
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
    }
}
