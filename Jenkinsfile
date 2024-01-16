pipeline{
    agent any
    tools {
        jdk 'JDK'
        maven 'Maven'
    }
    environment {
        registry = "holadmex/vproappdock"
        registryCredential = 'Docker-login'
    }
    stages {
        stage ('BUILD THE CODE') {
            steps{
                sh 'mvn clean install -DskipTest'
            }
            post {
                success {
                    echo 'Archiving artifacts now.'
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }
        stage ('UNIT TEST') {
            steps{
                sh 'mvn test'
            }
        }
        stage ('Checkstyle Analysis') {
            steps{
                sh 'mvn checkstyle:checkstyle'
            }
        }
        stage ('Sonar Analysis') {
            environment {
                scannerHome = tool 'sonarqube'
            }
            steps{
                withSonarQubeEnv('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=projectanalysis \
                   -Dsonar.projectName=projectanalysis \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }
        stage ('Quality Gate') {
            steps{
                timeout(time: 2, unit: 'MINUTES') {
                    // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                    // true = set pipeline to UNSTABLE, false = don't
                    waitForQualityGate abortPipeline: true
                }    
            }
        }
        stage('Build App Image') {
            steps {
                script {
                  dockerImage = docker.build registry + ":V$BUILD_NUMBER"
             }
        }
        stage('Upload image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        docker.Image.push("V$BUILD_NUMBER")
                        docker.Image.push('latest')
                    }
                }
            }
        }
        stage('Remove Unused docker image')
            steps{
                sh " docker rmi $registry:V$BUILD_NUMBER"
          }     
        }
        stage ('Kubernetes Deploy')
            agent { label 'KOPS'}
                steps{
                    sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${$BUILD_NUMBER} --namespace prod"
            }
        }    
    }
       
