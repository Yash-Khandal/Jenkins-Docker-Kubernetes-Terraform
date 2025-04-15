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
                    echo "Initializing Terraform..."
                    cd %TF_WORKING_DIR%
                    terraform init
                    '''
                }
            }
        }

        stage('Import Existing Resources') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    script {
                        try {
                            bat '''
                            cd %TF_WORKING_DIR%
                            echo "Importing existing resource group..."
                            terraform import azurerm_resource_group.rg /subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%
                            '''
                        } catch (Exception e) {
                            echo "Resource group import failed (may not exist yet)"
                        }
                        
                        try {
                            bat '''
                            cd %TF_WORKING_DIR%
                            echo "Importing existing ACR..."
                            terraform import azurerm_container_registry.acr /subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%/providers/Microsoft.ContainerRegistry/registries/%ACR_NAME%
                            '''
                        } catch (Exception e) {
                            echo "ACR import failed (may not exist yet)"
                        }
                        
                        try {
                            bat '''
                            cd %TF_WORKING_DIR%
                            echo "Importing existing AKS cluster..."
                            terraform import azurerm_kubernetes_cluster.aks /subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%/providers/Microsoft.ContainerService/managedClusters/%AKS_CLUSTER%
                            '''
                        } catch (Exception e) {
                            echo "AKS cluster import failed (may not exist yet)"
                        }
                    }
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    cd %TF_WORKING_DIR%
                    echo "Creating Terraform plan..."
                    terraform plan -out=tfplan -input=false
                    '''
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    cd %TF_WORKING_DIR%
                    echo "Applying Terraform plan..."
                    terraform apply -auto-approve -input=false tfplan
                    '''
                }
            }
        }

        stage('Assign ACR Pull Role') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    powershell '''
                    Write-Host "Assigning ACR Pull role..."
                    $aksObjectId = $(az aks show -g $env:RESOURCE_GROUP -n $env:AKS_CLUSTER --query identityProfile.kubeletidentity.objectId -o tsv)
                    $acrId = $(az acr show -g $env:RESOURCE_GROUP -n $env:ACR_NAME --query id -o tsv)
                    az role assignment create --assignee $aksObjectId --scope $acrId --role AcrPull
                    '''
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
