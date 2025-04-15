pipeline {
    agent any

    environment {
        ACR_NAME = 'acryash01'
        AZURE_CREDENTIALS_ID = 'azure-credentials'
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = 'webapidocker1'
        IMAGE_TAG = 'latest'
        RESOURCE_GROUP = 'myResourceGroup'
        AKS_CLUSTER = 'myAKSCluster'
        TF_WORKING_DIR = '.'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Yash-Khandal/Jenkins-Docker-Kubernetes-Terraform.git'
            }
        }

        stage('Build .NET App') {
            steps {
                bat 'dotnet publish WebApiJenkins/WebApiJenkins.csproj -c Release -o out'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t %ACR_LOGIN_SERVER%/%IMAGE_NAME%:%IMAGE_TAG% -f WebApiJenkins/Dockerfile WebApiJenkins"
            }
        }

       stage('Terraform Init') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: AZURE_CREDENTIALS_ID)]) {
                    bat """
                    echo "Navigating to Terraform Directory: %TF_WORKING_DIR%"
                    cd %TF_WORKING_DIR%
                    echo "Initializing Terraform..."
                    terraform init
                    """
                }
            }
        }

        stage('Terraform Plan') {
    steps {
        withCredentials([azureServicePrincipal(credentialsId: AZURE_CREDENTIALS_ID)]) {
            bat """
            echo "Navigating to Terraform Directory: %TF_WORKING_DIR%"
            cd %TF_WORKING_DIR%
            terraform plan -out=tfplan
            """
        }
    }
}


        stage('Terraform Apply') {
    steps {
        withCredentials([azureServicePrincipal(credentialsId: AZURE_CREDENTIALS_ID)]) {
            bat """
            echo "Navigating to Terraform Directory: %TF_WORKING_DIR%"
            cd %TF_WORKING_DIR%
            echo "Applying Terraform Plan..."
            terraform apply -auto-approve tfplan
            """
        }
    }
}
        stage('Login to ACR') {
            steps {
                bat "az acr login --name %ACR_NAME%"
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                bat "docker push %ACR_LOGIN_SERVER%/%IMAGE_NAME%:%IMAGE_TAG%"
            }
        }

        stage('Get AKS Credentials') {
            steps {
                bat "az aks get-credentials --resource-group %RESOURCE_GROUP% --name %AKS_CLUSTER% --overwrite-existing"
            }
        }

        stage('Deploy to AKS') {
            steps {
                bat "kubectl apply -f WebApiJenkins/test.yaml"
            }
        }
    }

    post {
        success {
            echo 'All stages completed successfully!'
        }
        failure {
            echo 'Build failed.'
        }
    }
}
