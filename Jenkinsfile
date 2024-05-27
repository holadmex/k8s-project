pipeline {
    agent any

    tools {
        maven "Maven"
    }

    environment {
        registry = "holadmex/vproappdock123"
        registryCredential = 'docker-login'
    }

    stages {
        stage('FETCH THE CODE FROM GITHUB') {
            steps {
                git branch: 'main', credentialsId: 'git-token', url: 'https://github.com/holadmex/k8s-project.git'
            }
        }

        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
            environment {
                scannerHome = tool 'sonarqube'
            }
            steps {
                withSonarQubeEnv('sonar-env') {
                    sh '''${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=kops-pro \
                        -Dsonar.projectName=kops-pro \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=src/ \
                        -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                        -Dsonar.junit.reportsPath=target/surefire-reports/ \
                        -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                        -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build("${env.registry}:V${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Upload image') {
            steps {
                script {
                    docker.withRegistry('', env.registryCredential) {
                        dockerImage.push("V${env.BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused docker image') {
            steps {
                sh "docker rmi ${env.registry}:V${env.BUILD_NUMBER}"
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${env.registry}:V${env.BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
