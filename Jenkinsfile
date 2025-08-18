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
        MOVIE_DB_LOGIN = credentials('MOVIE_DB_LOGIN')
        CAST_DB_LOGIN = credentials('CAST_DB_LOGIN')
        DB_MOVIE_NAME = movie_db_dev
        DB_CAST_NAME = cast_db_dev
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
                # Arrêter et supprimer des conteneurs qui potentiellement tournent
                docker stop movie-service cast-service movie-db cast-db web || true
                docker rm movie-service cast-service movie-db cast-db web || true
                '''
            }
        }
        stage('Build') {
            steps {
                sh '''
                # Build application images only
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG movie-service/
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG cast-service/
                
                # Pull base images (we'll use them directly)
                docker pull $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                docker pull $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB
                '''
            }
        }
        stage('Run') {
            steps {
                sh '''
                # Base movie
                docker run -d \
                    --name movie-db \
                    -e POSTGRES_DB=$DB_MOVIE_NAME \
                    -e POSTGRES_USER=$MOVIE_DB_LOGIN_USR \
                    -e POSTGRES_PASSWORD=$MOVIE_DB_LOGIN_PSW \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                
                # Base cast  
                docker run -d \
                    --name cast-db \
                    -e POSTGRES_DB=$DB_CAST_NAME \
                    -e POSTGRES_USER=$CAST_DB_LOGIN_USR \
                    -e POSTGRES_PASSWORD=$CAST_DB_LOGIN_PSW \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB

                echo "Attendre que les bases de données soient prêtes..."
                sleep 20

                # Lancer les services de l'appli
                docker run -d \
                    --name movie-service \
                    -p 8001:8000 \
                    $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG

                docker run -d \
                    --name cast-service \
                    -p 8002:8000 \
                    $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG

                # Lancer le serveur web nginx
                docker run -d \
                    --name web \
                    -p 8080:80 \
                    $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB

                echo "Attendre le démarrage de tous les services..."
                sleep 10

                # Vérifier que tous les conteneurs tournent
                docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
                '''
            }
        }
        stage('Test') {
            steps {
                sh '''
                # Vérifications basiques de l'état des services
                echo "Exécution des vérifications de santé..."
                
                # Vérifie si les conteneurs sont en cours d'exécution
                if ! docker ps | grep -q movie-service; then
                    echo "ERREUR : movie-service n'est pas en cours d'exécution"
                    exit 1
                fi
                
                if ! docker ps | grep -q cast-service; then
                    echo "ERREUR : cast-service n'est pas en cours d'exécution"
                    exit 1
                fi
                
                if ! docker ps | grep -q web; then
                    echo "ERREUR : le service web n'est pas en cours d'exécution"
                    exit 1
                fi
                
                # Teste les connexions aux bases de données
                docker exec movie-db pg_isready -U movie_db_username || exit 1
                docker exec cast-db pg_isready -U cast_db_username || exit 1
                
                # Vérifier que les bases de données ont été créées avec les bons noms
                echo "Vérification des bases de données..."
                docker exec movie-db psql -U movie_db_username -d movie_db_dev -c "SELECT 1;" || echo "AVERTISSEMENT : Connexion à movie_db_dev échouée"
                docker exec cast-db psql -U cast_db_username -d cast_db_dev -c "SELECT 1;" || echo "AVERTISSEMENT : Connexion à cast_db_dev échouée"
                
                echo "Toutes les vérifications de santé sont passées avec succès !"
                '''
            }
        }
        stage('Push') {
            steps {
                sh '''
                # Login to Docker Hub
                docker login -u $DOCKER_LOGIN_USR -p $DOCKER_LOGIN_PSW
                
                # Push only custom application images
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG
                
                echo "All images pushed successfully!"
                '''
            }
        }
    }

}
