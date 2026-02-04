pipeline {
  agent any

  environment {
    AWS_REGION  = "ap-south-1"
    ECR_REPO    = "087471322416.dkr.ecr.ap-south-1.amazonaws.com/netflix-clone"
    CLUSTER_NAME = "netflix-cluster"
    NAMESPACE   = "netflix"

    AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
  }

  stages {

    // stage('Checkout') {
    //   steps {
    //     checkout scm
    //   }
    // }

    stage('Build App') {
      steps {
        sh '''
          npm install
          npm run build
        '''
      }
    }

    stage('Build Docker Image') {
      steps {
        sh '''
          IMAGE_TAG=$(git rev-parse --short HEAD)
          echo "IMAGE_TAG=$IMAGE_TAG" > image.env

          docker build -t netflix-clone:$IMAGE_TAG .
          docker tag netflix-clone:$IMAGE_TAG $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Push Image to ECR') {
      steps {
        sh '''
          source image.env

          aws ecr get-login-password --region $AWS_REGION \
          | docker login --username AWS --password-stdin $ECR_REPO

          docker push $ECR_REPO:$IMAGE_TAG
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          source image.env

          aws eks update-kubeconfig \
            --region $AWS_REGION \
            --name $CLUSTER_NAME

          kubectl set image deployment/netflix-clone \
            netflix-clone=$ECR_REPO:$IMAGE_TAG \
            -n $NAMESPACE

          kubectl rollout status deployment/netflix-clone -n $NAMESPACE
        '''
      }
    }
  }

  post {
    always {
      emailext(
        subject: "Jenkins Build: ${env.JOB_NAME} #${env.BUILD_NUMBER} - ${currentBuild.currentResult}",
        body: """
          <h2>Jenkins Build Notification</h2>

          <p><b>Job:</b> ${env.JOB_NAME}</p>
          <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
          <p><b>Status:</b> ${currentBuild.currentResult}</p>

          <p><b>Build URL:</b>
          <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>

          <p><b>Console Output:</b>
          <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a></p>
        """,
        to: "siddharsmit@gmail.com"
      )
    }
  }
}
