pipeline {
    environment {
        KUBE_DOMAIN = "http://movie-cast.ip-ddns.com"
        KUBE_NAMESPACE_DEV = "dev"
        KUBE_NAMESPACE_QA = "qa"
        KUBE_NAMESPACE_STAGING = "staging"
        KUBE_NAMESPACE_PROD = "prod"
        DOCKER_HUB_REPOSITORY_IMAGE = "dos7/movie-cast"
        DOCKER_LOGIN = credentials('DOCKER_LOGIN')
        MOVIE_DB_LOGIN = credentials('MOVIE_DB_LOGIN')
        CAST_DB_LOGIN = credentials('CAST_DB_LOGIN')
        DB_MOVIE_NAME = "movie_db_dev"
        DB_CAST_NAME = "cast_db_dev"
        DOCKER_IMAGE_DB = "postgres"
        DOCKER_TAG_DB = "12.1-alpine"
        DOCKER_IMAGE_WEB = "nginx"
        DOCKER_TAG_WEB = "latest"

    } 
    agent any
    stages {
	    stage('Init') {
	       
	        steps {
	            script {
	                // ------------------------------
	                // Variables globales pour le pipeline
	                // ------------------------------
	                env.COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
	                env.DOCKER_TAG = "v.${env.BUILD_ID}.${env.COMMIT_SHORT}"
	            }

			        // ------------------------------
			        // Cleanup des conteneurs existants
			        // ------------------------------
			        sh '''
			        echo " Nettoyage des conteneurs Docker potentiellement existants..."
			        docker stop movie-service cast-service movie-db cast-db web || true
			        docker rm movie-service cast-service movie-db cast-db web || true
			        '''
			    }
			}
        // ----------------
        // Stage Build
        // ----------------
        stage('Build') {
            steps {
                sh '''
                #Construire les images de l appli
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG movie-service/
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG cast-service/
                
                # Pull les images de bases
                docker pull $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                docker pull $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB

                docker tag "$DOCKER_IMAGE_DB:$DOCKER_TAG_DB" "$DOCKER_HUB_REPOSITORY_IMAGE:movie-db-$DOCKER_TAG"
                docker tag "$DOCKER_IMAGE_DB:$DOCKER_TAG_DB" "$DOCKER_HUB_REPOSITORY_IMAGE:cast-db-$DOCKER_TAG"
                docker tag "$DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB" "$DOCKER_HUB_REPOSITORY_IMAGE:web-$DOCKER_TAG"


                '''
            }
        }
        
        // ----------------
        // Stage Run
        // ----------------
        stage('Run') {
            steps {
                sh '''
                # Créer un réseau Docker dédié
                docker network create movie-cast-net || true
        
                # Lancer la base de données movie
                docker run -d --name movie-db --network movie-cast-net \
                    -e POSTGRES_DB=$DB_MOVIE_NAME \
                    -e POSTGRES_USER=$MOVIE_DB_LOGIN_USR \
                    -e POSTGRES_PASSWORD=$MOVIE_DB_LOGIN_PSW \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
        
                # Lancer la base de données cast
                docker run -d --name cast-db --network movie-cast-net \
                    -e POSTGRES_DB=$DB_CAST_NAME \
                    -e POSTGRES_USER=$CAST_DB_LOGIN_USR \
                    -e POSTGRES_PASSWORD=$CAST_DB_LOGIN_PSW \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
        
                # Attendre que les bases de données soient prêtes
                echo "Attente des bases de données..."
                until docker exec movie-db pg_isready -U $MOVIE_DB_LOGIN_USR; do
                    echo "Waiting for movie-db..."
                    sleep 2
                done
        
                until docker exec cast-db pg_isready -U $CAST_DB_LOGIN_USR; do
                    echo "Waiting for cast-db..."
                    sleep 2
                done
        
                # Lancer le service movie
                docker run -d --name movie-service --network movie-cast-net \
                    -p 8001:8000 \
                    -e DATABASE_URI="postgresql://$MOVIE_DB_LOGIN_USR:$MOVIE_DB_LOGIN_PSW@movie-db:5432/$DB_MOVIE_NAME" \
                    $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG \
                    uvicorn app.main:app --host 0.0.0.0 --port 8000
        
                # Lancer le service cast
                docker run -d --name cast-service --network movie-cast-net \
                    -p 8002:8000 \
                    -e DATABASE_URI="postgresql://$CAST_DB_LOGIN_USR:$CAST_DB_LOGIN_PSW@cast-db:5432/$DB_CAST_NAME" \
                    $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG \
                    uvicorn app.main:app --host 0.0.0.0 --port 8000
        
                # Lancer le serveur web nginx
                docker run -d --name web --network movie-cast-net \
                    -p 80:80 \
                    $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB
        
                echo "Vérification que tous les conteneurs tournent..."
                docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
                '''
            }
        }
        // ----------------
        // Stage Test
        // ----------------
        stage('Test') {
                    steps {
                        sh '''
                        echo "🔍 Test des bases de données..."
                
                        # Vérifier que les bases sont prêtes
                        until docker exec movie-db pg_isready -U movie_db_username -d movie_db_dev; do
                            echo "Attente Movie DB..."
                            sleep 2
                        done
                        echo " Movie DB disponible"
                
                        until docker exec cast-db pg_isready -U cast_db_username -d cast_db_dev; do
                            echo "Attente Cast DB..."
                            sleep 2
                        done
                        echo " Cast DB disponible"
                
                        echo " Test des services depuis le conteneur web (Nginx)..."
                
                        # Vérifier que movie-service est prêt
                        until docker exec web curl -sf http://movie-service:8000/api/v1/movies/docs; do
                            echo "Attente movie-service..."
                            sleep 2
                        done
                        echo " Movie service OK"
                
                        # Vérifier que cast-service est prêt
                        until docker exec web curl -sf http://cast-service:8000/api/v1/casts/docs; do
                            echo "Attente cast-service..."
                            sleep 2
                        done
                        echo " Cast service OK"
                
                        echo " Tous les tests de santé sont passés avec succès !"
                        '''
                    }
                }
        // ----------------
        // Stage Push
        // ----------------
        stage('Push') {
            steps {
                sh '''
                # Login a Docker Hub
                docker login -u $DOCKER_LOGIN_USR -p $DOCKER_LOGIN_PSW
                
                # Push seulement les images des appli customisées
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:movie-db-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:cast-db-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:web-$DOCKER_TAG

                echo "Toute les images sont poussées avec succès !"
                '''
            }
        }
        
		stage('Dev') {
            environment {
                KUBECONFIG = credentials("KUBE_CONFIG")
            }
            steps {
                script {
                    deployToHelm($KUBE_NAMESPACE_DEV)
                }
            }
        }
    }
}

