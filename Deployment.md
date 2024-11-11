#Configure Jenkins job:

![image](https://github.com/user-attachments/assets/0d2c6df9-a82a-4fb9-b9c5-ce824631399f)

#Add your Jenkins Pipeline:

environment {
        AWS_ACCOUNT = '904233099353' // Add your AWS account number here
        ECR_REPO = "${AWS_ACCOUNT}.dkr.ecr.ap-south-1.amazonaws.com/frontend-dev"
        AWS_REGION = 'ap-south-1'
        IMAGE_TAG = "${Release_Version}" // Use BUILD_NUMBER as the tag
        ECS_CLUSTER = 'frontend' // Replace with your ECS cluster name
        ECS_SERVICE = 'frontend-svc' // Replace with your ECS service name
        ECS_TASK_DEF_FAMILY = 'frontend-task' // Replace with your ECS task definition family
        
    }
