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
        KUBECONFIG = credentials("KUBE_CONFIG")
        GITHUB_TOKEN = credentials("GITHUB_TOKEN")
        GITHUB_REPO = "github.com/dosseh/movie_cast.git"
    }

    agent any

    stages {
        stage('Init') {
            steps {
                script {
                    env.COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.DOCKER_TAG = "v.${env.BUILD_ID}.${env.COMMIT_SHORT}"
                    env.CURRENT_BRANCH = sh(script: 'git branch --show-current', returnStdout: true).trim()
                }

                sh '''
                echo "Nettoyage des conteneurs Docker existants..."
                docker stop movie-service cast-service movie-db cast-db web || true
                docker rm movie-service cast-service movie-db cast-db web || true
                '''
            }
        }

        stage('Build') {
            steps {
                sh '''
                echo "Construction des images Docker..."
                
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG movie-service/
                docker build -t $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG cast-service/
                docker pull $DOCKER_IMAGE_DB:$DOCKER_TAG_DB
                docker pull $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB
                
                docker tag "$DOCKER_IMAGE_DB:$DOCKER_TAG_DB" "$DOCKER_HUB_REPOSITORY_IMAGE:movie-db-$DOCKER_TAG"
                docker tag "$DOCKER_IMAGE_DB:$DOCKER_TAG_DB" "$DOCKER_HUB_REPOSITORY_IMAGE:cast-db-$DOCKER_TAG"
                docker tag "$DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB" "$DOCKER_HUB_REPOSITORY_IMAGE:web-$DOCKER_TAG"
                '''
            }
        }

        stage('Run') {
            steps {
                sh '''
                echo "Démarrage des conteneurs..."
                docker network create movie-cast-net || true

                docker run -d --name movie-db --network movie-cast-net \
                    -e POSTGRES_DB=$DB_MOVIE_NAME \
                    -e POSTGRES_USER=$MOVIE_DB_LOGIN_USR \
                    -e POSTGRES_PASSWORD=$MOVIE_DB_LOGIN_PSW \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB

                docker run -d --name cast-db --network movie-cast-net \
                    -e POSTGRES_DB=$DB_CAST_NAME \
                    -e POSTGRES_USER=$CAST_DB_LOGIN_USR \
                    -e POSTGRES_PASSWORD=$CAST_DB_LOGIN_PSW \
                    $DOCKER_IMAGE_DB:$DOCKER_TAG_DB

                echo "Attente des bases de données..."
                until docker exec movie-db pg_isready -U $MOVIE_DB_LOGIN_USR; do sleep 2; done
                until docker exec cast-db pg_isready -U $CAST_DB_LOGIN_USR; do sleep 2; done

                docker run -d --name movie-service --network movie-cast-net -p 8001:8000 \
                    -e DATABASE_URI="postgresql://$MOVIE_DB_LOGIN_USR:$MOVIE_DB_LOGIN_PSW@movie-db:5432/$DB_MOVIE_NAME" \
                    $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG uvicorn app.main:app --host 0.0.0.0 --port 8000

                docker run -d --name cast-service --network movie-cast-net -p 8002:8000 \
                    -e DATABASE_URI="postgresql://$CAST_DB_LOGIN_USR:$CAST_DB_LOGIN_PSW@cast-db:5432/$DB_CAST_NAME" \
                    $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG uvicorn app.main:app --host 0.0.0.0 --port 8000

                docker run -d --name web --network movie-cast-net -p 80:80 $DOCKER_IMAGE_WEB:$DOCKER_TAG_WEB
                docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                echo "Tests des services..."
                until docker exec movie-db pg_isready -U $MOVIE_DB_LOGIN_USR -d $DB_MOVIE_NAME; do sleep 2; done
                until docker exec cast-db pg_isready -U $CAST_DB_LOGIN_USR -d $DB_CAST_NAME; do sleep 2; done
                until docker exec web curl -sf http://movie-service:8000/api/v1/movies/docs; do sleep 2; done
                until docker exec web curl -sf http://cast-service:8000/api/v1/casts/docs; do sleep 2; done
                echo "Tous les tests de santé sont passés !"
                '''
            }
        }

        stage('Push') {
            steps {
                sh '''
                echo "Push vers Docker Hub..."
                docker login -u $DOCKER_LOGIN_USR -p $DOCKER_LOGIN_PSW
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:movie-db-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:cast-db-$DOCKER_TAG
                docker push $DOCKER_HUB_REPOSITORY_IMAGE:web-$DOCKER_TAG
                echo "Toutes les images sont poussées !"
                '''
            }
        }

        // Déploiements simplifiés avec fonction Helm
        stage('Deploy Dev') {
            when { branch 'dev' }
            steps {
                script {
                    echo "Déploiement vers DEV"
                    deployToHelm(env.KUBE_NAMESPACE_DEV)
                }
            }
            post {
                success {
                    echo "Déploiement DEV réussi - Merge vers QA"
                    autoMergeToNextEnvironment('dev','qa')
                }
            }
        }
        
        stage('Deploy QA') {
            when { branch 'qa' }
            steps {
                script {
                    echo "Déploiement vers QA"
                    deployToHelm(env.KUBE_NAMESPACE_QA)
                }
            }
            post {
                success {
                    echo "Déploiement QA réussi - Merge vers STAGING"
                    autoMergeToNextEnvironment('qa','staging')
                }
            }
        }
        
        stage('Deploy Staging') {
            when { branch 'staging' }
            steps {
                script {
                    echo "Déploiement vers STAGING"
                    deployToHelm(env.KUBE_NAMESPACE_STAGING)
                }
            }
            post {
                success {
                    echo "Déploiement STAGING réussi - Merge vers MASTER"
                    autoMergeToNextEnvironment('staging','master')
                }
            }
        }
        
        stage('Deploy Prod') {
            when { branch 'master' }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Déployer en production ?', ok: 'OUI'
                }
                script {
                    deployToHelm(env.KUBE_NAMESPACE_PROD)
                }
            }
            post {
                success {
                    echo "Déploiement PRODUCTION réussi !"
                }
            }
        }

        
    }

    post {
        always {
            sh '''
            echo "Nettoyage final..."
            docker stop movie-service cast-service movie-db cast-db web || true
            docker rm movie-service cast-service movie-db cast-db web || true
            docker network rm movie-cast-net || true
            '''
        }
    }
}

