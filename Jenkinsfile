pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    environment {

        // AWS / ECR
        AWS_REGION       = 'eu-north-1'
        AWS_ACCOUNT_ID   = '509989879246'
        ECR_REPOSITORY   = 'terraform-networking-dev-application'
        ECR_REGISTRY     = '509989879246.dkr.ecr.eu-north-1.amazonaws.com'

        // Nexus Docker Registry
        NEXUS_REGISTRY   = '10.10.11.124:8081'
        NEXUS_REPOSITORY = 'crm-docker'

        // Kubernetes / Helm
        K8S_NAMESPACE    = 'crm-dev'
        EKS_CLUSTER_NAME = 'terraform-networking-dev-eks'
        HELM_RELEASE     = 'crm'
        HELM_CHART       = 'deployment/helm/crm'
        HELM_VALUES      = 'deployment/helm/crm/values.yaml'
        HELM_DEV_VALUES  = 'deployment/helm/crm/values-dev.yaml'

        // Keep CD off until Nexus image pulling from EKS is configured
        ENABLE_DEPLOY = 'true'
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

                    echo "===== AWS Identity ====="
                    aws sts get-caller-identity

                    echo "===== SonarQube ====="
                    curl -fsS http://10.10.11.49:9000/api/system/status

                    echo
                    echo "===== Nexus ====="
                    curl -fsS http://${NEXUS_REGISTRY}/service/rest/v1/status
                '''
            }
        }

        stage('Maven Build & Unit Tests') {
            steps {
                sh '''
                    set -e
                    mvn clean verify
                '''
            }
        }

        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('sonarqube') {
            sh '''
                set -e
                mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
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

        stage('Build Docker Images') {
            steps {
                script {

                    def ecrServices = [
                        'gateway-service',
                        'auth-service'
                    ]

                    def nexusServices = [
                        'user-service',
                        'admin-service',
                        'employee-service',
                        'customer-service',
                        'hr-service',
                        'task-service'
                    ]

                    ecrServices.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Building ECR image: ${service}:${imageTag}"

                        sh """
                            docker build \
                              -f ${service}/Dockerfile \
                              -t ${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag} \
                              .
                        """
                    }

                    nexusServices.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Building Nexus image: ${service}:${imageTag}"

                        sh """
                            docker build \
                              -f ${service}/Dockerfile \
                              -t ${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${service}:${imageTag} \
                              .
                        """
                    }
                }
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

        stage('Push ECR Images') {
            steps {
                script {

                    def services = [
                        'gateway-service',
                        'auth-service'
                    ]

                    services.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Pushing ${service} to ECR"

                        sh """
                            docker push \
                              ${ECR_REGISTRY}/${ECR_REPOSITORY}:${imageTag}
                        """
                    }
                }
            }
        }

        stage('Nexus Login') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-credentials',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )
                ]) {

                    sh '''
                        set +x

                        echo "$NEXUS_PASS" | docker login \
                          ${NEXUS_REGISTRY} \
                          --username "$NEXUS_USER" \
                          --password-stdin
                    '''
                }
            }
        }

        stage('Push Nexus Images') {
            steps {
                script {

                    def services = [
                        'user-service',
                        'admin-service',
                        'employee-service',
                        'customer-service',
                        'hr-service',
                        'task-service'
                    ]

                    services.each { service ->

                        def imageTag = "${service}-${BUILD_NUMBER}"

                        echo "Pushing ${service} to Nexus"

                        sh """
                            docker push \
                              ${NEXUS_REGISTRY}/${NEXUS_REPOSITORY}/${service}:${imageTag}
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
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-credentials',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        set -e

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER_NAME}

                        kubectl cluster-info
                        kubectl get nodes

                        kubectl create namespace ${K8S_NAMESPACE} \
                          --dry-run=client -o yaml | kubectl apply -f -

                        kubectl create secret docker-registry nexus-registry-secret \
                          --docker-server=${NEXUS_REGISTRY} \
                          --docker-username="${NEXUS_USER}" \
                          --docker-password="${NEXUS_PASS}" \
                          --namespace=${K8S_NAMESPACE} \
                          --dry-run=client -o yaml | kubectl apply -f -

                        echo "Nexus image pull secret configured."
                    '''
                }
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
                      --set services.gateway.imageTag=gateway-service-${BUILD_NUMBER} \
                      --set services.auth.imageTag=auth-service-${BUILD_NUMBER} \
                      --set services.user.imageTag=user-service-${BUILD_NUMBER} \
                      --set services.admin.imageTag=admin-service-${BUILD_NUMBER} \
                      --set services.employee.imageTag=employee-service-${BUILD_NUMBER} \
                      --set services.customer.imageTag=customer-service-${BUILD_NUMBER} \
                      --set services.hr.imageTag=hr-service-${BUILD_NUMBER} \
                      --set services.task.imageTag=task-service-${BUILD_NUMBER} \
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
            echo 'CRM pipeline failed. Check the failed stage logs.'
        }

        always {
            sh '''
                docker logout ${ECR_REGISTRY} >/dev/null 2>&1 || true
                docker logout ${NEXUS_REGISTRY} >/dev/null 2>&1 || true
            '''
        }
    }
}
