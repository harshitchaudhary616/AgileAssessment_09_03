
pipeline {
    agent any

    environment {
        PATH = "/Applications/Docker.app/Contents/Resources/bin:/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
        IMAGE = "perkeylynx/college-placement-portal"
        KUBE_NAMESPACE = "default"
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
                sh 'ls -la'
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
                    set -e
                    docker rm -f placement-test 2>/dev/null || true
                    docker run -d --name placement-test -p 18083:80 $IMAGE:$BUILD_NUMBER
                    trap 'docker rm -f placement-test >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 15); do
                        if curl -fsS http://localhost:18083/ -o /tmp/placement-test.html; then
                            break
                        fi
                        sleep 2
                    done

                    grep -q "College Placement Portal" /tmp/placement-test.html
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
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE:$BUILD_NUMBER
                        docker tag $IMAGE:$BUILD_NUMBER $IMAGE:latest
                        docker push $IMAGE:latest
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
                        set -e
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        kubectl config use-context docker-desktop
                        kubectl apply -f deployment.yaml

                        kubectl set image deployment/college-placement-portal \
                          placement-portal=$IMAGE:$BUILD_NUMBER \
                          -n $KUBE_NAMESPACE

                        kubectl rollout status deployment/college-placement-portal \
                          -n $KUBE_NAMESPACE --timeout=180s

                        kubectl get deployment college-placement-portal \
                          -n $KUBE_NAMESPACE

                        kubectl get pods -l app=college-placement-portal \
                          -n $KUBE_NAMESPACE

                        kubectl get service college-placement-portal-service \
                          -n $KUBE_NAMESPACE
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f placement-test 2>/dev/null || true'
        }
        success {
            echo 'College Placement Portal deployed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check Console Output for the failing stage.'
        }
    }
}
