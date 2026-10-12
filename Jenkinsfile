pipeline {
    agent any

    environment {
        APP_NAME     = 'myapp'
        SERVICE_NAME = 'blue-green-service'
        NAMESPACE    = 'default'
        KUBECONFIG   = '/var/lib/jenkins/.kube/config'
        IMAGE_NAME   = 'myapp:v2'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out Blue-Green Deployment project'
                checkout scm
            }
        }

        stage('Build Green Image') {
            steps {
                echo 'Building the Green application image'

                sh '''
                    docker build \
                        -t "$IMAGE_NAME" \
                        -f Dockerfile \
                        green/

                    docker image inspect "$IMAGE_NAME"
                '''
            }
        }

        stage('Load Image into Minikube') {
            steps {
                echo 'Loading the Green image into Minikube'

                sh '''
                    set -eu
		    docker save "$IMAGE_NAME" -o /tmp/myapp-v2.tar
            	    docker cp /tmp/myapp-v2.tar minikube:/myapp-v2.tar
                    docker exec minikube ctr -n k8s.io images import /myapp-v2.tar
                    docker exec minikube rm -f /myapp-v2.tar
                    rm -f /tmp/myapp-v2.tar
                '''
            }
        }

        stage('Deploy Green') {
            steps {
                echo 'Deploying the Green environment'

                sh '''
                    set -eu

                    kubectl apply \
                        --kubeconfig="$KUBECONFIG" \
                        -f k8s/deployment-green.yaml

                    kubectl rollout status \
                        deployment/myapp-green \
                        --kubeconfig="$KUBECONFIG" \
                        --namespace="$NAMESPACE" \
                        --timeout=120s
                '''
            }
        }

        stage('Verify Green') {
            steps {
                echo 'Checking the Green application'

                sh '''
                    set -eu

                    GREEN_POD=$(kubectl get pods \
                        --kubeconfig="$KUBECONFIG" \
                        --namespace="$NAMESPACE" \
                        -l app=myapp,color=green \
                        -o jsonpath='{.items[0].metadata.name}')

                    test -n "$GREEN_POD"

                    kubectl exec \
                        --kubeconfig="$KUBECONFIG" \
                        --namespace="$NAMESPACE" \
                        "$GREEN_POD" -- \
                        wget -qO- http://localhost/ \
                        | grep -q 'GREEN ENVIRONMENT'

                    echo "Green application verification passed"
                '''
            }
        }

        stage('Switch Traffic to Green') {
            steps {
                echo 'Switching Service traffic to Green'

                sh '''
                    set -eu

                    kubectl patch service "$SERVICE_NAME" \
                        --kubeconfig="$KUBECONFIG" \
                        --namespace="$NAMESPACE" \
                        -p '{"spec":{"selector":{"app":"myapp","color":"green"}}}'

                    kubectl get endpoints "$SERVICE_NAME" \
                        --kubeconfig="$KUBECONFIG" \
                        --namespace="$NAMESPACE"
                '''
            }
        }

        stage('Verify Traffic') {
            steps {
                echo 'Verifying that the Service selects Green'

                sh '''
                    set -eu

                    COLOR=$(kubectl get service "$SERVICE_NAME" \
                        --kubeconfig="$KUBECONFIG" \
                        --namespace="$NAMESPACE" \
                        -o jsonpath='{.spec.selector.color}')

                    if [ "$COLOR" != "green" ]; then
                        echo "ERROR: Service is not selecting Green"
                        exit 1
                    fi

                    echo "Traffic selector verified: Green"
                '''
            }
        }
    }

    post {
        failure {
            echo 'Pipeline failed. Attempting to restore traffic to Blue.'

            sh '''
                kubectl patch service "$SERVICE_NAME" \
                    --kubeconfig="$KUBECONFIG" \
                    --namespace="$NAMESPACE" \
                    -p '{"spec":{"selector":{"app":"myapp","color":"blue"}}}' \
                    || echo "WARNING: Automatic rollback failed. Check the Service manually."
            '''
        }

        success {
            echo 'SUCCESS: Blue-Green deployment completed!'
            echo 'Green is now selected by the Kubernetes Service.'
        }

        always {
            echo 'Blue-Green pipeline execution finished.'
        }
    }
}
