pipeline {
    agent any

    environment {
        FUNCTION_NAME = 'hey-world-demo'
        REGION = 'us-east-1'
        ZIP_FILE = "lambda_package.zip"
        ARN_FILE = 'lambda_arn.txt'
        SONAR_SCANNER_HOME = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
    }

    parameters {
        string defaultValue: 'main', description: 'Enter git branch', name: 'GIT_BRANCH', trim: true
    }

    stages {
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

        stage('Prepare') {
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

        // ✅ New SonarQube Stage
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sq1') { // This must match your Jenkins SonarQube server name
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
                    credentialsId: 'lambda-function-aws-cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        echo "Creating Lambda function..."
                        aws lambda create-function \
                            --function-name $FUNCTION_NAME \
                            --runtime python3.13 \
                            --role arn:aws:iam::529088259986:role/service-role/s3_execRole \
                            --handler test.devtest.lambda_function.lambda_handler \
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
                    credentialsId: 'lambda-function-aws-cred',
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]) {
                    sh '''
                        LAMBDA_ARN=$(cat $ARN_FILE)

                        echo "Creating CloudWatch Events rule..."
                        aws events put-rule \
                            --schedule-expression "rate(15 minutes)" \
                            --name hello-demo-schedule \
                            --region $REGION

                        echo "Adding permission for CloudWatch to invoke Lambda..."
                        aws lambda add-permission \
                            --function-name $FUNCTION_NAME \
                            --statement-id hello-demo-event \
                            --action lambda:InvokeFunction \
                            --principal events.amazonaws.com \
                            --region $REGION || true

                        echo "Creating target for CloudWatch Events..."
                        aws events put-targets \
                            --rule hello-demo-schedule \
                            --targets "Id"="1","Arn"="$LAMBDA_ARN" \
                            --region $REGION
                    '''
                }
            }
        }
    }
}
