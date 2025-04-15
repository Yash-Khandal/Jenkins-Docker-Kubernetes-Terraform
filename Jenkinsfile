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
                    cd %TF_WORKING_DIR%
                    terraform init
                    '''
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                withCredentials([azureServicePrincipal(credentialsId: env.AZURE_CREDENTIALS_ID)]) {
                    bat '''
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
                        // First try normal apply
                        def applyExitCode = bat(
                            script: '''
                            cd %TF_WORKING_DIR%
                            terraform apply -auto-approve -input=false tfplan
                            ''',
                            returnStatus: true
                        )
                        
                        // If failed due to role assignment, do manual assignment
                        if (applyExitCode != 0) {
                            echo "Terraform apply failed, attempting manual role assignment..."
                            
                            // Get AKS object ID and ACR ID
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
                            
                            // Try to assign role (may still fail if no permissions)
                            bat """
                            az role assignment create --assignee ${aksObjectId} --scope ${acrId} --role AcrPull || (
                                echo "WARNING: Failed to assign ACR Pull role automatically"
                                echo "Please assign this role manually in Azure Portal:"
                                echo "1. Go to your ACR resource"
                                echo "2. Navigate to Access Control (IAM)"
                                echo "3. Add role assignment: AcrPull to your AKS cluster's managed identity"
                            )
                            """
                            
                            // Retry Terraform apply
                            bat '''
                            cd %TF_WORKING_DIR%
                            terraform apply -auto-approve -input=false tfplan
                            '''
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
            cleanWs()
        }
        failure {
            echo "Pipeline failed! Check above for errors."
        }
    }
}
