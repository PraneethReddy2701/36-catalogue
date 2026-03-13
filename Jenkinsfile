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
        timeout(time: 30, unit: 'MINUTES')
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
/*         stage('Sonar Scan'){
            environment{
                scannerHome = tool 'sonar-8.0'
            }
            steps{
                // sonar server environment
                withSonarQubeEnv('sonar-8.0') { 
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }
        // Enable webhook in sonarqube server and wait for sonarscanner to send the results to jenkins
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') { // Optional: adds a timeout to prevent indefinite waiting
                    waitForQualityGate abortPipeline: true 
                }
            }
        } */
        stage('Check Dependabot Alerts') {
            environment { 
                GITHUB_TOKEN = credentials('github-token')
            }
            steps {
                script {
                    // Fetch alerts from GitHub
                    def response = sh(
                        script: """
                            curl -s -H "Accept: application/vnd.github+json" \
                                 -H "Authorization: token ${GITHUB_TOKEN}" \
                                 https://api.github.com/repos/PraneethReddy2701/36-catalogue/dependabot/alerts
                        """,
                        returnStdout: true
                    ).trim()

                    // Parse JSON
                    def json = readJSON text: response

                    // Filter alerts by severity
                    def criticalOrHigh = json.findAll { alert ->
                        def severity = alert?.security_advisory?.severity?.toLowerCase()
                        def state = alert?.state?.toLowerCase()
                        return (state == "open" && (severity == "critical" || severity == "high"))
                    }

                    if (criticalOrHigh.size() > 0) {
                        error "❌ Found ${criticalOrHigh.size()} HIGH/CRITICAL Dependabot alerts. Failing pipeline!"
                    } else {
                        echo "✅ No HIGH/CRITICAL Dependabot alerts found."
                    }
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
                    //imageURL = "${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}"
                    //echo 'Image Url is : ${imageURL}'
                }
            }
        }
        stage('Check ECR Vulnerabilities') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {

                        // Get digest of the pushed image
                        def digest = sh(
                            script: """
                                aws ecr describe-images \
                                --repository-name ${PROJECT}/${COMPONENT} \
                                --image-ids imageTag=${appVersion} \
                                --query 'imageDetails[0].imageDigest' \
                                --output text
                            """,
                            returnStdout: true
                        ).trim()

                        echo "Image Digest: ${digest}"

                        // Wait for scan to complete
                        timeout(time: 2, unit: 'MINUTES') {
                            waitUntil {
                                def status = sh(
                                    script: """
                                        aws ecr describe-image-scan-findings \
                                        --repository-name ${PROJECT}/${COMPONENT} \
                                        --image-id imageDigest=${digest} \
                                        --query 'imageScanStatus.status' \
                                        --output text
                                    """,
                                    returnStdout: true
                                ).trim()

                                echo "Scan status: ${status}"

                                return status == "COMPLETE"
                            }
                        }

                        // Fetch scan findings
                        def findings = sh(
                            script: """
                                aws ecr describe-image-scan-findings \
                                --repository-name ${PROJECT}/${COMPONENT} \
                                --image-id imageDigest=${digest} \
                                --region ${REGION} \
                                --output json
                            """,
                            returnStdout: true
                        ).trim()

                        def json = readJSON text: findings
                        def counts = json.imageScanFindings.findingSeverityCounts ?: [:]

                        def high = counts.HIGH ?: 0
                        def critical = counts.CRITICAL ?: 0

                        echo "HIGH vulnerabilities: ${high}"
                        echo "CRITICAL vulnerabilities: ${critical}"

                        if (high > 0 || critical > 0) {
                            error(" Pipeline failed due to HIGH/CRITICAL vulnerabilities")
                        } else {
                            echo " No HIGH or CRITICAL vulnerabilities found"
                        }
                    }
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
                        string(name: 'deploy_to', value: 'dev')
                        //string(name: 'imageURL', value: "${imageURL}"),
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