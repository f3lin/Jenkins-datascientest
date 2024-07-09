pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID = "f3lin" // replace this with your docker-id
        DOCKER_CAST_IMAGE = "cast-api"
        DOCKER_MOVIE_IMAGE = "movie-api"
        DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    }
    agent any // Jenkins will be able to select all available agents
    stages {
        stage(' Docker Build Cast Image'){ // docker build image stage
            steps {
                script {
                    sh '''
                      docker rm -f jenkins || true
                      docker build -t $DOCKER_ID/DOCKER_CAST_IMAGE:$DOCKER_TAG .
                      sleep 6
                    '''
                }
            }
        }

        stage(' Docker Build Movie Image'){ // docker build image stage
            steps {
                script {
                    sh '''
                      docker rm -f jenkins || true
                      docker build -t $DOCKER_ID/DOCKER_MOVIE_IMAGE:$DOCKER_TAG .
                      sleep 6
                    '''
                }
            }
        }
        stage('Docker run Cast Image'){ // run container from our builded image
            steps {
                script {
                    sh '''
                     docker run -d -p 8002:8000 --name jenkins $DOCKER_ID/DOCKER_CAST_IMAGE:$DOCKER_TAG
                     sleep 10
                    '''
                }
            }
        }

        stage('Docker run Movie Image'){ // run container from our builded image
            steps {
                script {
                    sh '''
                     docker run -d -p 8001:8000 --name jenkins $DOCKER_ID/DOCKER_CAST_IMAGE:$DOCKER_TAG
                     sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                script {
                sh '''
                curl localhost:8801
                '''
                }
            }
        }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                script {
                sh '''
                curl localhost:8802
                '''
                }
            }
        }

        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
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

//         stage('Deploiement en dev'){
//             environment {
//                 KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
//             }
//             steps {
//                 script {
//                 sh '''
//                 rm -Rf .kube
//                 mkdir .kube
//                 ls
//                 cat $KUBECONFIG > .kube/config
//                 cp fastapi/values.yaml values.yml
//                 cat values.yml
//                 sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
//                 helm upgrade --install app fastapi --values=values.yml --namespace dev
//                 '''
//                 }
//             }
//         }
//         stage('Deploiement en staging'){
//             environment {
//                 KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
//             }
//             steps {
//                 script {
//                 sh '''
//                 rm -Rf .kube
//                 mkdir .kube
//                 ls
//                 cat $KUBECONFIG > .kube/config
//                 cp fastapi/values.yaml values.yml
//                 cat values.yml
//                 sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
//                 helm upgrade --install app fastapi --values=values.yml --namespace staging
//                 '''
//                 }
//             }
//         }
//         stage('Deploiement en prod'){
//             environment {
//                 KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
//             }
//             steps {
//                 // Create an Approval Button with a timeout of 15minutes.
//                 // this require a manuel validation in order to deploy on production environment
//                 timeout(time: 15, unit: "MINUTES") {
//                     input message: 'Do you want to deploy in production ?', ok: 'Yes'
//                 }
//
//                 script {
//                 sh '''
//                 rm -Rf .kube
//                 mkdir .kube
//                 ls
//                 cat $KUBECONFIG > .kube/config
//                 cp fastapi/values.yaml values.yml
//                 cat values.yml
//                 sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
//                 helm upgrade --install app fastapi --values=values.yml --namespace prod
//                 '''
//                 }
//             }
//         }
    }
}
