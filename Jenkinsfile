def reqMap = [
    'all':   [ zip: 'lambda_package.zip', exclude: '.' ],
    'core':  [ zip: 'core.zip', exclude: "**/numpy/** **/pandas/**" ],
    'abc': [ zip: 'uscls.zip', exclude: "**/six/** **/urllib3/** **/numpy/** **/pandas/**" ]
]

def setEnvParamMap(mapEnv, propertyName, paramName) {
    if(env[paramName]) {
        mapEnv.put(propertyName, env[paramName])
    } else {
        mapEnv.put(propertyName, 'false')
    }
    mapEnv[propertyName] = mapEnv[propertyName].toBoolean()
}

def jksEnv = [:]
setEnvParamMap(jksEnv, 'skipSonar', 'JXS_SKIP_SONAR_SCAN')

def RELEASE_VER = ''
def jobEnv = [:]
setEnvParamMap(jobEnv, 'skipUnitTests', 'SKIP_UNIT_TESTS')
setEnvParamMap(jobEnv, 'skipSonar', 'SKIP_SONAR_SCAN')

def buildContinue = true
def baseLayers = ['arn:aws:lambda:us-east-1:336992948345:layer:AMSSDXPands-Python39:29']

pipeline {
    agent {
        label 'any'
    }
    
    parameters {
        string defaultValue: 'develop', description: 'Select git branch', name: 'GIT_BRANCH', trim: true
        choice(name: 'ENV', choices: ['devtest', 'stage', 'prod'], description: 'Choose deployment environment')
    }
    
    environment {
        BASE_FUNCTION_NAME = 'lambda-function'
        REGION = 'us-east-1'
        ZIP_FILE = 'lambda_package.zip'
        ARN_FILE = 'lambda_arn.txt'
        AWS_ACCOUNT_ID = 'your-aws-account-id'
    }
    
    stages {
        stage('Clone Repository') {
            steps {
                cleanWs()
                checkout([$class: 'GitSCM',
                    branches: [[name: "${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/AngelsTech/aws-lambda-demo.git',
                        credentialsId: '66142c87-d271-41f4-82d1-cbbee8e844d0'
                    ]]
                ])
            }
        }
        
        stage('Install Requirements') {
            steps {
                script {
                    sh '''
                  echo "Installing Python dependencies into test/devtest..."
                  pip install -r test/devtest/requirements.txt -t test/devtest
                '''
            }
        }
        
        stage('Package Lambda') {
            steps {
                sh '''
                    echo "Zipping all required files for Lambda deployment..."
                    zip -r $ZIP_FILE test \
                        -x "**/.git/*" \
                        -x "**/README.md" \
                        -x "**/Jenkinsfile"

                    echo "Listing contents of the zip file to verify structure:"
                    unzip -l $ZIP_FILE
                '''
            }
        }
        
        stage('Publish Layers') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        reqMap.each { layer, config ->
                            if (layer != 'all') {
                                sh """
                                    echo "INFO: Publishing ${layer} layer to AWS Lambda..."
                                    aws lambda publish-layer-version \
                                        --layer-name ${layer} \
                                        --zip-file fileb://${config.zip} \
                                        --compatible-runtimes python3.9 \
                                        --region ${env.REGION}
                                """
                            }
                        }
                    }
                }
            }
        }
        
        stage('Update Lambda Function') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        def FUNCTION_NAME = "${env.BASE_FUNCTION_NAME}-${params.ENV}"
                        
                        def coreLayerArn = sh(
                            script: """
                                aws lambda list-layer-versions \
                                    --layer-name core \
                                    --region ${env.REGION} \
                                    --query 'LayerVersions[0].LayerVersionArn' \
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()
                        
                        def usclsLayerArn = sh(
                            script: """
                                aws lambda list-layer-versions \
                                    --layer-name uscls \
                                    --region ${env.REGION} \
                                    --query 'LayerVersions[0].LayerVersionArn' \
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()
                        
                        def allLayers = [coreLayerArn, usclsLayerArn] + baseLayers
                        
                        sh """
                            aws lambda update-function-configuration \
                                --function-name "${FUNCTION_NAME}" \
                                --region ${env.REGION} \
                                --layers ${allLayers.join(' ')}
                        """
                        
                        sh """
                            aws lambda update-function-code \
                                --function-name "${FUNCTION_NAME}" \
                                --region ${env.REGION} \
                                --zip-file fileb://${env.ZIP_FILE}
                        """
                    }
                }
            }
        }
        
        stage('Prepare Lambda Configuration') {
            steps {
                script {
                    def FUNCTION_NAME = "${env.BASE_FUNCTION_NAME}-${params.ENV}"
                    
                    echo "INFO: Waiting for Lambda function configuration update to complete..."
                    
                    def maxRetries = 30
                    def retryCount = 0
                    def status = ''
                    
                    while (retryCount < maxRetries) {
                        status = sh(
                            script: """
                                aws lambda get-function-configuration \
                                    --function-name "${FUNCTION_NAME}" \
                                    --region ${env.REGION} \
                                    --query 'State' \
                                    --output text 2>/dev/null || echo "Pending"
                            """,
                            returnStdout: true
                        ).trim()
                        
                        echo "Current Lambda state: ${status} (Attempt ${retryCount + 1}/${maxRetries})"
                        
                        if (status == 'Active') {
                            echo "Lambda function is now Active"
                            break
                        } else if (status == 'Failed') {
                            error("Lambda function update failed")
                        }
                        
                        sleep(10)
                        retryCount++
                    }
                    
                    if (status != 'Active') {
                        error("Lambda function did not reach Active state after ${maxRetries * 10} seconds")
                    }
                }
            }
        }
        
        stage('Setup CloudWatch Schedule') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        def FUNCTION_NAME = "${env.BASE_FUNCTION_NAME}-${params.ENV}"
                        def ruleName = "lambda-schedule-${params.ENV}"
                        
                        def functionArn = sh(
                            script: """
                                aws lambda get-function \
                                    --function-name "${FUNCTION_NAME}" \
                                    --region ${env.REGION} \
                                    --query 'Configuration.FunctionArn' \
                                    --output text
                            """,
                            returnStdout: true
                        ).trim()
                        
                        sh """
                            aws events put-rule \
                                --name ${ruleName} \
                                --schedule-expression "rate(5 minutes)" \
                                --state ENABLED \
                                --region ${env.REGION}
                        """
                        
                        sh """
                            aws events put-targets \
                                --rule ${ruleName} \
                                --targets "Id"="1","Arn"="${functionArn}" \
                                --region ${env.REGION}
                        """
                        
                        sh """
                            aws lambda add-permission \
                                --function-name "${FUNCTION_NAME}" \
                                --statement-id "CloudWatch-${params.ENV}" \
                                --action "lambda:InvokeFunction" \
                                --principal "events.amazonaws.com" \
                                --source-arn "arn:aws:events:${env.REGION}:${env.AWS_ACCOUNT_ID}:rule/${ruleName}" \
                                --region ${env.REGION} || echo "Permission already exists"
                        """
                        
                        echo "CloudWatch schedule configured successfully"
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
            echo "Pipeline execution completed"
        }
        success {
            echo "Pipeline executed successfully"
        }
        failure {
            echo "Pipeline failed - check logs for details"
        }
    }
}
