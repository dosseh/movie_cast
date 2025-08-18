pipeline {
    environment {
        KUBE_DOMAIN = "http://movie-cast.ip-ddns.com"
        KUBE_NAMESPACE_DEV = "dev"
        KUBE_NAMESPACE_QA = "qa"
        KUBE_NAMESPACE_STAGING = "staging"
        KUBE_NAMESPACE_PROD = "prod"
        DOCKER_HUB_REPOSITORY_IMAGE = "dos7/movie-cast"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        DOCKER_LOGIN = credentials('DOCKER_LOGIN')
        DOCKER_IMAGE_DB = "postgres"
        DOCKER_TAG_DB = "12.1-alpine"
        DOCKER_IMAGE_WEB = "nginx"
        DOCKER_TAG_WEB = "latest"
    }
    agent any
    stages {
        stage('Cleanup') {
            steps {
                sh '''
                # Arreter et supprimer des conteneurs qui potentielement tournent
                docker stop movie-service cast-service movie-db cast-db web || true
                docker rm movie-service cast-service movie-db cast-db web || true
                '''
            }
        }
        stage('Build') {
            steps {
                sh '''
                # Build les images de l appli uniquement
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG movie-service/
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG cast-service/
                
                # Pull des images de bases qui sont officielles
                docker pull $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                docker pull $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB
                '''
            }
        }
        stage('Run') {
            steps {
                sh '''
                docker run -d \
                    --name movie-db \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                
                docker run -d \
                    --name cast-db \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                
                echo "Attendre que la base de données soit prete..."
                sleep 15
                
                # Lancer les services de l appli
                docker run -d \
                    --name movie-service \
                    $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG
                
                docker run -d \
                    --name cast-service \
                    $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG
                
                # Lancer le serveur web nginx
                docker run -d \
                    --name web \
                    $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB
                
                echo "Attendre le demarage de toute les services..."
                sleep 10
                
                # Verifier que tout les conteneurs tournent
                docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
                '''
            }
        }
        stage('Test') {
			steps {
			    sh '''
			    # Vérifications basiques de l'état des services
			    echo "Exécution des vérifications de santé..."

			    # Vérifie si les conteneurs sont en cours d’exécution
			    if ! docker ps | grep -q movie-service; then
			        echo "ERREUR : movie-service n’est pas en cours d’exécution"
			        exit 1
			    fi

			    if ! docker ps | grep -q cast-service; then
			        echo "ERREUR : cast-service n’est pas en cours d’exécution"
			        exit 1
			    fi

			    if ! docker ps | grep -q web; then
			        echo "ERREUR : le service web n’est pas en cours d’exécution"
			        exit 1
			    fi

			    # Teste les connexions aux bases de données (utilisateur postgres par défaut)
			    docker exec movie-db pg_isready -U postgres || exit 1
			    docker exec cast-db pg_isready -U postgres || exit 1

			    echo "Toutes les vérifications de santé sont passées avec succès !"
			    '''
			}
        }
        stage('Push') {
            steps {
                sh '''
                # Login sur Docker Hub
                docker login -u $DOCKER_LOGIN_USR -p $DOCKER_LOGIN_PSW
                
                # Push des immages de l application 
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG
                
                echo "Tout les images sont poussé avec succes!"
                '''
            }
        }
    }
}
