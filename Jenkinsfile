pipeline {
  agent any

  environment {
    IMAGE_NAME = "jenkins-cicd-sample-app"
    // don't try to compute TAG here from env.BRANCH_NAME (may be null at parse time)
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare') {
      steps {
        script {
          // Make branch/tag safe for later use
          env.BRANCH_SAFE = env.BRANCH_NAME ?: 'local'
          env.TAG = env.BRANCH_SAFE.replaceAll('[^a-zA-Z0-9_\\-./]', '_')
          echo "Branch: ${env.BRANCH_SAFE}, TAG: ${env.TAG}"
        }
      }
    }

    stage('Install & Test') {
      steps {
        // Ensure Node is available on this agent
        script {
          echo "Ensure node/npm present on agent and tests produce JUnit XML into test-results/"
        }
        sh 'npm ci'
        sh 'npm test'
      }
    }

    stage('Build Image') {
      steps {
        script {
          echo "Building Docker image ${IMAGE_NAME}:${TAG}"
        }
        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
      }
    }

    stage('Push (optional)') {
      when {
        expression {
          def b = env.BRANCH_SAFE
          return (b ==~ /main/ || b ==~ /release\\/.*|release.*/ )
        }
      }
      steps {
        echo 'Push to registry step (configure credentials & registry)'
        // Example (uncomment and configure credentials properly):
        // withCredentials([usernamePassword(credentialsId: 'my-registry-creds', usernameVariable: 'REG_USER', passwordVariable: 'REG_PW')]) {
        //   sh "docker login -u $REG_USER -p $REG_PW myregistry.example.com"
        //   sh "docker tag ${IMAGE_NAME}:${TAG} myregistry.example.com/${IMAGE_NAME}:${TAG}"
        //   sh "docker push myregistry.example.com/${IMAGE_NAME}:${TAG}"
        // }
      }
    }

    stage('Deploy to environment') {
      steps {
        script {
          def branch = env.BRANCH_SAFE
          echo "Deploying for branch ${branch}"

          if (branch == 'main' || branch ==~ /release\/.*/) {
            echo "Deploying to UAT"
            sh "docker rm -f app_uat || true"
            sh "docker run -d --name app_uat -e ENV=uat -p 3003:3000 ${IMAGE_NAME}:${TAG}"
          } else if (branch == 'develop') {
            echo "Deploying to QA"
            sh "docker rm -f app_qa || true"
            sh "docker run -d --name app_qa -e ENV=qa -p 3002:3000 ${IMAGE_NAME}:${TAG}"
          } else if (branch ==~ /feature\/.*/ || branch == 'feature') {
            echo "Deploying to Dev (feature branch)"
            def safeName = branch.replaceAll('[^a-zA-Z0-9]', '_')
            sh "docker rm -f app_${safeName} || true"
            // NOTE: if you need to run multiple feature branches in parallel on same host,
            // you'll need unique ports per branch (not done here).
            sh "docker run -d --name app_${safeName} -e ENV=dev -p 3001:3000 ${IMAGE_NAME}:${TAG}"
          } else if (branch ==~ /hotfix\/.*/ ) {
            echo "Deploying hotfix to QA and UAT"
            sh "docker rm -f app_qa || true"
            sh "docker run -d --name app_qa -e ENV=qa -p 3002:3000 ${IMAGE_NAME}:${TAG}"
            sh "docker rm -f app_uat || true"
            sh "docker run -d --name app_uat -e ENV=uat -p 3003:3000 ${IMAGE_NAME}:${TAG}"
          } else {
            echo "Unrecognized branch pattern — defaulting to Dev"
            sh "docker rm -f app_dev || true"
            sh "docker run -d --name app_dev -e ENV=dev -p 3001:3000 ${IMAGE_NAME}:${TAG}"
          }
        }
      }
    }
  } // stage

  post {
    always {
      junit allowEmptyResults: true, testResults: 'test-results/**/*.xml'
      archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
    }
  }
}
