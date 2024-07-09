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
        // Set a timeout for the pipeline execution to avoid hanging builds
        timeout(time: 1, unit: 'HOURS')
        // Discard old builds to save space
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
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
                        // Define namespace-specific values
                        def nodePortMovie
                        def nodePortCast
                        def storage

                        switch (namespace) {
                            case "dev":
                                nodePortMovie = 30011
                                nodePortCast = 30012
                                storage = "100Mi"
                                break
                            case "qa":
                                nodePortMovie = 30013
                                nodePortCast = 30014
                                storage = "100Mi"
                                break
                            case "staging":
                                nodePortMovie = 30015
                                nodePortCast = 30016
                                storage = "150Mi"
                                break
                            default:
                                echo "Namespace ${namespace} not recognized"
                                continue
                        }

                        // Modify values.yaml using sed
                        sh """
                          rm -Rf .kube
                          mkdir .kube
                          ls
                          cat $KUBECONFIG > .kube/config
                          cp exam/values.yaml values.yaml
                         sed -i 's|movie_app:\n *nodePort: .*|movie_app:\n  nodePort: ${nodePortMovie}|g; s|cast_app:\n *nodePort: .*|cast_app:\n  nodePort: ${nodePortCast}|g; s|movie_app:\n *pvc:\n  *storage: .*|movie_app:\n  pvc:\n   storage: ${storage}|g; s|cast_app:\n *pvc:\n  *storage: .*|cast_app:\n  pvc:\n   storage: ${storage}|g' values.yaml
                         helm upgrade --install exam-app-${namespace} exam/ --namespace ${namespace} --create-namespace --values values.yaml
                        """
                    }
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
                timeout(time: 5, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                script {
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
                 docker rmi $DOCKER_ID/$DOCKER_CAST_IMAGE:latest || true
                 docker rmi $DOCKER_ID/$DOCKER_MOVIE_IMAGE:latest || true
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
