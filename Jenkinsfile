cat > Jenkinsfile <<'EOF'
pipeline {
  agent any
  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
    REPO_NAME = 'hello-devops'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Unit tests') {
      steps {
        sh '''
          docker run --rm \
            -v "$WORKSPACE":/app -w /app \
            python:3.12 bash -lc "
              python -m pip install --upgrade pip &&
              pip install -r requirements.txt &&
              pytest -q
            "
        '''
      }
    }

    stage('Login to ECR') {
      steps {
        withAWS(credentials: 'aws-creds', region: "${AWS_DEFAULT_REGION}") {
          sh '''
            ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} \
              | docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
            echo $ACCOUNT_ID > account_id.txt
          '''
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        sh '''
          ACCOUNT_ID=$(cat account_id.txt)
          REPO=${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}
          GIT_SHA=$(git rev-parse --short=12 HEAD)

          docker build -t ${REPO}:${GIT_SHA} -t ${REPO}:latest .
          docker push ${REPO}:${GIT_SHA}
          docker push ${REPO}:latest
        '''
      }
    }

    stage('Deploy on EC2') {
      steps {
        sh '''
          ACCOUNT_ID=$(cat account_id.txt)
          REPO=${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${REPO_NAME}

          docker pull ${REPO}:latest
          docker rm -f app || true
          docker run -d --name app --restart unless-stopped -p 80:5000 ${REPO}:latest
        '''
      }
    }
  }
}
EOF
