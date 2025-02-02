pipeline {
    agent any

    tools{
        maven 'maven3'
        jdk 'jdk17'
    }

    environment{
        SCANNER_HOME=tool 'sonar-scanner'
    }

    stages {
        stage('Git Ckeckout') {
            steps {
                git branch: 'main', url: 'https://github.com/Shrikesh1996/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Unit Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=EKART -Dsonar.projectName=EKART \
                    -Dsonar.java.binaries=. '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: ' --scan ./', odcInstallation: 'DC'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus Repo') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                        sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build docker image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t shrikesh1996/ekart:latest -f docker/Dockerfile ."
                    }
                }
            }
        }

        stage('Trivy Image Scanning') {
            steps {
                sh "trivy image --format table -o trivy-report.html shrikesh1996/ekart:latest "
            }
        }

        stage('Build push image') {
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push shrikesh1996/ekart:latest"
                    }
                }
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8-token', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://10.0.1.200:6443') {
                  sh "kubectl apply -f deploymentservice.yml -n webapps"
                  sh "kubectl get svc -n webapps"  
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
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'karmoreshrikesh0@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
}
