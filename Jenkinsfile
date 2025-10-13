pipeline {
  agent none
  
  environment {
    IMAGE_NAME = "alpinehelloworld"
    IMAGE_TAG = "latest"
    STAGING = "eazytraining-staging"
    PRODUCTION = "eazytraining-production"
    DOCKER_IMAGE = "systoker/${IMAGE_NAME}:${IMAGE_TAG}"
  }
  
  stages {
    stage('Build image') {
      agent any
      steps {
        script {
          sh 'docker build -t ${DOCKER_IMAGE} .'
        }
      }
    }
    
    stage('Run container based on builded image') {
      agent any
      steps {
        script {
          sh '''
            # Nettoyer le conteneur s'il existe déjà
            docker stop ${IMAGE_NAME} 2>/dev/null || true
            docker rm ${IMAGE_NAME} 2>/dev/null || true
            
            # Lancer le nouveau conteneur
            docker run --name ${IMAGE_NAME} -d -p 8081:5000 -e PORT=5000 ${DOCKER_IMAGE}
            sleep 5
          ''' 
        }
      }
    }
    
    stage('Test image') {
      agent any
      steps {
        script {
          sh '''
            # Récupérer l'IP du conteneur Docker
            CONTAINER_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${IMAGE_NAME})
            echo "Testing container at IP: $CONTAINER_IP"
            
            # Tester l'application
            curl -f http://$CONTAINER_IP:5000 | grep -q "Hello world!"
          ''' 
        }
      }
    }
    
    stage('Clean container') {
      agent any
      steps {
        script {
          sh '''
            docker stop ${IMAGE_NAME} || true
            docker rm ${IMAGE_NAME} || true
          ''' 
        }
      }
    }
    
    stage('Push image to registry') {
      when {
        expression { GIT_BRANCH == 'origin/master' }
      }
      agent any
      environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub_credentials')
      }
      steps {
        script {
          sh '''
            echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
            docker push ${DOCKER_IMAGE}
            docker logout
          '''
        }
      }
    }
    
    stage('Deploy to staging') {
      when {
        expression { GIT_BRANCH == 'origin/master' }
      }
      agent any 
      environment {
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps {
        script {
          sh '''
            heroku container:login
            heroku create $STAGING || echo "Project already exists"
            heroku container:push -a $STAGING web
            heroku container:release -a $STAGING web
          ''' 
        }
      }
    }
    
    stage('Deploy to production') {
      when {
        expression { GIT_BRANCH == 'origin/master' }
      }
      agent any 
      environment {
        HEROKU_API_KEY = credentials('heroku_api_key')
      }
      steps {
        script {
          sh '''
            heroku container:login
            heroku create $PRODUCTION || echo "Project already exists"
            heroku container:push -a $PRODUCTION web
            heroku container:release -a $PRODUCTION web
          ''' 
        }
      }
    }
  }
  
  post {
    always {
      script {
        // Nettoyage effectué dans le stage dédié
        echo 'Pipeline terminé'
      }
    }
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed. Please check the logs.'
    }
  }
}
