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
                sh "trivy image shrikesh1996/ekart:latest > trivy-report.html"
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
}
