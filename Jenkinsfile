pipeline {
    agent any

    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'preprod', 'prod'], description: 'Deployment environment')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Git branch to deploy')
    }

    environment {
        REGION = 'us-east-1'
        ZIP_FILE = 'lambda_package.zip'
        ARN_FILE = 'lambda_arn.txt'
        SONAR_SCANNER_HOME = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    }

    stages {
        stage('Set Environment Variables') {
            steps {
                script {
                    // Set per-environment configuration
                    def envConfig = [
                        dev: [
                            FUNCTION_NAME: 'hey-world-demo-dev',
                            HANDLER: 'test.devtest.lambda_function.lambda_handler',
                            SCHEDULE: 'rate(15 minutes)',
                            ROLE_ARN: 'arn:aws:iam::529088259986:role/service-role/s3_execRole',
                            AWS_CRED_ID: 'lambda-function-aws-cred-dev',
                            EVENT_RULE_NAME: 'hello-demo-schedule-dev',
                            SECRET_NAME: 'my-secret-dev'
                        ],
                        preprod: [
                            FUNCTION_NAME: 'hey-world-demo-preprod',
                            HANDLER: 'test.devtest.lambda_function.lambda_handler',
                            SCHEDULE: 'rate(10 minutes)',
                            ROLE_ARN: 'arn:aws:iam::529088259986:role/service-role/s3_execRole',
                            AWS_CRED_ID: 'lambda-function-aws-cred-preprod',
                            EVENT_RULE_NAME: 'hello-demo-schedule-preprod',
                            SECRET_NAME: 'my-secret-preprod'
                        ],
                        prod: [
                            FUNCTION_NAME: 'hey-world-demo-prod',
                            HANDLER: 'test.devtest.lambda_function.lambda_handler',
                            SCHEDULE: 'rate(5 minutes)',
                            ROLE_ARN: 'arn:aws:iam::529088259986:role/service-role/s3_execRole',
                            AWS_CRED_ID: 'lambda-function-aws-cred-prod',
                            EVENT_RULE_NAME: 'hello-demo-schedule-prod',
                            SECRET_NAME: 'my-secret-prod'
                        ]
                    ]

                    def selected = envConfig[params.ENVIRONMENT]
                    env.FUNCTION_NAME = selected.FUNCTION_NAME
                    env.HANDLER = selected.HANDLER
                    env.SCHEDULE = selected.SCHEDULE
                    env.ROLE_ARN = selected.ROLE_ARN
                    env.AWS_CRED_ID = selected.AWS_CRED_ID
                    env.EVENT_RULE_NAME = selected.EVENT_RULE_NAME
                    env.SECRET_NAME = selected.SECRET_NAME
                }
            }
        }

        stage('Clone Repository') {
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: "*/${params.GIT_BRANCH}"]],
                    userRemoteConfigs: [[
                        url: 'https://github.com/AngelsTech/aws-lambda-demo.git',
                        credentialsId: '66142c87-d271-41f4-82d1-cbbee8e844d0'
                    ]]
                ])
            }
        }

        stage('Prepare Package') {
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

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sq1') {
                    withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'TOKEN')]) {
                        sh '''
                            echo "Running SonarQube Scan..."
                            ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=aws-lambda-demo \
                            -Dsonar.sources=test \
                            -Dsonar.login=$TOKEN
                        '''
                    }
                }
            }
        }

        stage('Deploy Lambda') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${env.AWS_CRED_ID}",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        echo "Creating Lambda function (if not exists)..."
                        aws lambda create-function \
                            --function-name $FUNCTION_NAME \
                            --runtime python3.13 \
                            --role $ROLE_ARN \
                            --handler $HANDLER \
                            --zip-file fileb://$ZIP_FILE \
                            --region $REGION || true

                        echo "Waiting for Lambda function to become Active..."
                        while true; do
                            STATUS=$(aws lambda get-function-configuration \
                                --function-name $FUNCTION_NAME \
                                --region $REGION \
                                --query 'State' --output text)
                            echo "Current status: $STATUS"
                            if [ "$STATUS" = "Active" ]; then
                                break
                            fi
                            sleep 5
                        done

                        echo "Updating function code..."
                        aws lambda update-function-code \
                            --function-name $FUNCTION_NAME \
                            --zip-file fileb://$ZIP_FILE \
                            --region $REGION

                        echo "Getting Lambda ARN..."
                        aws lambda get-function \
                            --function-name $FUNCTION_NAME \
                            --region $REGION \
                            --query 'Configuration.FunctionArn' \
                            --output text > $ARN_FILE
                    '''
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                withCredentials([[ 
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "${env.AWS_CRED_ID}",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        LAMBDA_ARN=$(cat $ARN_FILE)

                        echo "Creating CloudWatch Events rule..."
                        aws events put-rule \
                            --schedule-expression "$SCHEDULE" \
                            --name "$EVENT_RULE_NAME" \
                            --region $REGION

                        echo "Adding permission for CloudWatch to invoke Lambda..."
                        aws lambda add-permission \
                            --function-name $FUNCTION_NAME \
                            --statement-id "$EVENT_RULE_NAME-stmt" \
                            --action lambda:InvokeFunction \
                            --principal events.amazonaws.com \
                            --region $REGION || true

                        echo "Creating target for CloudWatch Events..."
                        aws events put-targets \
                            --rule "$EVENT_RULE_NAME" \
                            --targets "Id"="1","Arn"="$LAMBDA_ARN" \
                            --region $REGION
                    '''
                }
            }
        }
    }
}
