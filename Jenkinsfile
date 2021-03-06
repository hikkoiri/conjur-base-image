pipeline {
  agent { label 'executor-v2' }

  parameters {
    booleanParam(name: 'PUBLISH_DOCKERHUB', defaultValue: false,
                 description: 'Publish images to DockerHub')
  }

  triggers {
    cron(getDailyCronString())
  }

  environment {
    TAG = sh(returnStdout: true, script: "git rev-parse --short HEAD | tr -d '\n'")
  }

  stages {
    stage ('Build and push openssl-builder image') {
      steps {
        sh "./openssl-builder/build.sh"
      }
    }
    stage ('Build and push subsequent builder images') {
      parallel {
        stage ('Build and push phusion-ruby-builder image') {
          steps {
            sh "./phusion-ruby-builder/build.sh"
          }
        }
        stage ('Build and push ubuntu-ruby-builder image') {
          steps {
            sh "./ubuntu-ruby-builder/build.sh"
          }
        }
        stage ('Build and tag postgres-client-builder image') {
          steps {
            sh "./postgres-client-builder/build.sh"
          }
        }
        stage ('Build and tag openldap-builder image') {
          steps {
            sh "./phusion-openldap-builder/build.sh"
          }
        }
      }
    }
    stage ('Build and Test fips base images') {
      parallel {
        stage ('Build and Test phusion-ruby-fips image') {
          steps {
            buildAndTestImage('phusion-ruby-fips')
          }
        }
        stage ('Build and Test ubuntu-ruby-fips image') {
          steps {
            buildAndTestImage('ubuntu-ruby-fips')
          }
        }
      }
    }
    stage ('Push images') {
      steps {
        sh "./phusion-ruby-fips/push.sh ${TAG} registry.tld"
        sh "./ubuntu-ruby-fips/push.sh ${TAG} registry.tld"
      }
    }
    stage ('Publish images') {
      when { 
        anyOf {
          triggeredBy 'TimerTrigger'
          expression { params.PUBLISH_DOCKERHUB }
        }
      }      
      
      steps {
        sh "./phusion-ruby-fips/push.sh ${TAG}"
        sh "./ubuntu-ruby-fips/push.sh ${TAG}"
      }
    }
  }

  post {
    always {
      archiveArtifacts allowEmptyArchive: true, artifacts: 'test-results/**/*.json', fingerprint: true
      cleanupAndNotify(currentBuild.currentResult, "#development")
    }
  }
}

def buildAndTestImage(name) {
  sh "./${name}/build.sh ${TAG}"
  sh "./${name}/test.sh ${TAG}"
  scanAndReport("${name}:${TAG}", "HIGH", false)
  scanAndReport("${name}:${TAG}", "NONE", true)
}
