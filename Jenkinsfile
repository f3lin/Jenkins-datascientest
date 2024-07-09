pipeline {
    environment {
        DOCKER_ID = "f3lin"
        DOCKER_CAST_IMAGE = "cast-api"
        DOCKER_MOVIE_IMAGE = "movie-api"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        NAMESPACE_LIST = "dev,qa,staging"  // Define a list of namespaces
    }
    agent any
    options {
        timeout(time: 1, unit: 'HOURS')  // Set a timeout for the pipeline execution
        buildDiscarder(logRotator(numToKeepStr: '10'))  // Discard old builds to save space
    }
    stages {

        stage('Checkout') {
            steps {
                script {
                    def branch = env.GIT_BRANCH ?: 'master'

                    echo "Current branch is: ${branch}"
                }
            }
        }

        stage('Docker Build Images') {
            steps {
                script {
                    // Fetch the latest image tags
                    def castLatestTag = sh(script: "docker pull $DOCKER_ID/$DOCKER_CAST_IMAGE:latest || true && docker images $DOCKER_ID/$DOCKER_CAST_IMAGE:latest --format '{{.Tag}}'", returnStdout: true).trim()
                    def movieLatestTag = sh(script: "docker pull $DOCKER_ID/$DOCKER_MOVIE_IMAGE:latest || true && docker images $DOCKER_ID/$DOCKER_MOVIE_IMAGE:latest --format '{{.Tag}}'", returnStdout: true).trim()

                    // Determine new image tags based on the latest tags
                    def newCastTag = "v.${BUILD_ID}.0"
                    def newMovieTag = "v.${BUILD_ID}.0"

                    // Build new images with the new tags
                    sh """
                      docker rm -f jenkins || true
                      docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$newCastTag cast-service/
                      docker build -t $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$newMovieTag movie-service/
                    """

                    // Tag the new images as latest
                    sh """
                      docker tag $DOCKER_ID/$DOCKER_CAST_IMAGE:$newCastTag $DOCKER_ID/$DOCKER_CAST_IMAGE:latest
                      docker tag $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$newMovieTag $DOCKER_ID/$DOCKER_MOVIE_IMAGE:latest
                    """
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
                     docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:latest
                     docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:$DOCKER_TAG
                     docker push $DOCKER_ID/$DOCKER_MOVIE_IMAGE:latest
                    '''
                }
            }
        }

        stage('Helm Deployment to Dev/QA/Staging') {
            environment {
                KUBECONFIG = credentials("config") // assuming you have Kubernetes config stored as a Jenkins credential
            }
            steps {
                script {
                    def namespaces = env.NAMESPACE_LIST.split(',')

                    for (String namespace : namespaces) {
                        // Determine which values.yaml to use based on the namespace
                        def valuesFile = namespace == 'dev' ? 'values.yaml' : "values-${namespace}.yaml"

                        // Helm deployment command using the correct values.yaml for each namespace
                        sh """
                         rm -Rf .kube
                         mkdir .kube
                         ls
                         cat $KUBECONFIG > .kube/config
                         helm upgrade --install exam-app-${namespace} exam/ --namespace ${namespace} --create-namespace --values exam/${valuesFile}
                        """
                    }
                }
            }
        }

        stage('Helm Deployment to Prod') {
            environment {
                KUBECONFIG = credentials("config")
            }
            when {
                expression {
                  def branch = env.GIT_BRANCH ?: 'master'
                  return branch == 'origin/main'
                }
            }
            steps {
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                script {
                    sh '''
                      rm -Rf .kube
                      mkdir .kube
                      ls
                      cat $KUBECONFIG > .kube/config
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
