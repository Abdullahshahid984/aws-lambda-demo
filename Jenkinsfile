pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter git branch to deploy', trim: true)
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
    }

    environment {
        BASE_FUNCTION_NAME = 'hey-world-demo'
        REGION = 'us-east-1'
        SONAR_PROJECT_KEY = "aws-lambda"
        SONAR_ORG = "your-org"
        ZIP_FILE = "lambda_package.zip"
    }

    stages {

        stage('Checkout') {
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

        stage('Install Dependencies') {
            steps {
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

        // Commented Sonar Stage
        /*
        stage('SonarQube Analysis') {
            environment {
                SONAR_TOKEN = credentials('SONAR_TOKEN')
            }
            steps {
                sh '''
                  sonar-scanner \
                    -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                    -Dsonar.organization=${SONAR_ORG} \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.login=${SONAR_TOKEN}
                '''
            }
        }
        */

        stage('Deploy to AWS Lambda') {
            steps {
                script {
                    def awsCredId = ''
                    def lambdaRole = ''
                    def functionName = "${env.BASE_FUNCTION_NAME}-${params.ENV}"

                    if (params.ENV == 'dev') {
                        awsCredId = 'aws-cred-dev'
                        lambdaRole = 'arn:aws:iam::529088259986:role/lambda_exec_role_dev'
                    } else if (params.ENV == 'stage') {
                        awsCredId = 'aws-cred-stage'
                        lambdaRole = 'arn:aws:iam::529088259986:role/lambda_exec_role_stage'
                    } else if (params.ENV == 'prod') {
                        awsCredId = 'aws-cred-prod'
                        lambdaRole = 'arn:aws:iam::529088259986:role/lambda_exec_role_prod'
                    }

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredId]]) {
                        sh '''
                          export AWS_DEFAULT_REGION=$REGION
                          FUNCTION_NAME="${BASE_FUNCTION_NAME}-${ENV}"

                          # Check if function exists
                          if ! aws lambda get-function --function-name $FUNCTION_NAME > /dev/null 2>&1; then
                            echo "Creating new Lambda function: $FUNCTION_NAME"
                            aws lambda create-function \
                              --function-name $FUNCTION_NAME \
                              --runtime python3.13 \
                              --role ''' + lambdaRole + ''' \
                              --handler hello-world.lambda_function.lambda_handler \
                              --zip-file fileb://$ZIP_FILE

                            echo "Waiting for Lambda to become active..."
                            aws lambda wait function-active --function-name $FUNCTION_NAME
                          fi

                          # Update Lambda code (safe after wait)
                          aws lambda update-function-code \
                            --function-name $FUNCTION_NAME \
                            --zip-file fileb://$ZIP_FILE
                        '''
                    }
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                script {
                    def awsCredId = ''

                    if (params.ENV == 'dev') {
                        awsCredId = 'aws-cred-dev'
                    } else if (params.ENV == 'stage') {
                        awsCredId = 'aws-cred-stage'
                    } else if (params.ENV == 'prod') {
                        awsCredId = 'aws-cred-prod'
                    }

                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: awsCredId]]) {
                        sh '''
                        export AWS_DEFAULT_REGION=$REGION
                        FUNCTION_NAME="${BASE_FUNCTION_NAME}-${ENV}"
                        RULE_NAME="hello-world-schedule-${ENV}"
                        SCHEDULE="rate(5 minutes)"
                        STATEMENT_ID="cw-invoke-${ENV}"

                        # Create or update CloudWatch rule
                        aws events put-rule \
                            --name $RULE_NAME \
                            --schedule-expression "$SCHEDULE" \
                            --state ENABLED

                        # Resolve Lambda ARN
                        FUNCTION_ARN=$(aws lambda get-function --function-name $FUNCTION_NAME --query 'Configuration.FunctionArn' --output text)

                        # Add Lambda as target of the rule
                        aws events put-targets \
                            --rule $RULE_NAME \
                            --targets "Id"="1","Arn"=$FUNCTION_ARN

                        # Resolve Rule ARN
                        SOURCE_ARN=$(aws events describe-rule --name $RULE_NAME --query 'Arn' --output text)

                        # Check if permission already exists
                        if ! aws lambda get-policy --function-name $FUNCTION_NAME | grep -q $STATEMENT_ID; then
                            echo "Adding new permission: $STATEMENT_ID"
                            aws lambda add-permission \
                                --function-name $FUNCTION_NAME \
                                --statement-id $STATEMENT_ID \
                                --action 'lambda:InvokeFunction' \
                                --principal events.amazonaws.com \
                                --source-arn $SOURCE_ARN
                        else
                            echo "Permission $STATEMENT_ID already exists, skipping..."
                        fi
                        '''
                    }
                }
            }
        }

    }
}
