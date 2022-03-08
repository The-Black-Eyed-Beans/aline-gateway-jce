pipeline {
    agent any

    tools {
        //Using the latest version of Maven
        maven "Maven 3.8.4"
    }
    
    environment {
        withCredentials([string(credentialsId: 'AWS_USER_ID', variable: 'AWS_USER_ID'), string(credentialsId: 'AWS_REGION', variable: 'AWS_REGION')]) {
            AWS_USER_ID                 = AWS_USER_ID
            AWS_REGION                  = AWS_REGION
        }
        MICROSERVICE_IMAGE_NAME     = 'gateway-jce'
        TAG                         = 'latest'
    }

    stages {
        stage('Checkout') {
            steps {
                sh 'git submodule init'
                sh 'git submodule update'
            }
        }
        
        stage('Packaging'){
            steps{
                // Run Maven on a Unix agent.
                sh "mvn clean package -DskipTests"
            }
        }
        
        stage('Sonar Scan'){
            steps {
                withSonarQubeEnv(installationName: 'SonarQube-Server'){
                    sh 'mvn sonar:sonar'
                }
            }
        }
        
        stage('Quality Gate'){
            steps {
                timeout(time: 3, unit: 'MINUTES'){
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Image'){
            steps{
                sh "docker build -t ${MICROSERVICE_IMAGE_NAME}:${TAG} -f Gateway-Dockerfile ."
            }
        }
        
        // stage('Archive Previous Latest'){
        //     //TODO: Versioning for previous builds
        // }
        
        stage('Push Image'){
            steps{
                withAWS(credentials: 'jce-key') {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                    sh "docker tag ${MICROSERVICE_IMAGE_NAME}:${TAG} ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:${TAG}"
                    sh "docker push ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:${TAG}"
                }
            }
        }
    }
    
    post {
        success{
            //Remove image locally
            sh "docker rmi ${MICROSERVICE_IMAGE_NAME}:${TAG}"
            sh "docker rmi ${AWS_USER_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${MICROSERVICE_IMAGE_NAME}:${TAG}"
        }
    }
}
