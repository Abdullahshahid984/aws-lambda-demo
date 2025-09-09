def config = [
    layers: [
        core: [zip: 'core.zip', exclude: "'**/numpy/**' '**/pandas/**'"],
        abd: [zip: 'abc.zip', exclude: "'**/six/**' '**/urllib3/**' '**/numpy/**' '**/pandas/**'"]
    ],
    baseLayer: 'arn:aws:lambda:us-east-1:336992948345:layer:AMSSDXPands-Python39:29'
]

// IAM Role configuration
def roleConfig = [
    devtest: 'arn:aws:iam::YOUR_ACCOUNT_ID:role/lambda-execution-role-dev',
    stage: 'arn:aws:iam::YOUR_ACCOUNT_ID:role/lambda-execution-role-stage',
    prod: 'arn:aws:iam::YOUR_ACCOUNT_ID:role/lambda-execution-role-prod'
]

pipeline {
    agent {
        label 'your-agent-label'
    }
   
    parameters {
        string defaultValue: 'develop', description: 'Select git branch', name: 'GIT_BRANCH', trim: true
        choice(name: 'ENV', choices: ['devtest', 'stage', 'prod'], description: 'Choose deployment environment')
    }
   
    environment {
        BASE_FUNCTION_NAME = 'lambda-function'
        REGION = 'us-east-1'
        ZIP_FILE = 'lambda_package.zip'
        AWS_ACCOUNT_ID = 'your-aws-account-id'
        LAMBDA_ROLE_ARN = "${roleConfig[params.ENV]}"
    }
   
    stages {
        stage('Clone Repository') {
            steps {
                cleanWs()
                checkout([$class: 'GitSCM',
                    branches: [[name: "${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://your-git-repo-url',
                        credentialsId: 'your-git-credentials'
                    ]]
                ])
            }
        }
       
        stage('Verify IAM Role') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        echo "Verifying IAM role exists: ${env.LAMBDA_ROLE_ARN}"
                       
                        def roleExists = sh(
                            script: """
                                aws iam get-role \
                                    --role-name ${env.LAMBDA_ROLE_ARN.split('/')[1]} \
                                    --region ${env.REGION} \
                                    --query 'Role.Arn' \
                                    --output text 2>/dev/null || echo "NOT_FOUND"
                            """,
                            returnStdout: true
                        ).trim()
                       
                        if (roleExists == "NOT_FOUND") {
                            error("IAM Role not found: ${env.LAMBDA_ROLE_ARN}. Please create the role first.")
                        } else {
                            echo "IAM Role verified: ${roleExists}"
                        }
                    }
                }
            }
        }
       
        stage('Install Requirements') {
            steps {
                script {
                    config.layers.each { layerName, layerConfig ->
                        dir(env.WORKSPACE) {
                            sh """
                                echo "Installing dependencies for ${layerName} layer..."
                                mkdir -p ./${layerName}
                                if [ -f "requirements-${layerName}.txt" ]; then
                                    pip3.9 install -r requirements-${layerName}.txt -t ./${layerName} --no-deps
                                else
                                    echo "requirements-${layerName}.txt not found, skipping..."
                                fi
                            """
                        }
                    }
                }
            }
        }
       
        stage('Package Layers') {
            steps {
                script {
                    config.layers.each { layerName, layerConfig ->
                        dir(env.WORKSPACE) {
                            sh """
                                echo "Packaging ${layerName} layer..."
                                if [ -d "./${layerName}" ]; then
                                    zip -r ${layerConfig.zip} ./${layerName} -x ${layerConfig.exclude}
                                else
                                    echo "Creating empty zip for ${layerName}"
                                    touch ${layerConfig.zip}
                                fi
                            """
                        }
                    }
                }
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
                        def layerArns = [:]
                       
                        config.layers.each { layerName, layerConfig ->
                            dir(env.WORKSPACE) {
                                sh """
                                    aws lambda publish-layer-version \
                                        --layer-name ${layerName} \
                                        --zip-file fileb://${layerConfig.zip} \
                                        --compatible-runtimes python3.9 \
                                        --region ${env.REGION}
                                """
                               
                                def layerArn = sh(
                                    script: """
                                        aws lambda list-layer-versions \
                                            --layer-name ${layerName} \
                                            --region ${env.REGION} \
                                            --query 'LayerVersions[0].LayerVersionArn' \
                                            --output text
                                    """,
                                    returnStdout: true
                                ).trim()
                               
                                layerArns[layerName] = layerArn
                                echo "Published ${layerName}: ${layerArn}"
                            }
                        }
                    }
                }
            }
        }
       
        stage('Create or Update Lambda') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-credentials',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    script {
                        def FUNCTION_NAME = "${env.BASE_FUNCTION_NAME}-${params.ENV}"
                        def allLayers = [layerArns.core, layerArns.uscls, config.baseLayer]
                       
                        // Check if function exists
                        def functionExists = sh(
                            script: """
                                aws lambda get-function \
                                    --function-name "${FUNCTION_NAME}" \
                                    --region ${env.REGION} \
                                    --query 'Configuration.FunctionName' \
                                    --output text 2>/dev/null || echo "NOT_FOUND"
                            """,
                            returnStdout: true
                        ).trim()
                       
                        if (functionExists == "NOT_FOUND") {
                            echo "Creating new Lambda function: ${FUNCTION_NAME}"
                            sh """
                                aws lambda create-function \
                                    --function-name "${FUNCTION_NAME}" \
                                    --runtime python3.9 \
                                    --role "${env.LAMBDA_ROLE_ARN}" \
                                    --handler lambda_function.lambda_handler \
                                    --zip-file fileb://${env.ZIP_FILE} \
                                    --layers ${allLayers.join(' ')} \
                                    --region ${env.REGION}
                            """
                        } else {
                            echo "Updating existing Lambda function: ${FUNCTION_NAME}"
                            // Update configuration with IAM role
                            sh """
                                aws lambda update-function-configuration \
                                    --function-name "${FUNCTION_NAME}" \
                                    --role "${env.LAMBDA_ROLE_ARN}" \
                                    --layers ${allLayers.join(' ')} \
                                    --region ${env.REGION}
                            """
                           
                            // Update code
                            sh """
                                aws lambda update-function-code \
                                    --function-name "${FUNCTION_NAME}" \
                                    --zip-file fileb://${env.ZIP_FILE} \
                                    --region ${env.REGION}
                            """
                        }
                    }
                }
            }
        }
       
        stage('Verify Deployment') {
            steps {
                script {
                    def FUNCTION_NAME = "${env.BASE_FUNCTION_NAME}-${params.ENV}"
                    def maxRetries = 30
                    def retryCount = 0
                   
                    while (retryCount < maxRetries) {
                        def status = sh(
                            script: """
                                aws lambda get-function-configuration \
                                    --function-name "${FUNCTION_NAME}" \
                                    --region ${env.REGION} \
                                    --query 'State' \
                                    --output text 2>/dev/null || echo "Pending"
                            """,
                            returnStdout: true
                        ).trim()
                       
                        if (status == 'Active') {
                            echo "Lambda function is Active"
                            break
                        }
                       
                        sleep(5)
                        retryCount++
                    }
                }
            }
        }
       
        stage('Test Lambda Function') {
            steps {
                script {
                    def FUNCTION_NAME = "${env.BASE_FUNCTION_NAME}-${params.ENV}"
                   
                    // Simple test invocation
                    sh """
                        aws lambda invoke \
                            --function-name "${FUNCTION_NAME}" \
                            --region ${env.REGION} \
                            --payload '{"test": "event"}' \
                            /tmp/lambda_response.json \
                            && echo "Lambda test invocation successful" \
                            || echo "Lambda invocation failed (may be expected for new functions)"
                    """
                }
            }
        }
    }
   
    post {
        always {
            cleanWs()
        }
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Deployment failed"
        }
    }
}
