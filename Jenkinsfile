pipeline {
    agent any

    environment {
        ACR_NAME = 'acryash20240415'
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
                bat "docker build -t ${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG} -f WebApiJenkins/Dockerfile WebApiJenkins"
            }
        }

        stage('Terraform Setup') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    cd %TF_WORKING_DIR%
                    terraform init
                    '''
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    cd %TF_WORKING_DIR%
                    terraform plan -out=tfplan -input=false
                    terraform apply -auto-approve -input=false tfplan
                    '''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    az acr login --name %ACR_NAME%
                    '''
                    bat "docker push ${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    az aks get-credentials --resource-group %RESOURCE_GROUP% --name %AKS_CLUSTER% --overwrite-existing
                    kubectl apply -f WebApiJenkins/test.yaml
                    '''
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed! Check above logs for details.'
            script {
                currentBuild.result = 'FAILURE'
            }
        }
    }
}
