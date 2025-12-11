pipeline {
    agent any

    environment {
        // Mettez ici votre identifiant DockerHub
        registry = "chriskrack"
        // L'ID que nous avons créé dans Jenkins
        registryCredential = 'dockerhub-auth'
        // Tag simple basé sur le numéro de build (1, 2, 3...)
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Recuperation du Code') {
            steps {
                // Etape 1 : On clone le code github
                checkout scm
            }
        }

        stage('Build Movie Service') {
            steps {
                script {
                    echo 'Construction de l image Movie Service...'
                    // On va dans le dossier et on construit
                    dir('movie-service') {
                        sh "docker build -t ${registry}/movie-service:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Build Cast Service') {
            steps {
                script {
                    echo 'Construction de l image Cast Service...'
                    dir('cast-service') {
                        sh "docker build -t ${registry}/cast-service:${IMAGE_TAG} ."
                    }
                }
            }
        }

        stage('Push sur DockerHub') {
            steps {
                script {
                    // On se connecte à DockerHub avec les identifiants Jenkins
                    docker.withRegistry('', registryCredential) {
                        echo 'Envoi des images sur DockerHub...'
                        sh "docker push ${registry}/movie-service:${IMAGE_TAG}"
                        sh "docker push ${registry}/cast-service:${IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Deploiement Base de Donnees') {
            steps {
                script {
                    echo 'Deploiement des bases de donnees...'
                    sh 'kubectl apply -f k8s/databases.yaml'
                }
            }
        }

        stage('Deploiement Applications') {
            steps {
                script {
                    echo 'Deploiement via Helm...'
                    
                    // Déploiement Cast Service
                    sh """
                    helm upgrade --install cast-service ./charts \
                        --namespace dev \
                        --set image.repository=${registry}/cast-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.port=8001 \
                        --set service.targetPort=8001 \
                        --set service.nodePort=30007 \
                        --set env[0].name=DATABASE_URI \
                        --set env[0].value='postgresql://cast_user:cast_password@cast-db:5432/cast_db' \
                        --set livenessProbe.httpGet.path='/api/v1/casts/docs' \
                        --set readinessProbe.httpGet.path='/api/v1/casts/docs'
                    """

                    // Déploiement Movie Service
                    sh """
                    helm upgrade --install movie-service ./charts \
                        --namespace dev \
                        --set image.repository=${registry}/movie-service \
                        --set image.tag=${IMAGE_TAG} \
                        --set service.port=8000 \
                        --set service.nodePort=30008 \
                        --set env[0].name=DATABASE_URI \
                        --set env[0].value='postgresql://movie_user:movie_password@movie-db:5432/movie_db' \
                        --set livenessProbe.httpGet.path='/api/v1/movies/docs' \
                        --set readinessProbe.httpGet.path='/api/v1/movies/docs'
                    """
                }
            }
        }
    }
}
