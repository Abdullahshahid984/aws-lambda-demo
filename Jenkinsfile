pipeline {
    agent any

    environment {
        AWS_REGION = credentials('AWS_REGION')
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        LAMBDA_ARN = credentials('LAMBDA_ARN')
        EXEC_ROLE_ARN = credentials('EXEC_ROLE_ARN')
        SCHEDULE_EXPR = credentials('SCHEDULE_EXPR')
    }

    stages {
        stage('Update Lambda Function') {
            steps {
                sh "aws lambda update-function-code --function-name ${LAMBDA_ARN} --zip-file fileb://function.zip --region ${AWS_REGION}"
            }
        }

        stage('Update Lambda Configuration') {
            steps {
                sh "aws lambda update-function-configuration --function-name ${LAMBDA_ARN} --role ${EXEC_ROLE_ARN} --region ${AWS_REGION}"
            }
        }

        stage('Update CloudWatch Event Rule') {
            steps {
                sh "aws events put-rule --name lambda-schedule-rule --schedule-expression ${SCHEDULE_EXPR} --region ${AWS_REGION}"
            }
        }

        stage('Add Lambda Permission') {
            steps {
                sh "aws lambda add-permission --function-name ${LAMBDA_ARN} --statement-id lambda-schedule-permission --action 'lambda:InvokeFunction' --principal events.amazonaws.com --source-arn arn:aws:events:${AWS_REGION}:${AWS_ACCOUNT_ID}:rule/lambda-schedule-rule --region ${AWS_REGION}"
            }
        }

        stage('Add Lambda Target') {
            steps {
                sh "aws events put-targets --rule lambda-schedule-rule --targets Id=1,Arn=${LAMBDA_ARN} --region ${AWS_REGION}"
            }
        }
    }
}
