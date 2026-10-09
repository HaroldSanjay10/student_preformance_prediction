pipeline {
  agent any

  environment {
    REGISTRY        = credentials('registry-namespace')   // Secret text, e.g. docker.io/youruser
    EC2_HOST        = credentials('ec2-host')             // Secret text, e.g. ubuntu@1.2.3.4
    IMAGE_TAG       = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'local'}"
    BACKEND_IMAGE   = "${REGISTRY}/spps-backend"
    FRONTEND_IMAGE  = "${REGISTRY}/spps-frontend"
    REMOTE_DIR      = "/opt/spps"
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 45, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Test') {
      agent {
        docker { image 'python:3.11-slim'; reuseNode true }
      }
      steps {
        dir('backend') {
          sh '''
            python -m venv /tmp/venv
            . /tmp/venv/bin/activate
            pip install -q -r requirements.txt
            python -m pytest --junitxml=test-results.xml
          '''
        }
      }
      post { always { junit 'backend/test-results.xml' } }
    }

    stage('Build images') {
      steps {
        sh '''
          docker build -f backend/Dockerfile -t ${BACKEND_IMAGE}:${IMAGE_TAG} .
          docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} frontend
        '''
      }
    }

    stage('Trivy scan') {
      steps {
        sh '''
          for img in ${BACKEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:${IMAGE_TAG}; do
            docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
              -v trivy-cache:/root/.cache aquasec/trivy:0.54.1 image \
              --exit-code 1 --severity HIGH,CRITICAL --ignore-unfixed --no-progress "$img"
          done
        '''
      }
    }

    stage('Push to registry') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'registry-creds', usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
          sh '''
            REG_HOST=$(echo "${REGISTRY}" | cut -d/ -f1)
            echo "$REG_PASS" | docker login "$REG_HOST" -u "$REG_USER" --password-stdin
            docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
            docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
            docker logout "$REG_HOST"
          '''
        }
      }
    }

    stage('Deploy to EC2') {
      when { branch 'main' }
      steps {
        sshagent(credentials: ['ec2-ssh-key']) {
          sh '''
            ssh -o StrictHostKeyChecking=accept-new ${EC2_HOST} "mkdir -p ${REMOTE_DIR}/scripts"
            scp -o StrictHostKeyChecking=accept-new docker-compose.yml ${EC2_HOST}:${REMOTE_DIR}/docker-compose.yml
            scp -o StrictHostKeyChecking=accept-new scripts/deploy.sh ${EC2_HOST}:${REMOTE_DIR}/scripts/deploy.sh
            ssh ${EC2_HOST} "chmod +x ${REMOTE_DIR}/scripts/deploy.sh && ${REMOTE_DIR}/scripts/deploy.sh ${IMAGE_TAG}"
          '''
        }
      }
    }
  }

  post {
    always {
      sh 'docker image rm ${BACKEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:${IMAGE_TAG} || true'
    }
  }
}
