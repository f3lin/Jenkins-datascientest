pipeline {
    environment {
        DOCKER_ID = "f3lin"
        DOCKER_CAST_IMAGE = "cast-api"
        DOCKER_MOVIE_IMAGE = "movie-api"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    agent any
    options {
        // Set a timeout for the pipeline execution to avoid hanging builds
        timeout(time: 1, unit: 'HOURS')
        // Discard old builds to save space
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage('Docker Build Images') {
            steps {
                script {
                    sh '''
                      docker rm -f jenkins || true
                      docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG cast-service/
                      docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG movie-service/
                    '''
                }
            }
        }

        stage('Docker Run Images') {
            steps {
                script {
                    sh '''
                     docker run -d -p 8002:8000 --name jenkins-cast $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                     docker run -d -p 8001:8000 --name jenkins-movie $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Docker Test Containers') {
            steps {
                script {
                    sh '''
                     # Test if cast-service is running
                     if ! docker ps -a | grep -q jenkins-cast; then
                        echo "Cast service container failed to start."
                        exit 1
                     fi

                     # Test if movie-service is running
                     if ! docker ps -a | grep -q jenkins-movie; then
                        echo "Movie service container failed to start."
                        exit 1
                     fi

                     echo "Both containers are running successfully."
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                     docker login -u $DOCKER_ID -p $DOCKER_PASS
                     docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                     docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }

        stage('Helm Deployment to Dev/QA/Staging') {
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
            steps {
                script {
                    sh '''
                     rm -Rf .kube
                     mkdir .kube
                     ls
                     cat $KUBECONFIG > .kube/config
                     # Set the namespace for Helm deployment
                     NAMESPACE=dev

                     # Check for the branch name and set the namespace accordingly
                     if [ "${BRANCH_NAME}" == "qa" ]; then
                         NAMESPACE=qa
                     elif [ "${BRANCH_NAME}" == "staging" ]; then
                         NAMESPACE=staging
                     fi

                     # Deploy with Helm using values.yaml
                     helm upgrade --install exam-app exam/ --namespace $NAMESPACE --create-namespace --values exam/values.yaml
                    '''
                }
            }
        }

        stage('Helm Deployment to Prod') {
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
            when {
                branch 'main'
            }
            steps {
                script {
                    timeout(time: 1, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production?', ok: 'Yes'
                    }
                    sh '''
                     rm -Rf .kube
                     mkdir .kube
                     ls
                     cat $KUBECONFIG > .kube/config
                     # Deploy with Helm using values-prod.yaml
                     helm upgrade --install exam-app exam/ --namespace prod --create-namespace --values exam/values-prod.yaml
                    '''
                }
            }
        }
    }
    post {
        always {
            script {
                // Cleanup Docker containers
                sh '''
                 docker rm -f jenkins-cast jenkins-movie || true
                 docker rmi $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG || true
                 docker rmi $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG || true
                '''
            }
        }
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