// -------------------
// Fonction réutilisable pour le deploiement dans helm
// -------------------
def deployToHelm(namespace) {
    sh """
    rm -Rf .kube
    mkdir .kube
    cat $KUBECONFIG > .kube/config
    cp charts/values.yaml values.yml

    helm upgrade --install app charts/ \
      --values=values.yml \
      --namespace ${namespace} \
      --set movieService.db.image=${DOCKER_HUB_REPOSITORY_IMAGE} \
      --set movieService.db.tag="movie-db-\$DOCKER_TAG" \
      --set castService.db.image=${DOCKER_HUB_REPOSITORY_IMAGE} \
      --set castService.db.tag="cast-db-\$DOCKER_TAG" \
      --set movieService.image=${DOCKER_HUB_REPOSITORY_IMAGE} \
      --set movieService.tag="movie-\$DOCKER_TAG" \
      --set castService.image=${DOCKER_HUB_REPOSITORY_IMAGE} \
      --set castService.tag="cast-\$DOCKER_TAG" \
      --set nginx.image=${DOCKER_HUB_REPOSITORY_IMAGE} \
      --set nginx.tag="web-\$DOCKER_TAG" \
      --set environment=${namespace}

    sleep 10
    kubectl get pods -n ${namespace}
    rm -f values.yml
    """
}
