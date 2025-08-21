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
                    // Map of env-specific config
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

                    // Pick config for selected environment
                    def selected = envConfig[params.ENVIRONMENT]

                    // Export into pipeline env
                    selected.each { key, value -> env[key] = value }
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
                    echo "Zipping Lambda package..."
                    zip -r $ZIP_FILE test \
                        -x "**/.git/*" \
                        -x "**/README.md" \
                        -x "**/Jenkinsfile"

                    echo "Zip file contents:"
                    unzip -l $ZIP_FILE
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sq1') {
                    withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'TOKEN')]) {
                        sh '''
                            echo "Running SonarQube analysis..."
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
                        echo "Checking if Lambda exists..."
                        if ! aws lambda get-function --function-name $FUNCTION_NAME --region $REGION >/dev/null 2>&1; then
                            echo "Creating new Lambda..."
                            aws lambda create-function \
                                --function-name $FUNCTION_NAME \
                                --runtime python3.13 \
                                --role $ROLE_ARN \
                                --handler $HANDLER \
                                --zip-file fileb://$ZIP_FILE \
                                --region $REGION
                        fi

                        echo "Waiting until Lambda is Active..."
                        while true; do
                            STATUS=$(aws lambda get-function-configuration \
                                --function-name $FUNCTION_NAME \
                                --region $REGION \
                                --query 'State' --output text)
                            echo "Status: $STATUS"
                            if [ "$STATUS" = "Active" ]; then break; fi
                            sleep 5
                        done

                        echo "Updating Lambda code..."
                        aws lambda update-function-code \
                            --function-name $FUNCTION_NAME \
                            --zip-file fileb://$ZIP_FILE \
                            --region $REGION

                        echo "Setting Lambda environment variables..."
                        aws lambda update-function-configuration \
                            --function-name $FUNCTION_NAME \
                            --environment "Variables={SECRET_NAME=$SECRET_NAME}" \
                            --region $REGION

                        echo "Saving Lambda ARN..."
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

                        echo "Creating/Updating CloudWatch rule..."
                        aws events put-rule \
                            --schedule-expression "$SCHEDULE" \
                            --name "$EVENT_RULE_NAME" \
                            --region $REGION

                        echo "Cleaning old Lambda permissions..."
                        aws lambda remove-permission \
                            --function-name $FUNCTION_NAME \
                            --statement-id "$EVENT_RULE_NAME-stmt" \
                            --region $REGION || true

                        echo "Adding permission for CloudWatch..."
                        aws lambda add-permission \
                            --function-name $FUNCTION_NAME \
                            --statement-id "$EVENT_RULE_NAME-stmt" \
                            --action lambda:InvokeFunction \
                            --principal events.amazonaws.com \
                            --region $REGION

                        echo "Attaching Lambda target to CloudWatch rule..."
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
