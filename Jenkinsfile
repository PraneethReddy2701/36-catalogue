pipeline{
    // agent any
    agent {
        node {
            label 'AGENT-1'
        }
    }
    // Pre-Build
    environment{
        appVersion = ''
        REGION = "us-east-1"
        ACC_ID = "235270182665"
        PROJECT = "roboshop"
        COMPONENT = "catalogue"
        imageURL = ''
    }
    options {
        timeout(time: 30, unit: 'MMINUTES')
        disableConcurrentBuilds()
    }
    parameters {
        booleanParam(name: 'deploy', defaultValue: false, description: 'Toggle this value')
    }
   
   // Build
    stages{
        stage('Read package.json'){
            steps{
                script{
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "The package version is: ${appVersion}"
                    
                }
            }
        }
        stage('Install Dependencies'){
            steps{
                script{
                    sh """
                        npm install
                    """
                }
            }
        }
        stage('Unit Testing'){
            steps{
                script{
                    sh """
                        echo 'unit testing'
                    """
                }
            }
        }
        stage('Docker Build'){
            steps{
                script{
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com
                            docker build -t ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} .
                            docker push ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                        """
                    }
                    imageURL = "${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}"
                    echo 'Image Url is : ${imageURL}'
                }
            }
        }
        stage('Trigger Deploy'){
            when {
                expression {
                    params.deploy // Access the parameter value via params.CHOICE
                }
            }
            steps{
                script{
                    build job: 'catalogue-cd', 
                    parameters: [
                        string(name: 'appVersion', value: "${appVersion}"),
                        string(name: 'imageURL', value: "${imageURL}"),
                        string(name: 'deploy_to', value: dev)
                    ],
                    propagate: false,   // even if SG fails, VPC will not fail
                    wait: false         // VPC doesnot wait for the SG pipeline to complete
                }
            }
        }
    }

   // Post-Build
    post { 
        always { 
            echo 'I will always say Hello pipeline!'
            deleteDir()
        }
        success { 
            echo 'Hello pipeline is success!'
        }
        failure { 
            echo 'Hello pipeline is failure!'
        }
    }
}