// -------------------
// Fonctions réutilisables
// -------------------
def deployToHelm(namespace) {
    withCredentials([file(credentialsId: 'KUBE_CONFIG', variable: 'KUBECONFIG_FILE')]) {
        sh """
        mkdir -p .kube
        cp \$KUBECONFIG_FILE .kube/config
        cp charts/values.yaml values.yml
        helm upgrade --install app charts/ --values=values.yml --namespace ${namespace} --create-namespace \
          --set movieService.db.image=\${DOCKER_HUB_REPOSITORY_IMAGE} --set movieService.db.tag="movie-db-\$DOCKER_TAG" \
          --set castService.db.image=\${DOCKER_HUB_REPOSITORY_IMAGE} --set castService.db.tag="cast-db-\$DOCKER_TAG" \
          --set movieService.image=\${DOCKER_HUB_REPOSITORY_IMAGE} --set movieService.tag="movie-\$DOCKER_TAG" \
          --set castService.image=\${DOCKER_HUB_REPOSITORY_IMAGE} --set castService.tag="cast-\$DOCKER_TAG" \
          --set nginx.image=\${DOCKER_HUB_REPOSITORY_IMAGE} --set nginx.tag="web-\$DOCKER_TAG" \
          --set environment=${namespace}
        sleep 10
        kubectl get pods -n ${namespace}
        rm -f values.yml
        """
    }
}

def autoMergeToNextEnvironment(sourceBranch, targetBranch) {
    sh """
    echo " Merge automatique de ${sourceBranch} vers ${targetBranch}"

    # Config Git
    git config user.name "CI Bot"
    git config user.email "ci-bot@movie-cast.com"

    echo " Fetch de toutes les branches"
    git fetch --all --prune

    # Vérifier si la branche cible existe sur le remote
    if git show-ref --verify --quiet refs/remotes/origin/${targetBranch}; then
        echo " La branche distante ${targetBranch} existe"
        git checkout -B ${targetBranch} origin/${targetBranch}
    else
        echo "La branche distante ${targetBranch} n'existe pas encore"
        echo " Création de ${targetBranch} à partir de ${sourceBranch}"
        git checkout -B ${targetBranch} origin/${sourceBranch}
    fi

    echo " Merge de origin/${sourceBranch} vers ${targetBranch}"
    git merge --no-ff --no-edit origin/${sourceBranch} -m "Auto-merge ${sourceBranch} → ${targetBranch} - Build #\${BUILD_NUMBER}" || {
        echo " Conflits détectés entre ${sourceBranch} et ${targetBranch}"
        git diff --name-only --diff-filter=U
        exit 1
    }

    echo " Push vers GitHub"
    git push https://${GITHUB_TOKEN}@${GITHUB_REPO} ${targetBranch}

    echo " Merge réussi: ${sourceBranch} → ${targetBranch}"
    """
}
