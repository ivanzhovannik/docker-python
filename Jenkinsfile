String cron_string = BRANCH_NAME == "ci" ? "H 12 * * 1-5" : ""

pipeline {
  agent { label 'linux && !gpu' }
  options {
    disableConcurrentBuilds()
  }
  triggers {
    cron(cron_string)
  }
  environment {
    GIT_COMMIT_SHORT = sh(returnStdout: true, script:"git rev-parse --short=7 HEAD").trim()
    GIT_COMMIT_SUBJECT = sh(returnStdout: true, script:"git log --format=%s -n 1 HEAD").trim()
    GIT_COMMIT_AUTHOR = sh(returnStdout: true, script:"git log --format='%an' -n 1 HEAD").trim()
    GIT_COMMIT_SUMMARY = "${GIT_BRANCH} `<https://github.com/Kaggle/kaggle-python/commit/${GIT_COMMIT}|${GIT_COMMIT_SHORT}>` ${GIT_COMMIT_SUBJECT} - ${GIT_COMMIT_AUTHOR}"
    SLACK_CHANNEL = sh(returnStdout: true, script: "if [[ \"${GIT_BRANCH}\" == \"ci\" ]]; then echo \"#kernels\"; else echo \"#builds\"; fi").trim()
  }

  stages {
    stage('Docker Build') {
      steps {
        slackSend color: 'none', message: "*<${env.BUILD_URL}console|docker build>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
        sh '''#!/bin/bash
          ./build
        '''
      }
    }

    stage('Test Image') {
      steps {
        slackSend color: 'none', message: "*<${env.BUILD_URL}console|test image>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
        sh '''#!/bin/bash
          ./test
        '''
      }
    }

    stage('Push Temporary Image') {
      steps {
        slackSend color: 'none', message: "*<${env.BUILD_URL}console|pushing temporary image>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
        sh '''#!/bin/bash
          ./push gpu-test
        '''
      }
    }

    stage('Test Image on GPU') {
      agent { label 'linux && gpu' }
      steps {
        slackSend color: 'none', message: "*<${env.BUILD_URL}console|testing image on GPU>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
        sh '''#!/bin/bash
          gcloud docker -- pull gcr.io/kaggle-private-byod/python:gpu-test
          docker tag gcr.io/kaggle-private-byod/python:gpu-test kaggle/python-build:latest
          EXPECT_GPU=1 ./test
        '''
      }
    }

    stage('Push Final Image') {
      steps {
        slackSend color: 'none', message: "*<${env.BUILD_URL}console|pushing image>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
        sh '''#!/bin/bash
          ./push staging
        '''
      }
    }
  }

  post {
    failure {
      slackSend color: 'danger', message: "*<${env.BUILD_URL}console|failed>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
    }
    success {
      slackSend color: 'good', message: "*<${env.BUILD_URL}console|passed>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
    }
    aborted {
      slackSend color: 'warning', message: "*<${env.BUILD_URL}console|aborted>* ${GIT_COMMIT_SUMMARY}", channel: env.SLACK_CHANNEL
    }
  }
}