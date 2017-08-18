//ldop-gerrit/Jenkinsfile
pipeline {
    agent none

    stages {
      stage('hadolint-lint'){
          agent {
              docker {
                  image "lukasmartinelli/hadolint"
                  args "-u root"
              }
          }
          steps {
              sh 'hadolint Dockerfile || true'
          }
         post {
              changed {
                  script {
                      subject = "failure: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
                  }
                  slackSend (color: '#FF0000', message: "${subject} (${env.BUILD_URL})")
              }
          }
      }
      stage('dockerlint-lint'){
          agent {
              docker {
                  image "redcoolbeans/dockerlint"
              }
          }
          steps {
              sh 'dockerlint -f Dockerfile || true'
          }
         post {
              changed {
                  script {
                      subject = "failure: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
                  }
                  slackSend (color: '#FF0000', message: "${subject} (${env.BUILD_URL})")
              }
          }
      }
      stage('dockerfile-lint'){
          agent {
              docker {
                  image "projectatomic/dockerfile-lint"
                  args "-u root"
              }
          }
          steps {
              sh 'dockerfile_lint -f Dockerfile || true'
          }
         post {
              changed {
                  script {
                      subject = "failure: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
                  }
                  slackSend (color: '#FF0000', message: "${subject} (${env.BUILD_URL})")
              }
          }
      }
      stage('ldop-gerrit-validate'){
          agent any
          steps {
              sh "echo \$(git tag --sort version:refname | tail -1) > result"
              script {
                  TAG = readFile 'result'
                  TAG = TAG.trim()
              }
              doesVersionExist('liatrio', 'ldop-gerrit', "${TAG}") 
          }
      }
      stage('ldop-gerrit-build'){
          agent any
          steps {
              sh "docker login -u ${env.USERNAME} -p ${env.PASSWORD}"
              sh "docker build -t chadliatrio/ldop-gerrit:${env.BRANCH_NAME} ."
              sh "docker push chadliatrio/ldop-gerrit:${env.BRANCH_NAME}"
          }
      }
      stage('ldop-integration-testing'){
          agent {
            docker {
              image "hashicorp/terraform:full"
              args "-u root"
            }
          }
          steps {
              git branch: 'master', url: 'https://github.com/liatrio/ldop-docker-compose'
              sh "echo \$(pwd) > result"
              script {
                DIR = readFile 'result'
                DIR = DIR.trim()
              }
              testSuite("ldop-gerrit", "${TAG}", "${DIR}")
              sh "export TF_VAR_branch_name=\"${env.BRANCH_NAME}\""
              sh '''export AWS_ACCESS_KEY_ID=ACCESS_KEY_HERE &&
                    export AWS_SECRET_ACCESS_KEY=SECRET_KEY_HERE &&
                    sed -i 's/timeout 30m/timeout -t 1800/g' test/integration/run-integration-test.sh &&
                    bash test/validation/validation.sh &&
                    cd test/integration/ &&
                    terraform init &&
                    bash run-integration-test.sh'''
          }
      }
      stage('ldop-image-deploy'){
          agent any
          steps {
              sh "docker login -u ${env.USERNAME} -p ${env.PASSWORD}"
              sh "docker tag chadliatrio/ldop-gerrit:${env.BRANCH_NAME} chadliatrio/ldop-gerrit:${TAG}"
              sh "docker push chadliatrio/ldop-gerrit:${TAG}"
          }
      }
  }
}
