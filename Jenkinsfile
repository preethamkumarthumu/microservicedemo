pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {

        // AWS
        AWS_REGION = 'eu-north-1'
        AWS_ACCOUNT_ID = '509989879246'

        // ECR
        ECR_REPOSITORY = 'terraform-networking-dev-application'
        ECR_REGISTRY = '509989879246.dkr.ecr.eu-north-1.amazonaws.com'

        // Kubernetes / Helm
        K8S_NAMESPACE = 'crm-dev'
        HELM_RELEASE = 'crm'
        HELM_CHART = 'deployment/helm/crm'
        HELM_VALUES = 'deployment/helm/crm/values.yaml'
        HELM_DEV_VALUES = 'deployment/helm/crm/values-dev.yaml'

        // CD is disabled until the Terraform EKS cluster is available
        ENABLE_DEPLOY = 'false'
        EKS_CLUSTER_NAME = ''
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Environment Validation') {
            steps {
                sh '''
                    set -e

                    echo "===== Java ====="
                    java -version

                    echo "===== Maven ====="
                    mvn -version

                    echo "===== Docker ====="
                    docker --version

                    echo "===== AWS CLI ====="
                    aws --version

                    echo "===== kubectl ====="
                    kubectl version --client

                    echo "===== Helm ====="
                    helm version --short

                    echo "===== AWS Identity ====="
                    aws sts get-caller-identity
                '''
            }
        }

        stage('Maven Build & Test') {
            steps {
                sh '''
                    set -e
                    mvn clean verify
                '''
            }
        }

        /*
         * SonarQube stages are temporarily disabled.
         *
         * Enable these after Jenkins <-> SonarQube
         * configuration is completed.
         *
         * Do not use the existing root sonar-project.properties
         * because it currently points to tenantCrm rather than
         * this Maven reactor.
         */

        /*
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=crm-microservices \
                          -Dsonar.projectName="CRM Microservices"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        */

        stage('Helm Validation') {
            steps {
                sh '''
                    set -e

                    helm lint ${HELM_CHART} \
                      -f ${HELM_VALUES} \
                      -f ${HELM_DEV_VALUES}

                    helm template ${HELM_RELEASE} ${HELM_CHART} \
                      -f ${HELM_VALUES} \
                      -f ${HELM_DEV_VALUES} \
                      > /tmp/crm-rendered.yaml
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    set -e

                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                    | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                script {

                    def services = [
                        'auth-service',
                        'user-service',
                        'lead-service',
                        'customer-service',
                        'contact-service',
                        'opportunity-service',
                        'quotation-service',
                        'invoice-service',
                        'task-service',
                        'notification-service',
                        'file-service',
                        'report-service',
                        'audit-service',
                        'gateway-service'
                    ]

                    services.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Building ${service}:${imageTag}"

                        sh """
                            docker build \
                              -f ${service}/Dockerfile \
                              -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag} \
                              .
                        """
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {

                    def services = [
                        'auth-service',
                        'user-service',
                        'lead-service',
                        'customer-service',
                        'contact-service',
                        'opportunity-service',
                        'quotation-service',
                        'invoice-service',
                        'task-service',
                        'notification-service',
                        'file-service',
                        'report-service',
                        'audit-service',
                        'gateway-service'
                    ]

                    services.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Pushing ${service}:${imageTag}"

                        sh """
                            docker push \
                              ${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag}
                        """
                    }
                }
            }
        }

        stage('Configure EKS') {

            when {
                environment name: 'ENABLE_DEPLOY', value: 'true'
            }

            steps {
                sh '''
                    set -e

                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER_NAME}

                    kubectl cluster-info
                '''
            }
        }

        stage('Deploy with Helm') {

            when {
                environment name: 'ENABLE_DEPLOY', value: 'true'
            }

            steps {
                sh '''
                    set -e

                    helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                      --namespace ${K8S_NAMESPACE} \
                      --create-namespace \
                      -f ${HELM_VALUES} \
                      -f ${HELM_DEV_VALUES} \
                      --set services.auth.imageTag=auth-service-${BUILD_NUMBER} \
                      --set services.user.imageTag=user-service-${BUILD_NUMBER} \
                      --set services.lead.imageTag=lead-service-${BUILD_NUMBER} \
                      --set services.customer.imageTag=customer-service-${BUILD_NUMBER} \
                      --set services.contact.imageTag=contact-service-${BUILD_NUMBER} \
                      --set services.opportunity.imageTag=opportunity-service-${BUILD_NUMBER} \
                      --set services.quotation.imageTag=quotation-service-${BUILD_NUMBER} \
                      --set services.invoice.imageTag=invoice-service-${BUILD_NUMBER} \
                      --set services.task.imageTag=task-service-${BUILD_NUMBER} \
                      --set services.notification.imageTag=notification-service-${BUILD_NUMBER} \
                      --set services.file.imageTag=file-service-${BUILD_NUMBER} \
                      --set services.report.imageTag=report-service-${BUILD_NUMBER} \
                      --set services.audit.imageTag=audit-service-${BUILD_NUMBER} \
                      --set services.gateway.imageTag=gateway-service-${BUILD_NUMBER} \
                      --wait \
                      --timeout 10m
                '''
            }
        }

        stage('Verify Deployment') {

            when {
                environment name: 'ENABLE_DEPLOY', value: 'true'
            }

            steps {
                sh '''
                    set -e

                    echo "===== Deployments ====="
                    kubectl get deployments -n ${K8S_NAMESPACE}

                    echo "===== Pods ====="
                    kubectl get pods -n ${K8S_NAMESPACE}

                    echo "===== Services ====="
                    kubectl get svc -n ${K8S_NAMESPACE}

                    echo "===== Ingress ====="
                    kubectl get ingress -n ${K8S_NAMESPACE}
                '''
            }
        }
    }

    post {

        success {
            echo 'CRM CI pipeline completed successfully.'
        }

        failure {
            echo 'CRM CI pipeline failed. Check the failed stage logs.'
        }

        always {
            sh '''
                docker logout ${ECR_REGISTRY} >/dev/null 2>&1 || true
            '''
        }
    }
}
