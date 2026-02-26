pipeline {
    agent any
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }
    environment {SCANNER_HOME = tool 'sonar-scanner'}
    stages {
        stage('git checkout/git clone') {
            steps {
               git branch: 'main', url: 'https://github.com/Saumya-km/Boardgame.git'
            }
        }
        stage('compile source code') {
            steps {
               sh "mvn compile"
            }
        }
        stage('execute test cases') {
            steps {
               sh "mvn test"
            }
        }
         stage('Scan System Vulnerability') {
          steps {
            sh "trivy fs --format table -o trivy-fs-report.html ."
         }
      }
      stage('SonarQube Analysis') {
        steps {
            withSonarQubeEnv('sonar') {
              sh '''
                $SCANNER_HOME/bin/sonar-scanner \
               -Dsonar.projectName=BoardGame \
              -Dsonar.projectKey=BoardGame \
              -Dsonar.java.binaries=.
             ''' }
         }
      }
       stage('Quality Gate') {
          steps {
           script {
    waitForQualityGate(
        abortPipeline: false,
        credentialsId: 'sonar-token' )
           }
         }
      }
     
      stage('Upload or public artifact in nexus repo') {
        steps{
          withMaven(globalMavenSettingsConfig: 'global-setting', jdk: 'jdk17', maven: 'maven3', traceability: true) {
              sh "mvn deploy"
             }
         }
      }
     stage('Build Docker Image') {
       steps {
          script {
            sh 'docker build -t skuma011/boardgame:latest .'
            }
          }
       }
      stage("Docker Image Scan") {
         steps {
            sh "trivy image --format table -o trivy-image-report.html skuma011/boardgame:latest"
            }
        }

     stage("Push Docker Image to Docker Hub") {
        steps {
          script {
              withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                  sh "docker push skuma011/boardgame:latest"
                  }
              }
          }
       }
       stage("deploy in kubernetes") {
         steps {
            withKubeConfig(caCertificate: '', clusterName: 'Kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.5.120:6443') {
                     sh "kubectl apply -f deployment-svc.yml" }
          }
       }
       stage('Verify Deployment') {
           steps {
             script {
            withKubeConfig(caCertificate: '', clusterName: 'Kubernetes', contextName: '', credentialsId: 'k8s-cred', namespace: 'boardgame', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.5.120:6443') {
                     sh "kubectl get pods -n boardgame"
                     sh "kubectl get svc -n boardgame"}
                 }
             }
        }
    }
post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
            <html>
            <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                        <h3 style="color:white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                </div>
            </body>
            </html>
            """

            emailext(
                subject: "${jobName} - Build ${buildNumber} - Status: ${pipelineStatus}",
                body: body,
                to: "kmsaumya0041@gmail.com",
                from: "singh.saumya0208@gmail.com",
                replyTo: "singh.saumya0208@gmail.com",
                mimeType: 'text/html',
                attachLog: true,
                attachmentPattern: 'trivy.image-report.html'
            )
        }
    }
  }
}
