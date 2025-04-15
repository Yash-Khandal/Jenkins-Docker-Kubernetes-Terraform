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

        stage('Terraform Init') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    echo "Navigating to Terraform Directory: %TF_WORKING_DIR%"
                    cd %TF_WORKING_DIR%
                    echo "Initializing Terraform..."
                    terraform init
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    echo "Navigating to Terraform Directory: %TF_WORKING_DIR%"
                    cd %TF_WORKING_DIR%
                    terraform plan -out=tfplan -input=false
                    '''
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    script {
                        try {
                            bat '''
                            echo "Navigating to Terraform Directory: %TF_WORKING_DIR%"
                            cd %TF_WORKING_DIR%
                            echo "Applying Terraform Plan..."
                            terraform apply -auto-approve -input=false tfplan
                            '''
                        } catch (Exception e) {
                            echo "Terraform apply failed, attempting to import existing resources..."
                            bat '''
                            cd %TF_WORKING_DIR%
                            terraform import azurerm_resource_group.rg /subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%
                            terraform apply -auto-approve -input=false tfplan
                            '''
                        }
                    }
                }
            }
        }

        stage('Assign ACR Pull Role') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    script {
                        try {
                            bat '''
                            echo "Assigning ACR Pull role to AKS..."
                            az login --service-principal -u %AZURE_CLIENT_ID% -p %AZURE_CLIENT_SECRET% --tenant %AZURE_TENANT_ID%
                            az role assignment create --assignee $(az aks show -g %RESOURCE_GROUP% -n %AKS_CLUSTER% --query identityProfile.kubeletidentity.objectId -o tsv) --scope $(az acr show -g %RESOURCE_GROUP% -n %ACR_NAME% --query id -o tsv) --role AcrPull
                            '''
                        } catch (Exception e) {
                            echo "Failed to assign ACR Pull role automatically"
                            echo "Please assign this role manually in Azure Portal:"
                            echo "1. Go to your ACR resource"
                            echo "2. Navigate to Access Control (IAM)"
                            echo "3. Add role assignment: AcrPull to your AKS cluster's managed identity"
                        }
                    }
                }
            }
        }

        stage('Login to ACR') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    az acr login --name %ACR_NAME% --expose-token
                    '''
                }
            }
        }

        stage('Push Docker Image to ACR') {
            steps {
                bat "docker push ${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Get AKS Credentials') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    az aks get-credentials --resource-group %RESOURCE_GROUP% --name %AKS_CLUSTER% --overwrite-existing
                    '''
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                bat 'kubectl apply -f WebApiJenkins/test.yaml'
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
            script {
                currentBuild.result = 'FAILURE'
            }
        }
    }
}
