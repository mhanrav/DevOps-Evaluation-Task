pipeline {
  agent any

  // set global environment vars (override in multibranch/job config if needed)
  environment {
    IMAGE_NAME = "${env.IMAGE_NAME ?: 'myapp'}"
    TAG        = "${env.TAG ?: env.BUILD_NUMBER}"
  }

  options {
    // keep builds' console logs reasonable
    ansiColor('xterm')
    timeout(time: 60, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Install & Test') {
      steps {
        // run npm install & tests; ensure tests produce junit xml if you want testResults
        sh 'npm ci'
        sh 'npm test'
      }
    }

    stage('Build Image') {
      steps {
        script {
          // Build docker image locally
          sh "docker build -t ${IMAGE_NAME}:${TAG} ."
        }
      }
    }

    stage('Push (optional)') {
      when {
        expression { env.BRANCH_NAME ==~ /^(main|release\/.*)$/ }
      }
      steps {
        echo "Push to registry (configure credentials block below if you want to actually push)"
        // Example using Jenkins credentials (uncomment and configure credentials in Jenkins)
        /*
        withCredentials([usernamePassword(credentialsId: 'docker-reg-cred', usernameVariable: 'REG_USER', passwordVariable: 'REG_PSW')]) {
          sh "docker tag ${IMAGE_NAME}:${TAG} myregistry/${IMAGE_NAME}:${TAG}"
          sh "echo $REG_PSW | docker login myregistry -u $REG_USER --password-stdin"
          sh "docker push myregistry/${IMAGE_NAME}:${TAG}"
        }
        */
      }
    }

    stage('Deploy to environment') {
      steps {
        script {
          // Guard: Branch name from Multibranch pipeline
          def branch = env.BRANCH_NAME ?: 'local'

          // helper to remove existing container if exists
          def removeIfExists = { name ->
            sh "if [ \$(docker ps -a -q -f name=^/${name}$ | wc -l) -gt 0 ]; then docker rm -f ${name} || true; fi"
          }

          if (branch == 'main' || branch ==~ /^release\/.*/) {
            echo "Deploying to UAT"
            removeIfExists('app_uat')
            // map fixed host port for UAT
            sh "docker run -d --name app_uat -e ENV=uat -p 3003:3000 ${IMAGE_NAME}:${TAG}"
          } else if (branch == 'develop') {
            echo "Deploying to QA"
            removeIfExists('app_qa')
            sh "docker run -d --name app_qa -e ENV=qa -p 3002:3000 ${IMAGE_NAME}:${TAG}"
          } else if (branch ==~ /^feature\/.*/ || branch == 'feature') {
            echo "Deploying to Dev (feature branch) with ephemeral host port"
            // make a safe container name and run with -P to auto-assign host port to avoid collisions
            def safeName = branch.replaceAll('[^A-Za-z0-9_.-]', '_')
            def containerName = "app_${safeName}"
            removeIfExists(containerName)

            // Run container and let Docker publish a random host port (-P).
            sh "docker run -d --name ${containerName} -e ENV=dev -P ${IMAGE_NAME}:${TAG}"

            // retrieve mapped host port for container's 3000/tcp and print for user
            def hostPort = sh(script: "docker port ${containerName} 3000/tcp | awk -F':' '{print \$2}'", returnStdout: true).trim()
            echo "Feature branch deployed as container ${containerName} on host port: ${hostPort}"
          } else if (branch ==~ /^hotfix\/.*/) {
            echo "Deploying hotfix to QA and UAT (example flow)"
            removeIfExists('app_qa')
            sh "docker run -d --name app_qa -e ENV=qa -p 3002:3000 ${IMAGE_NAME}:${TAG}"
            removeIfExists('app_uat')
            sh "docker run -d --name app_uat -e ENV=uat -p 3003:3000 ${IMAGE_NAME}:${TAG}"
          } else {
            echo "Unrecognized branch pattern — defaulting to Dev"
            removeIfExists('app_dev')
            sh "docker run -d --name app_dev -e ENV=dev -p 3001:3000 ${IMAGE_NAME}:${TAG}"
          }
        } // script
      } // steps
    } // stage Deploy

  } // stages

  post {
    always {
      // Publish junit results if your test runner writes xml (configure test reporter)
      junit allowEmptyResults: true, testResults: 'test-results/**/*.xml'
      // Archive build artifacts if any (adjust pattern)
      archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
    }

    success {
      echo "Build succeeded: ${env.BUILD_URL}"
    }

    failure {
      echo "Build failed: ${env.BUILD_URL}"
    }
  }
}
