pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-kaniko
spec:
  serviceAccountName: jenkins
  containers:
    - name: kaniko
      image: gcr.io/kaniko-project/executor:v1.16.0-debug
      imagePullPolicy: Always
      command:
        - sleep
      args:
        - 99d
      tty: true
      volumeMounts:
        - name: docker-config
          mountPath: /kaniko/.docker/
    - name: git
      image: alpine/git:2.45.2
      command:
        - sleep
      args:
        - 99d
      tty: true
  volumes:
    - name: docker-config
      emptyDir: {}
'''
    }
  }

  environment {
    AWS_REGION     = 'eu-central-1'
    ECR_REPOSITORY = '165690630824.dkr.ecr.eu-central-1.amazonaws.com/lesson-7-ecr'
    GITOPS_REPO    = 'https://github.com/AndriiRohovenko/django-gitops.git'
    GITOPS_BRANCH  = 'main'
    GITOPS_VALUES  = 'charts/django-app/values.yaml'
    COMMIT_EMAIL   = 'jenkins@local'
    COMMIT_NAME    = 'Jenkins'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Build and Push Image') {
      steps {
        container('kaniko') {
          withCredentials([
            string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
            string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
          ]) {
            sh '''
              set -eux

              printf '{"credsStore":"ecr-login"}' > /kaniko/.docker/config.json

              /kaniko/executor \
                --context "$WORKSPACE/app" \
                --dockerfile "$WORKSPACE/app/Dockerfile" \
                --destination "$ECR_REPOSITORY:$IMAGE_TAG" \
                --destination "$ECR_REPOSITORY:latest" \
                --snapshot-mode=redo \
                --use-new-run
            '''
          }
        }
      }
    }

    stage('Update GitOps Repo') {
      steps {
        container('git') {
          withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
            sh '''
              set -eux

              apk add --no-cache yq

              rm -rf gitops
              git clone https://AndriiRohovenko:${GITHUB_TOKEN}@github.com/AndriiRohovenko/django-gitops.git gitops
              cd gitops
              git checkout "$GITOPS_BRANCH"

              yq -i '.image.tag = env(IMAGE_TAG)' "$GITOPS_VALUES"

              git config user.email "$COMMIT_EMAIL"
              git config user.name "$COMMIT_NAME"

              git add "$GITOPS_VALUES"
              git commit -m "Update django image tag to $IMAGE_TAG" || true
              git push https://AndriiRohovenko:${GITHUB_TOKEN}@github.com/AndriiRohovenko/django-gitops.git "$GITOPS_BRANCH"
            '''
          }
        }
      }
    }
  }
}