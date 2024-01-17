pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "hmatondo"  //DockerHub useraccount
DOCKER_CAST_IMAGE = "jenkins_devops_exams_test-cast_service-1"
DOCKER_MOVIES_IMAGE = "jenkins_devops_exam_test-movie_service-1"
DOCKER_CAST_DB_IMAGE = "postgres:12.1-alpine"
DOCKER_MOVIE_DB_IMAGE = "postgres:12.1-alpine"
DOCKER_TAG = "v.${BUILD_ID}.0" // Tag our image with the current build in order to increment the value by 1 with each new build
HELM_PATH = "/usr/local/bin/helm" // In my case it is important to specify helm path ; if none Jenkins will not find tool
}
agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                  docker rm -f jenkins_devop_exam_test-cast_db-1 || true
                  docker rm -f jenkins_devop_exam_test-movie_db-1 || true
                  docker rm -f jenkins_devop_exam_test-cast_service-1 || true
                  docker rm -f jenkins_devop_exam_test-movie_service-1 || true
                  docker compose up -d
                  sleep 6
                  docker build -t $DOCKER_ID/$DOCKER_CAST_DB_IMAGE:$DOCKER_TAG .
                  docker build -t $DOCKER_ID/$DOCKER_MOVIES_DB_IMAGE:$DOCKER_TAG .
                  docker build -t $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG .
                  docker build -t $DOCKER_ID/$DOCKER_MOVIES_IMAGE:$DOCKER_TAG .
                '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''             
                    docker run -d -p 81:80 --name jenkins_devop_exam_test-movie_service-1 $DOCKER_ID/$DOCKER_MOVIES_IMAGE:$DOCKER_TAG
                    docker run -d -p 82:80 --name jenkins_devop_exam_test-cast_service-1 $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG
                    docker run -d -p 83:80 --name jenkins_devop_exam_test-cast_db-1 $DOCKER_ID/$DOCKER_CAST_DB_IMAGE:$DOCKER_TAG
                    docker run -d -p 84:80 --name jenkins_devop_exam_test-movie_db-1 $DOCKER_ID/$DOCKER_MOVIE_DB_IMAGE:$DOCKER_TAG
                    sleep 10
                    '''
                    }
                }
        }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl localhost
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_CAST_DB_IMAGE:$DOCKER_TAG .
                docker push $DOCKER_ID/$DOCKER_MOVIES_DB_IMAGE:$DOCKER_TAG .
                docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG .
                docker push $DOCKER_ID/$DOCKER_MOVIES_IMAGE:$DOCKER_TAG .
                '''
                }
            }

        }

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config_Jenkins_exam_test") // we retrieve kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                $HELM_PATH upgrade --install cast-service movie-service fastapi --values=values.yml --namespace dev
                '''
                }
            }

        }

stage('Deploiement en QA'){
        environment
        {
        KUBECONFIG = credentials("config_Jenkins_exam_test") // we retrieve kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                $HELM_PATH upgrade --install cast-service movie-service fastapi --values=values.yml --namespace qa
                '''
                }
            }

        }

stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config_Jenkins_exam_test") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                $HELM_PATH upgrade --install cast-service movie-service fastapi --values=values.yml --namespace staging
                '''	 
                }
            }

        }
  stage('Deploiement en prod'){
        environment
        {
        KUBECONFIG = credentials("config_Jenkins_exam_test") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp fastapi/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                $HELM_PATH upgrade --install cast-service movie-service fastapi --values=values.yml --namespace prod
                '''
                }
            }

        }
}
post {
    success {
        echo "This will run if the job succeeded"
        mail to: "hermann.acm@gmail.com",
             subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} is success",
             body: "Hi, this is Hermann, pipeline build was realized with success. Thanks you."
    }
}
}
