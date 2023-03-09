pipeline {
  agent any
  stages {
    stage('Check Code Quality') {
      steps {
          script {
            docker.image('python:3.9').inside {c ->
              sh '''
              python -m venv .venv
              . .venv/bin/activate
              pip install pylint
              pip install -r requirements.txt
              pylint --exit-zero --report=y --output-format=json:pylint-report.json,colorized ./*.py
              '''
              publishHTML target : [
                    allowMissing: true,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: './',
                    reportFiles: 'pylint-report.json',
                    reportName: 'pylint Scan',
                    reportTitles: 'pylint Scan'
                ]
            }
          }
      }
    }
    stage('build') {
      steps {
        script {
          checkout scm
          def customImage = docker.build("${registry}:${env.BUILD_ID}")
        }

      }
    }
    stage('unit-test') {
      steps {
        script {
          docker.image("${registry}:${env.BUILD_ID}").inside {c ->
          sh 'python app_test.py'}
        }

      }
    }
    stage('http-test') {
      steps {
        script {
          docker.image("${registry}:${env.BUILD_ID}").withRun('-p 9005:9000') {c ->
          sh "sleep 5; curl -i http://localhost:9005/test_string"}
        }

      }
      post {
        success {
          echo 'something went wrong'
        }
        failure {
          echo 'success'
        }
      }
    }
    stage('Scan Docker Image for Vulnerabilities') {
      steps {
        script {
          def vulnerabilities = sh(script: "trivy image --exit-code 0 --severity HIGH,MEDIUM,LOW --no-progress ${registry}:${env.BUILD_ID}", returnStdout: true).trim()
          echo "Vulnerability Report:\n${vulnerabilities}"
        }
      }
    }
    stage('Publish') {
      steps {
        script {
          docker.withRegistry('', 'docker-hub') {
            docker.image("${registry}:${env.BUILD_ID}").push('latest')
            docker.image("${registry}:${env.BUILD_ID}").push("${env.BUILD_ID}")
          }
        }

      }
    }
    stage('Deploy') {
      steps {
        sh 'docker stop flask-app || true; docker rm flask-app || true;docker run -d --name flask-app -p 9000:9000 yucherpak/flask:latest'
      }
    }

  }
  environment {
    registry = 'yucherpak/flask'
  }
}
