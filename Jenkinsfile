
pipeline {
    agent any

    environment {
        PATH = "/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
        IMAGE = "perkeylynx/hospital-appointment-portal"
        DEPLOYMENT = "hospital-appointment-portal"
        CONTAINER = "hospital-portal"
        SERVICE = "hospital-appointment-service"
        NAMESPACE = "default"
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
                sh 'test -f index.html && test -f Dockerfile && test -f deployment.yaml'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE:$BUILD_NUMBER .'
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -eu
                    docker rm -f hospital-portal-test >/dev/null 2>&1 || true
                    docker run -d --name hospital-portal-test \
                        -p 18084:80 $IMAGE:$BUILD_NUMBER

                    trap 'docker rm -f hospital-portal-test >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 20); do
                        if curl -fsS http://localhost:18084/ -o /tmp/hospital-test.html; then
                            break
                        fi
                        sleep 2
                    done

                    grep -q "CarePoint Hospital" /tmp/hospital-test.html
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        set +x
                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE:$BUILD_NUMBER
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"
                        kubectl config use-context docker-desktop

                        # Save the currently deployed image for recovery.
                        OLD_IMAGE=$(kubectl get deployment "$DEPLOYMENT" \
                            -n "$NAMESPACE" \
                            -o jsonpath="{.spec.template.spec.containers[?(@.name==\\"$CONTAINER\\")].image}")

                        if [ -z "$OLD_IMAGE" ]; then
                            echo "Deployment or container not found."
                            exit 1
                        fi

                        echo "Previous image: $OLD_IMAGE"

                        # Apply the Deployment and Service configuration.
                        kubectl apply -f deployment.yaml

                        # Update to the new build.
                        kubectl set image deployment/"$DEPLOYMENT" \
                            "$CONTAINER"="$IMAGE:$BUILD_NUMBER" \
                            -n "$NAMESPACE"

                        # Wait for the new version to become healthy.
                        if kubectl rollout status deployment/"$DEPLOYMENT" \
                            -n "$NAMESPACE" --timeout=120s; then

                            echo "Rollout succeeded."
                        else
                            echo "Rollout failed. Restoring previous image."

                            kubectl set image deployment/"$DEPLOYMENT" \
                                "$CONTAINER"="$OLD_IMAGE" \
                                -n "$NAMESPACE"

                            kubectl rollout status deployment/"$DEPLOYMENT" \
                                -n "$NAMESPACE" --timeout=180s

                            echo "Previous image restored."
                            exit 1
                        fi

                        # Check that all three replicas are ready.
                        READY=$(kubectl get deployment "$DEPLOYMENT" \
                            -n "$NAMESPACE" \
                            -o jsonpath='{.status.readyReplicas}')

                        if [ "$READY" != "3" ]; then
                            echo "Expected 3 ready replicas; restoring previous image."

                            kubectl set image deployment/"$DEPLOYMENT" \
                                "$CONTAINER"="$OLD_IMAGE" \
                                -n "$NAMESPACE"

                            kubectl rollout status deployment/"$DEPLOYMENT" \
                                -n "$NAMESPACE" --timeout=180s

                            exit 1
                        fi

                        kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE"
                        kubectl get pods -l app=hospital-portal -n "$NAMESPACE"
                        kubectl get service "$SERVICE" -n "$NAMESPACE"
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f hospital-portal-test >/dev/null 2>&1 || true'
        }
        success {
            echo 'Hospital Appointment Portal deployed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output for the failure and recovery result.'
        }
    }
}
