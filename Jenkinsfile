pipeline {
    agent any

    tools {
        jdk   'jdk21'          // must match Manage Jenkins -> Tools names exactly
        maven 'maven3'
    }

    environment {                               // was misspelled "enviornment"
        SCANNER_HOME = tool 'sonar-scanner'     // SonarQube Scanner tool name
        IMAGE        = 'affan2310/boardgame'
        TAG          = "${BUILD_NUMBER}"        // unique tag per build, not just :latest
    }

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '15'))
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Affan2310/Boardgame.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn -B compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('File System Scan') {
            steps {
                sh 'trivy fs --format table -o trivy-fs-report.txt .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=BoardGame \
                          -Dsonar.projectKey=BoardGame \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    // true = a failed gate stops the pipeline (set false only while learning)
                    waitForQualityGate abortPipeline: true, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B package -DskipTests'
            }
        }

        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk21', maven: 'maven3', traceability: true) {
                    sh 'mvn -B deploy -DskipTests'
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh "docker build -t ${IMAGE}:${TAG} -t ${IMAGE}:latest ."
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.txt ${IMAGE}:${TAG}"
            }
        }

        stage('Push Docker Image') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/') {
                    sh "docker push ${IMAGE}:${TAG}"
                    sh "docker push ${IMAGE}:latest"
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '',
                               credentialsId: 'k8-cred', namespace: 'webapps',
                               restrictKubeConfigAccess: false, serverUrl: 'https://172.31.14.12:6443') {
                    // point the manifest at this build's image so every run actually rolls out
                    sh """
                        sed -i -E 's|(image:[[:space:]]*).*|\\1${IMAGE}:${TAG}|' deployment-service.yaml
                        grep -n 'image:' deployment-service.yaml
                        kubectl apply -f deployment-service.yaml -n webapps
                    """
                }
            }
        }

        stage('Verify the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '',
                               credentialsId: 'k8-cred', namespace: 'webapps',
                               restrictKubeConfigAccess: false, serverUrl: 'https://172.31.14.12:6443') {
                    sh '''
                        for d in $(kubectl get deploy -n webapps -o name); do
                            kubectl rollout status "$d" -n webapps --timeout=180s
                        done
                        kubectl get pods -n webapps -o wide
                        kubectl get svc -n webapps
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-*.txt', allowEmptyArchive: true
            script {
                def jobName        = env.JOB_NAME
                def buildNumber    = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.currentResult      // never null, unlike currentBuild.result
                def bannerColor    = pipelineStatus == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus}</h3>
                    </div>
                    <p>Image: ${env.IMAGE}:${env.TAG}</p>
                    <p>Check the <a href="${env.BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus}",
                    body: body,
                    to: 'affan.m@codilar.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-*.txt'
                )
            }
        }
    }
}
