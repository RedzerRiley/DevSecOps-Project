pipeline {
  agent any
  environment {
    DOCKER_IMAGE = "netflix-demo:${env.BUILD_ID}"
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }
    stage('Build Docker Image') {
      steps {
        script {
          sh "docker build -t ${DOCKER_IMAGE} ."
        }
      }
    }
    stage('Trivy Image Scan') {
      steps {
        script {
          // ensure a local cache dir exists for trivy
          sh 'mkdir -p $HOME/.cache/trivy'
          // generate JSON report of image scan
          sh '''
            docker run --rm \
              -v $HOME/.cache/trivy:/root/.cache/trivy \
              -v "$WORKSPACE":/reports \
              aquasec/trivy:latest image --format json -o /reports/trivy-image-report.json ${DOCKER_IMAGE} || true
          '''
        }
        archiveArtifacts artifacts: 'trivy-image-report.json', fingerprint: true
      }
    }
    stage('Optional: Push to DockerHub') {
      when { expression { return false } } // set to true to enable
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          sh "echo $DOCKER_PASS | docker login --username $DOCKER_USER --password-stdin"
          sh "docker tag ${DOCKER_IMAGE} ${DOCKER_USER}/${DOCKER_IMAGE}"
          sh "docker push ${DOCKER_USER}/${DOCKER_IMAGE}"
        }
      }
    }
    stage('Run Container (demo)') {
      steps {
        script {
          sh "docker rm -f demo-app || true"
          sh "docker run -d --name demo-app -p 8081:80 ${DOCKER_IMAGE}"
        }
      }
    }
  }
  post {
    always {
      echo "Build ${currentBuild.fullDisplayName} finished"
    }
  }
}
