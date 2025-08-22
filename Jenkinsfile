pipeline {
    agent any

    environment {
        FUNCTION_NAME = 'hello-demo'
        REGION = 'us-east-1'
        ZIP_FILE = "lambda_package.zip"
        ARN_FILE = 'lambda_arn.txt'
    }

    parameters {
        string defaultValue: 'main', description: 'Enter git branch', name: 'GIT_BRANCH', trim: true
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
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
      stage('Setup IAM Role') {
         steps {
          withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'lambda-function-aws-cred',
            accessKeyVariable: 'AWS_ACCESS_KEY_ID',
            secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
        ]])  {
            sh '''
                echo "Creating IAM role for Lambda if it doesn't exist..."
                aws iam get-role --role-name lambda_basic_execution || \
                aws iam create-role \
                    --role-name lambda_basic_execution \
                    --assume-role-policy-document file://trust-policy.json

                echo "Attaching policy..."
                aws iam attach-role-policy \
                    --role-name lambda_basic_execution \
                    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole || true

                echo "Fetching IAM Role ARN..."
                ROLE_ARN=$(aws iam get-role \
                    --role-name lambda_basic_execution \
                    --query 'Role.Arn' \
                    --output text)
                echo $ROLE_ARN > role_arn.txt
            '''
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
                echo "Reading IAM Role ARN..."
                ROLE_ARN=$(cat role_arn.txt)

                echo "Creating Lambda function..."
                aws lambda create-function \
                    --function-name $FUNCTION_NAME \
                    --runtime python3.13 \
                    --role $ROLE_ARN \
                    --handler test.devtest.lambda_function.lambda_handler \
                    --zip-file fileb://$ZIP_FILE \
                    --region $REGION || true

                # rest of your deployment logic remains unchanged...
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
