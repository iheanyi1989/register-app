pipeline {
    agent { label 'jenkins-agent' }
    
    tools {
        jdk 'java17'
        maven 'Maven3'
    }
    
    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "iheanyi1989"
        DOCKER_PASS = 'dockerhub'
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
    }
    
    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/iheanyi1989/register-app'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            parallel {
                stage("Unit Tests") {
                    steps {
                        sh "mvn test"
                    }
                }
                
                stage("SonarQube Analysis") {
                    steps {
                        script {
                            withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                                sh "mvn sonar:sonar"
                            }
                        }
                    }
                }
            }
        }

        // Quality Gate stage is commented out
        //    stage("Quality Gate"){
        //        steps {
        //            script {
        //                 waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
        //             }	
        //         }
        //     }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }

                    docker.withRegistry('',DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image iheanyi1989/register-app-pipeline:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
                }
            }
        }

        // Alternative Trivy Scan stage is commented out
        // stage("Trivy Scan") {
        //     steps {
        //         script {
        //             sh '''
        //                 docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \\
        //                 -v $WORKSPACE:/root/.cache/ \\
        //                 aquasec/trivy image --format table \\
        //                 -o /root/.cache/trivy-image-report.txt \\
        //                 --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL \\
        //                 iheanyi1989/register-app-pipeline:latest
        //             '''
        //         }
        //     }
        //     post {
        //         always {
        //             archiveArtifacts artifacts: 'trivy-image-report.txt'
        //             publishHTML target: [
        //                 allowMissing: false, 
        //                 alwaysLinkToLastBuild: true,
        //                 keepAll: true,
        //                 reportDir: './',
        //                 reportFiles: 'trivy-image-report.txt',
        //                 reportName: 'Trivy Scan Report'
        //             ]
        //         }
        //         failure {
        //             error "Trivy scan failed due to HIGH/CRITICAL vulnerabilities found"
        //         }
        //     }
        // }

       stage ('Cleanup Artifacts') {
           steps {
               script {
                   sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                   sh "docker rmi ${IMAGE_NAME}:latest" 
               }
          }
       }

       stage("Trigger CD Pipeline") {
            steps {
                script {
                    sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-34-228-15-121.compute-1.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
                }
            }
       }
   }

    post {
        failure {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed", 
                mimeType: 'text/html',to: "ioncloudjourney@gmail.com"
        }
        success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful", 
                mimeType: 'text/html',to: "ioncloudjourney@gmail.com"
        }      
    }
}