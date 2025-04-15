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
                    script {
                        // Initialize Terraform
                        bat '''
                        cd %TF_WORKING_DIR%
                        terraform init
                        '''

                        // Import existing resources if needed
                        def importCommands = [
                            "azurerm_resource_group.rg": "/subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%",
                            "azurerm_container_registry.acr": "/subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%/providers/Microsoft.ContainerRegistry/registries/%ACR_NAME%",
                            "azurerm_kubernetes_cluster.aks": "/subscriptions/%AZURE_SUBSCRIPTION_ID%/resourceGroups/%RESOURCE_GROUP%/providers/Microsoft.ContainerService/managedClusters/%AKS_CLUSTER%"
                        ]

                        importCommands.each { resource, id ->
                            try {
                                bat """
                                cd %TF_WORKING_DIR%
                                terraform import ${resource} ${id}
                                """
                            } catch (Exception e) {
                                echo "Failed to import ${resource} (may not exist yet or already imported)"
                            }
                        }
                    }
                }
            }
        }

        stage('Terraform Apply') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    script {
                        // Create a new plan
                        bat '''
                        cd %TF_WORKING_DIR%
                        terraform plan -out=tfplan -input=false
                        '''

                        // Apply the plan
                        try {
                            bat '''
                            cd %TF_WORKING_DIR%
                            terraform apply -auto-approve -input=false tfplan
                            '''
                        } catch (Exception e) {
                            echo "Terraform apply failed, attempting manual recovery..."
                            
                            // Handle role assignment separately
                            try {
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
                                az role assignment create --assignee ${aksObjectId} --scope ${acrId} --role AcrPull || (
                                    echo "WARNING: Failed to assign ACR Pull role automatically"
                                    echo "Please assign this role manually in Azure Portal:"
                                    echo "1. Go to your ACR resource"
                                    echo "2. Navigate to Access Control (IAM)"
                                    echo "3. Add role assignment: AcrPull to your AKS cluster's managed identity"
                                )
                                """
                                
                                // Final apply attempt
                                bat '''
                                cd %TF_WORKING_DIR%
                                terraform apply -auto-approve -input=false
                                '''
                            } catch (Exception ex) {
                                error("Failed to complete Terraform apply after recovery attempts")
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to AKS') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    script {
                        // Login to ACR
                        bat '''
                        az acr login --name %ACR_NAME% --expose-token
                        '''
                        
                        // Push Docker image
                        bat "docker push ${ACR_LOGIN_SERVER}/${IMAGE_NAME}:${IMAGE_TAG}"
                        
                        // Get AKS credentials
                        bat '''
                        az aks get-credentials --resource-group %RESOURCE_GROUP% --name %AKS_CLUSTER% --overwrite-existing
                        '''
                        
                        // Deploy to AKS
                        bat 'kubectl apply -f WebApiJenkins/test.yaml'
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed! Check above for errors.'
            script {
                currentBuild.result = 'FAILURE'
            }
        }
    }
}
