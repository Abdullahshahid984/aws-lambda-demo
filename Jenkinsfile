pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter git branch', trim: true)
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
    }

    environment {
        FUNCTION_NAME = 'hey-world-demo'
        REGION = 'us-east-1'
        ZIP_FILE = 'lambda_package.zip'
        ARN_FILE = 'lambda_arn.txt'
        SONAR_SCANNER_HOME = tool name: 'SonarScanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
        AWS_CRED_ID = ''
        EXEC_ROLE_ARN_CRED_ID = ''
        SCHEDULE_EXPR = ''
        RULE_NAME = ''
        STATEMENT_ID = ''
    }

    stages {
        stage('Set Environment Config') {
            steps {
                script {
                    if (params.ENV == 'dev') {
                        env.AWS_CRED_ID = 'aws-cred-dev'
                        env.EXEC_ROLE_ARN_CRED_ID = 'exec-role-arn-dev'
                        env.SCHEDULE_EXPR = 'rate(15 minutes)'
                    } else if (params.ENV == 'stage') {
                        env.AWS_CRED_ID = 'aws-cred-stage'
                        env.EXEC_ROLE_ARN_CRED_ID = 'exec-role-arn-stage'
                        env.SCHEDULE_EXPR = 'rate(30 minutes)'
                    } else if (params.ENV == 'prod') {
                        env.AWS_CRED_ID = 'aws-cred-prod'
                        env.EXEC_ROLE_ARN_CRED_ID = 'exec-role-arn-prod'
                        env.SCHEDULE_EXPR = 'rate(5 minutes)'
                    }
                    env.RULE_NAME = "${env.FUNCTION_NAME}-${params.ENV}-schedule"
                    env.STATEMENT_ID = "${env.FUNCTION_NAME}-${params.ENV}-event"
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

        stage('Prepare') {
            steps {
                sh '''
                    echo "Zipping files"
                    zip -r $ZIP_FILE test -x "**/.git/*" -x "**/README.md" -x "**/Jenkinsfile"
                    echo "Listing zip"
                    unzip -l $ZIP_FILE
                '''
            }
        }

        stage('Deploy Lambda') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${env.AWS_CRED_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'],
                    [string(credentialsId: "${env.EXEC_ROLE_ARN_CRED_ID}", variable: 'EXEC_ROLE_ARN')]
                ]) {
                    sh '''
                        if aws lambda get-function --function-name $FUNCTION_NAME --region $REGION > /dev/null 2>&1; then
                            echo "Updating existing Lambda..."
                            aws lambda update-function-code --function-name $FUNCTION_NAME --zip-file fileb://$ZIP_FILE --region $REGION
                        else
                            echo "Creating new Lambda..."
                            aws lambda create-function --function-name $FUNCTION_NAME --runtime python3.13 --role $EXEC_ROLE_ARN --handler test.devtest.lambda_function.lambda_handler --zip-file fileb://$ZIP_FILE --region $REGION
                        fi
                        aws lambda get-function --function-name $FUNCTION_NAME --region $REGION --query 'Configuration.FunctionArn' --output text > $ARN_FILE
                    '''
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding', credentialsId: "${env.AWS_CRED_ID}", accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']

                ]) {
                    sh '''
                        LAMBDA_ARN=$(cat $ARN_FILE)
                        echo "Creating/Updating CloudWatch Event Rule..."
                        aws events put-rule --schedule-expression "$SCHEDULE_EXPR" --name "$RULE_NAME" --region $REGION
                        echo "Checking Lambda permissions..."
                        if ! aws lambda get-policy --function-name $FUNCTION_NAME --region $REGION | grep -q "$STATEMENT_ID"; then
                            echo "Adding permission for CloudWatch Events to invoke Lambda..."
                            aws lambda add-permission --function-name $FUNCTION_NAME --statement-id $STATEMENT_ID --action lambda:InvokeFunction --principal events.amazonaws.com --region $REGION
                        fi
                        echo "Adding Lambda as CloudWatch target..."
                        aws events put-targets --rule "$RULE_NAME" --targets Id=1,Arn="$LAMBDA_ARN" --region $REGION
                    '''
                }
            }
        }
    }
}
