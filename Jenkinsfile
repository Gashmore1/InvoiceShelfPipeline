pipeline {
  agent {
		kubernetes {
			inheritFrom 'default'
			yaml '''
      spec:
        containers:
        - name: dind
          image: docker:dind
          tty: true
          securityContext:
            allowPrivilegeEscalation: true
            privileged: true
'''
		}
	}

  stages {
    stage('Get Latest Tag') {
      steps {
        script {
          def latestTag = sh(
            script: """
              git ls-remote --tags https://github.com/InvoiceShelf/InvoiceShelf.git |
                awk '{print \$2}' |
                cut -d/ -f3 |
                sort -V |
                tail -n 1
            """,
            returnStdout: true
          ).trim()
          env.LATEST_TAG = latestTag
        }
      }
    }
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: "refs/tags/${env.LATEST_TAG}" ]],
          userRemoteConfigs: [[url: 'https://github.com/InvoiceShelf/InvoiceShelf.git']]
        ])
      }
    }
    stage('Build Docker Images') {
      parallel {
        stage("fontend") {
          steps {
            container('dind') {
              echo "In stage Nested 2 within Branch frontend"
              sh 'docker build -t gashmore/invoice-shelf-mariadb -f docker/production/Dockerfile .'
            }
          }
        }
      }
    }
  }
}
