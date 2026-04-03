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
            allowPrivilegeEscalation: false
            privileged: false
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
       withCredentials([usernamePassword(credentialsId: 'dockerPAT',
        usernameVariable: 'DOCKER_USERNAME',
        passwordVariable: 'DOCKER_TOKEN')]){
          sh 'mkdir -p ~/.docker'
          sh 'echo "{\"auths\":{\"docker.io\":{\"username\":\"$DOCKER_USERNAME\",\"password\":\"$DOCKER_TOKEN\"}}}" > ~/.docker/config.json'
        }
      }
    }
    stage('Build Docker Images') {
      parallel {
        stage('frontend') {
          steps {
            container('buildkit') {
              sh 'buildctl-daemonless.sh build --frontend dockerfile.v0 -local context=. -local dockerfile=./docker/production/ --output type=image,name=docker.io/gashmore/invoice-shelf-mariadb,push=true'
            }
          }
        }
      }
    }
  }
}
