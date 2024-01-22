pipeline {
    environment { // Declaration of environment variables
        DOCKER_ID = "hmatondo"  //DockerHub useraccount
        DOCKER_CAST_IMAGE = "jenkins_devops_exam_test-cast_service"    // Cast service image
        DOCKER_MOVIES_IMAGE = "jenkins_devops_exam_test-movie_service"    // Movie service image
        DOCKER_CAST_DB_IMAGE = "postgres:12.1-alpine"    // Cast database image
        DOCKER_MOVIE_DB_IMAGE = "postgres:12.1-alpine"    // Movie database image
        DOCKER_TAG = "v.${BUILD_ID}.0" // Tag our api(s) images with the current build in order to increment the value by 1 with each new build
        HELM_PATH = "/usr/local/bin/helm" // In my case it is important to specify helm path ; if none Jenkins will not find tool
    }
    agent any // Jenkins will be able to select all available agents
    parameters {
        string(name: 'user', defaultValue: 'Please past your name', description: 'Pipeline CI/CD is realized using few tools such as Git, GitHub, Jenkins, Docker, Kubernetes, k3s.')    //  Using paramters too start pipeline
    }

    stages {
        stage('Debug Docker Build'){
            steps {
                script {
                    echo "START PIPELINE JENKINS DEVOPS EXAM TEST"
                    echo "DOCKER_ID: $DOCKER_ID"
                    echo "DOCKER_TAG: $DOCKER_TAG"
                    echo "DOCKER_CAST_IMAGE: $DOCKER_CAST_IMAGE"
                    echo "DOCKER_MOVIES_IMAGE: $DOCKER_MOVIES_IMAGE"
                    echo "DOCKER_CAST_DB_IMAGE: $DOCKER_CAST_DB_IMAGE"
                    echo "DOCKER_MOVIE_DB_IMAGE: $DOCKER_MOVIE_DB_IMAGE"
                    echo 'STAGE SUCCESS : Debug Docker Build'
                }
            }
        }

        stage('Docker Compose -> Build and Run'){ // docker build image stage
            steps {
                script {
                sh '''
                  docker rm -f jenkins_devops_exam_test-cast_db-1    // Remove cast database
                  docker rm -f jenkins_devops_exam_test-movie_db-1    // Remove movie database
                  docker rm -f jenkins_devops_exam_test-movie_service-1    // Remove movie service
                  docker rm -f jenkins_devops_exam_test-cast_service-1    // Remove cast service
                  docker rm -f jenkins_devops_exam_test-nginx-1    // Remove nginx server service
                  docker compose up -d    // Launch docker compose script to build api
                  echo 'STAGE SUCCESS : Docker Compose'
                  sleep 6
                '''
                }
            }
        }

        stage('Docker Tag Images'){ // docker tag image to be able to push them to DockerHub
            steps {
                script {
                sh '''
                  docker ps    // Check running containers
                  docker images    // Check images already downloaded
                  docker tag jenkins_devops_exam_test-cast_service $DOCKER_ID/jenkins_devops_exam_test-cast_service:$DOCKER_TAG    // Tag cast service image
                  docker tag jenkins_devops_exam_test-movie_service $DOCKER_ID/jenkins_devops_exam_test-movie_service:$DOCKER_TAG    // Tag movie service image
                  echo 'STAGE SUCCESS : Docker Tag'
                  sleep 4
                '''
                }
            }
        }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                      curl localhost    // HTTP Request/Response Test to local working environment ; web app test without needing a server ; You can already replace localhost with your server IP address http://IP_ADDRESS 
                      echo "STAGE SUCCESS : Test d'acceptation réussi"
                    '''
                    }
            }

        }

        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                  docker login -u $DOCKER_ID -p $DOCKER_PASS    // Log to DockerHub account
                  docker push $DOCKER_ID/$DOCKER_CAST_IMAGE:$DOCKER_TAG    // Push cast service image
                  docker push $DOCKER_ID/$DOCKER_MOVIES_IMAGE:$DOCKER_TAG    // Push movie service image
                  echo 'STAGE SUCCESS : Docker Push - Images des applications disponibles sur DockerHub'
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
                  rm -Rf .kube    // Remove existing kube folder to avoid conflicts
                  mkdir .kube    // Create a new kube folder
                  ls -alt    // Check if kube folder was correctly created   
                  cat $KUBECONFIG > .kube/config    // Write kube config in a kube folder config file
                  cp fastapi/values.yaml values.yml    // Copy cast fastapi configuration
                  cp movieapi/movie-values.yaml movie-values.yaml    // Copy movie fastapi configuration
                  cat values.yml    // See cast fastapi config
                  cat movie-values.yaml    // See movie fastapi config
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml    // Change tag value in cast fastapi yaml config file
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yaml    // Change tag value in movie fastapi yaml config file
                  $HELM_PATH upgrade --install cast-service ./fastapi --values=values.yml --namespace dev    
                  $HELM_PATH upgrade --install movie-service ./movieapi --values=movie-values.yaml --namespace dev
                  echo "STAGE SUCCESS : Deploiement sur l'environnement de développement effectué !"
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
                  rm -Rf .kube    // Remove existing kube folder to avoid conflicts
                  mkdir .kube    // Create a new kube folder
                  ls -alt    // Check if kube folder was correctly created
                  cat $KUBECONFIG > .kube/config    // Write kube config in a kube folder config file
                  cp fastapi/values.yaml values.yml    // Copy cast fastapi configuration
                  cp movieapi/movie-values.yaml movie-values.yaml    // Copy movie fastapi configuration
                  cat values.yml
                  cat movie-values.yaml
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yaml
                  $HELM_PATH upgrade --install cast-service ./fastapi --values=values.yml --namespace qa
                  $HELM_PATH upgrade --install movie-service ./movieapi --values=movie-values.yaml --namespace qa
                  echo "STAGE SUCCESS : Deploiement sur l'environnement de test effectué !"
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
                  cp movieapi/movie-values.yaml movie-values.yaml
                  cat values.yml
                  cat movie-values.yaml
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yaml
                  $HELM_PATH upgrade --install cast-service ./fastapi --values=values.yml --namespace staging
                  $HELM_PATH upgrade --install movie-service ./movieapi --values=movie-values.yaml --namespace staging
                  echo "STAGE SUCCESS : Deploiement sur l'environnement de pré-poduction réussi !"
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
                  echo "USER APPROBATION SUCCESS : ${DOCKER_ID} give his/her authorization to deploy on prod environment"
                  rm -Rf .kube
                  mkdir .kube
                  ls
                  cat $KUBECONFIG > .kube/config
                  cp fastapi/values.yaml values.yml
                  cp movieapi/movie-values.yaml movie-values.yaml
                  cat values.yml
                  cat movie-values.yaml
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                  sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" movie-values.yaml
                  $HELM_PATH upgrade --install cast-service ./fastapi --values=values.yml --namespace prod
                  $HELM_PATH upgrade --install movie-service ./movieapi --values=movie-values.yaml --namespace prod
                  echo "STAGE SUCCESS : Deploiement sur l'environnement de production réussi !"
                  echo "END : Pipeline realized and triggered by ${params.user}. Thank you for your courses, tips and advices! So Grateful !!"
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
             body: "Hi, this is Hermann, Jenkins pipeline was realized with success."
    }
}
}
