pipeline {
    agent any

    agent any
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

    stages {
    		stage('Build'){
    		
	    			steps {
	    			
	    			sh '''	
					docker build -t $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG movie-service/
					docker build -t $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG cast-service/					
              		sleep 6

                    docker tag $DOCKER_IMAGE_DB:DOCKER_TAG_DB $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: movie-db-$DOCKER_TAG
                    docker tag $DOCKER_IMAGE_DB:DOCKER_TAG_DB $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: cast-db-$DOCKER_TAG
                    docker tag $DOCKER_IMAGE_WEB:DOCKER_TAG_WEB $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: web-db-$DOCKER_TAG
	    			'''
	    				
	    			}

    		}

        stage('Run'){ 
                steps {
                    script {
                    sh '''
                    docker run -d --name movie-service $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE:movie-$DOCKER_TAG
                    docker run -d --name cast-service $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE:cast-$DOCKER_TAG
                    docker run -d --name movie-db -p 5432:5432 $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: movie-db-$DOCKER_TAG
                    docker run -d --name movie-db -p 5432:5432 $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: cast-db-$DOCKER_TAG
                    docker run -d --name web -p 8080:80 $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: web-db$DOCKER_TAG                    
                    sleep 10
                    '''
                    }
                }
            }

        stage('Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_LOGIN_USR -p $DOCKER_LOGIN_PSW
                docker push $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE:$movie-$DOCKER_TAG
                docker push $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE:$cast-$DOCKER_TAG
       			docker push $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: movie-db-$DOCKER_TAG
                docker push $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: cast-db-$DOCKER_TAG
                docker push $DOCKER_LOGIN_USR/$DOCKER_HUB_REPOSITORY_IMAGE: web-db$DOCKER_TAG                    
                
                '''
                }
            }

        }
    }
}
