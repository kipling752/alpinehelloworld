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
            # Vérifier que le conteneur tourne
            echo "=== Container Status ==="
            docker ps | grep ${IMAGE_NAME}
            
            # Vérifier les logs du conteneur
            echo "=== Container Logs ==="
            docker logs ${IMAGE_NAME}
            
            # Tester via l'IP de la gateway Docker (accessible depuis Jenkins)
            GATEWAY_IP=$(docker network inspect bridge --format='{{range .IPAM.Config}}{{.Gateway}}{{end}}')
            echo "=== Testing via Gateway IP: $GATEWAY_IP:8081 ==="
            
            # Attendre que l'application soit prête
            sleep 3
            
            # Tester l'application
            curl -f --max-time 10 http://$GATEWAY_IP:8081 | grep -q "Hello world!"
            
            echo "=== Test passed successfully! ==="
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
