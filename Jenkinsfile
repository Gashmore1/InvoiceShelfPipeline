pipeline {
  agent {
		kubernetes {
			inheritFrom 'default'
			yaml '''
      spec:
        containers:
        - name: buildkit
          image: moby/buildkit
          tty: true
          securityContext:
            allowPrivilegeEscalation: true
            privileged: true
'''
		}
	}

  parameters {
    string(name: 'TAG', defaultValue: '', description: 'Tag to build (leave blank for latest)')
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
    stage('Set Build Tag') {
      steps {
        script {
          env.BUILD_TAG = params.TAG ?: env.LATEST_TAG
        }
      }
    }
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM',
          branches: [[name: "refs/tags/${env.BUILD_TAG}" ]],
          userRemoteConfigs: [[url: 'https://github.com/InvoiceShelf/InvoiceShelf.git']]
        ])
      }
    }
    stage('Configuring Docker') {
      steps {
        container('buildkit') {
         withCredentials([usernamePassword(
          credentialsId: 'dockerPAT',
          usernameVariable: 'DOCKER_USERNAME',
          passwordVariable: 'DOCKER_TOKEN'
        )]){
            sh 'mkdir -p /root/.docker'
            writeFile(
              file:'config.json',
              text: """
              {"auths":{"https://index.docker.io/v1/":{"username":"$DOCKER_USERNAME","password":"$DOCKER_TOKEN"}}}
              """
            )
            sh 'mv config.json /root/.docker/config.json'
          }
        }
      }
    }
    stage('Build Docker Images') {
      parallel {
        stage('Production') {
          steps {
            container('buildkit') {
              sh 'buildctl-daemonless.sh build --frontend dockerfile.v0 -local context=. -local dockerfile=./docker/production/ --output type=image,name=docker.io/gashmore/invoice-shelf:${BUILD_TAG},push=true'
            }
          }
        }
        stage('Development') {
          steps {
            container('buildkit') {
              sh '''
              buildctl-daemonless.sh build \
              --frontend dockerfile.v0 \
              -local context=. \
              -local dockerfile=./docker/development/ \
              --opt build-arg:UID=${USRID:-1000} \
              --opt build-arg:GID=${GRPID:-1000} \
              --output type=image,name=docker.io/gashmore/invoice-shelf:${BUILD_TAG}-dev,push=true
              '''
            }
          }
        }
      }
    }
    stage('Push Latest') {
      when {
        expression { env.BUILD_TAG == env.LATEST_TAG }
      }
      steps {
        container('buildkit') {
          sh '''
          buildctl-daemonless.sh build \
          --frontend dockerfile.v0 \
          -local context=. \
          -local dockerfile=./docker/production/ \
          --output type=image,name=docker.io/gashmore/invoice-shelf:latest,push=true
          '''
        }
      }
    }

    stage('Upload Tag') {
      steps {
          echo "Built image with tag ${env.BUILD_TAG}"
      }
    }
  }
}
