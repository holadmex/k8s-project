pipeline {

    agent any
/*
	tools {
        maven "Maven"
    }
*/
    environment {
        registry = "holadmex/vproappdock"
        registryCredential = 'Docker-login'
    }
    stages{

        stage('BUILD'){
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

        stage ('CODE ANALYSIS WITH CHECKSTYLE') {
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

                timeout(time: 10, unit: 'MINUTES') {
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

        stage('Remove Unused docker image') {
            steps{
                sh " docker rmi $registry:V$BUILD_NUMBER"
            }     
        }
        
        stage ('Kubernetes Deploy') {
          agent { label 'KOPS'}
            steps{
              sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${$BUILD_NUMBER} --namespace prod"
            }
        }    
    }
}    
    
       
