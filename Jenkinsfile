pipeline {
    agent any

    environment {
        ACR_NAME = 'acryash20240415'
        AZURE_CREDENTIALS_ID = 'azure-jenkins'
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

        stage('Terraform Init & Import') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    cd %TF_WORKING_DIR%
                    terraform init
                    terraform import azurerm_resource_group.rg /subscriptions/6c1e198f-37fe-4942-b348-c597e7bef44b/resourceGroups/myResourceGroup || exit 0
                    terraform import azurerm_container_registry.acr /subscriptions/6c1e198f-37fe-4942-b348-c597e7bef44b/resourceGroups/myResourceGroup/providers/Microsoft.ContainerRegistry/registries/acryash20240415 || exit 0
                    terraform import azurerm_kubernetes_cluster.aks /subscriptions/6c1e198f-37fe-4942-b348-c597e7bef44b/resourceGroups/myResourceGroup/providers/Microsoft.ContainerService/managedClusters/myAKSCluster || exit 0
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
                            cd %TF_WORKING_DIR%
                            terraform plan -out=tfplan -input=false
                            terraform apply -auto-approve -input=false tfplan
                            '''
                        } catch (Exception e) {
                            echo "Terraform apply failed, trying role assignment..."
                            def aksObjectId = bat(
                                script: '''
                                az aks show -g %RESOURCE_GROUP% -n %AKS_CLUSTER% --query identityProfile.kubeletidentity.objectId -o tsv
                                ''',
                                returnStdout: true
                            ).trim()

                            def acrId = bat(
                                script: '''
                                az acr show -g %RESOURCE_GROUP% -n %ACR_NAME% --query id -o tsv
                                ''',
                                returnStdout: true
                            ).trim()

                            bat """
                            az role assignment create --assignee ${aksObjectId} --scope ${acrId} --role AcrPull || exit 0
                            """

                            bat '''
                            cd %TF_WORKING_DIR%
                            terraform apply -auto-approve -input=false
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
                    az acr login --name %ACR_NAME%
                    docker push %ACR_LOGIN_SERVER%/%IMAGE_NAME%:%IMAGE_TAG%
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
