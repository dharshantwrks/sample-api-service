pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  environment {
    APP_NAME = "sample-api-service"
    IMAGE_REGISTRY = "rmkanda"
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh './mvnw install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
      }
    }
//     stage('Secrets scanner') {
//           steps {
//             container('trufflehog') {
//               git(
//        url: 'https://github.com/dharshantwrks/sample-api-service/',
//        credentialsId: '87beede8-7819-4f1b-b35e-4298861304fe',
//        branch: "main"
//     )
//               sh 'cd sample-api-service && ls -al'
//               sh 'cd sample-api-service && trufflehog .'
//               sh 'rm -rf sample-api-service'
//             }
//           }
//         }
    stage('Build') {
      steps {
        container('maven') {
          sh './mvnw package -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh './mvnw test'
            }
          }
        }
        stage('SCA - Dependency Checker') {
            steps {
              container('maven') {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                  sh './mvnw org.owasp:dependency-check-maven:check'
                }
                }
            }
            post {
              always {
                archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: false
                dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
              }
            }
          }
      }
    }
    stage('OSS License Checker') {
             steps {
               container('licensefinder') {
                 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                   sh '''#!/bin/bash --login
                         /bin/bash --login
                         rvm use default
                         gem install license_finder
                         license_finder
                       '''
                 }
               }
             }
           }
    stage('Package') {
      steps {
        container('docker-tools') {
          sh "docker build . -t ${APP_NAME}"
        }
        
      }
      
    }
    stage('Artefact Analysis') {
      parallel {
        stage('Contianer Scan') {
          steps {
            container('docker-tools') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh "grype ${APP_NAME}"
              }
            }
          }
        }
        stage('Container Audit') {
          steps {
            container('docker-tools') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh "dockle ${APP_NAME}"
              }
            }
          }
        }
      }
    }
    stage('Publish') {
      steps {
        container('docker-tools') {
          echo "Publishing docker image"
          // sh "docker push ${APP_NAME}"
        }
      }
    }
    stage('Deploy to Dev') {
      steps {
        container('docker-tools') {
          echo "Deploying the app"
          // sh "kubectl apply -f k8s.yaml"
        }
      }
    }
    stage('Promote to Prod') {
      steps {
        container('docker-tools') {
          echo "Promote to Prod"
        }
      }
    }
  }
}
