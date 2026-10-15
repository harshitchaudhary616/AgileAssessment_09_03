
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
                sh 'docker build -t "$IMAGE:$BUILD_NUMBER" .'
            }
        }

        stage('Test Application') {
            steps {
                sh '''
                    set -eu
                    docker rm -f hospital-portal-test >/dev/null 2>&1 || true
                    docker run -d --name hospital-portal-test \
                        -p 18084:80 "$IMAGE:$BUILD_NUMBER"

                    trap 'docker rm -f hospital-portal-test >/dev/null 2>&1 || true' EXIT

                    success=0
                    for i in $(seq 1 20); do
                        if curl -fsS http://localhost:18084/ -o /tmp/hospital-test.html; then
                            success=1
                            break
                        fi
                        sleep 2
                    done

                    test "$success" -eq 1
                    grep -q "CarePoint Hospital" /tmp/hospital-test.html
                    echo "Application test passed."
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
                        docker push "$IMAGE:$BUILD_NUMBER"
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

                        kubectl apply -f deployment.yaml

                        kubectl set image deployment/"$DEPLOYMENT" \
                            "$CONTAINER"="$IMAGE:$BUILD_NUMBER" \
                            -n "$NAMESPACE"

                        kubectl rollout status deployment/"$DEPLOYMENT" \
                            -n "$NAMESPACE" --timeout=120s

                        READY=$(kubectl get deployment "$DEPLOYMENT" \
                            -n "$NAMESPACE" \
                            -o jsonpath='{.status.readyReplicas}')

                        if [ "$READY" != "3" ]; then
                            echo "ERROR: Expected three ready replicas."
                            exit 1
                        fi

                        echo "Stable release deployed with 3 replicas."
                        kubectl get deployment "$DEPLOYMENT" -n "$NAMESPACE"
                        kubectl get pods -l app=hospital-portal -n "$NAMESPACE"
                    '''
                }
            }
        }

        stage('Rollback Demonstration') {
            steps {
                withCredentials([file(
                    credentialsId: 'kuberconfig',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    sh '''
                        set -eu
                        export KUBECONFIG="$KUBECONFIG_FILE"
                        kubectl config use-context docker-desktop

                        STABLE_IMAGE=$(kubectl get deployment "$DEPLOYMENT" \
                            -n "$NAMESPACE" \
                            -o jsonpath='{.spec.template.spec.containers[0].image}')

                        if [ -z "$STABLE_IMAGE" ]; then
                            echo "ERROR: Could not identify the stable image."
                            exit 1
                        fi

                        echo "Previous stable image: $STABLE_IMAGE"
                        echo "Deploying deliberately faulty image..."

                        # This tag is intentionally nonexistent.
                        FAULTY_IMAGE="$IMAGE:faulty-demo"

                        kubectl set image deployment/"$DEPLOYMENT" \
                            "$CONTAINER"="$FAULTY_IMAGE" \
                            -n "$NAMESPACE"

                        if kubectl rollout status deployment/"$DEPLOYMENT" \
                            -n "$NAMESPACE" --timeout=45s; then
                            echo "ERROR: Faulty rollout unexpectedly succeeded."
                            kubectl set image deployment/"$DEPLOYMENT" \
                                "$CONTAINER"="$STABLE_IMAGE" \
                                -n "$NAMESPACE"
                            kubectl rollout status deployment/"$DEPLOYMENT" \
                                -n "$NAMESPACE" --timeout=120s
                            exit 1
                        else
                            echo "EXPECTED FAILURE: Faulty rollout detected."
                        fi

                        echo "Automatically restoring stable release..."

                        kubectl set image deployment/"$DEPLOYMENT" \
                            "$CONTAINER"="$STABLE_IMAGE" \
                            -n "$NAMESPACE"

                        kubectl rollout status deployment/"$DEPLOYMENT" \
                            -n "$NAMESPACE" --timeout=120s

                        READY=$(kubectl get deployment "$DEPLOYMENT" \
                            -n "$NAMESPACE" \
                            -o jsonpath='{.status.readyReplicas}')

                        if [ "$READY" != "3" ]; then
                            echo "ERROR: Stable release did not restore all replicas."
                            exit 1
                        fi

                        echo "Checking restored application..."

                        SUCCESS=0
                        for i in $(seq 1 15); do
                            if curl -fsS http://localhost:30034/ \
                                -o /tmp/hospital-restored.html &&
                                grep -q "CarePoint Hospital" /tmp/hospital-restored.html; then
                                SUCCESS=1
                                break
                            fi
                            sleep 2
                        done

                        if [ "$SUCCESS" -ne 1 ]; then
                            echo "ERROR: Restored application HTTP check failed."
                            exit 1
                        fi

                        echo "SUCCESS: Stable image restored."
                        echo "SUCCESS: Three replicas are ready."
                        echo "SUCCESS: Restored website responds correctly."

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
            echo 'SUCCESS: Deployment, faulty rollout detection, rollback, and recovery verification completed.'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output for the failing stage.'
        }
    }
}
