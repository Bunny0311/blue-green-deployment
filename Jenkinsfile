pipeline {
    agent any

    environment {
        APP_NAME = 'myapp'
        SERVICE_NAME = 'blue-green-service'
        NAMESPACE = 'default'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Green Image') {
            steps {
                sh '''
                    docker build -t myapp:v2 -f Dockerfile green/
                '''
            }
        }

        stage('Load Image into Minikube') {
            steps {
                sh '''
                    minikube image load myapp:v2
                '''
            }
        }

        stage('Deploy Green') {
            steps {
                sh '''
                    kubectl apply -f k8s/deployment-green.yaml
                    kubectl rollout status deployment/myapp-green \
                        --timeout=120s
                '''
            }
        }

        stage('Verify Green') {
            steps {
                sh '''
                    GREEN_POD=$(kubectl get pods \
                        -l app=myapp,color=green \
                        -o jsonpath='{.items[0].metadata.name}')

                    kubectl exec "$GREEN_POD" -- \
                        wget -qO- http://localhost/ \
                        | grep -q 'GREEN ENVIRONMENT'
                '''
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                sh '''
                    kubectl patch service "$SERVICE_NAME" \
                        -p '{"spec":{"selector":{"app":"myapp","color":"green"}}}'
                '''
            }
        }

        stage('Verify Traffic') {
            steps {
                sh '''
                    kubectl get service "$SERVICE_NAME" \
                        -o jsonpath='{.spec.selector}{"\\n"}'
                '''
            }
        }
    }

    post {
        success {
            echo 'Blue-Green deployment completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs before proceeding.'
        }
    }
}
