pipeline {
    agent any

    parameters {
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose target AWS environment')
    }

    environment {
        FUNCTION_NAME = "hello-world-lambda"
        REGION = "us-east-1"
        SONAR_HOST_URL = "https://sonarcloud.io"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Sonar') {
            environment {
                SONAR_TOKEN = credentials('SONAR_TOKEN')
            }
            steps {
                withSonarQubeEnv('SonarQubeServer') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=hello-world-lambda \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Deploy Lambda') {
            steps {
                script {
                    def awsCred = (params.ENV == 'dev') ? 'aws-cred-dev' :
                                  (params.ENV == 'stage') ? 'aws-cred-stage' :
                                  'aws-cred-prod'

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh '''
                            zip function.zip lambda_function.py
                            aws lambda update-function-code \
                                --function-name ''' + "${FUNCTION_NAME}-${params.ENV}" + ''' \
                                --zip-file fileb://function.zip \
                                --region ''' + "${REGION}" + '''
                        '''
                    }
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                script {
                    def awsCred = (params.ENV == 'dev') ? 'aws-cred-dev' :
                                  (params.ENV == 'stage') ? 'aws-cred-stage' :
                                  'aws-cred-prod'

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh '''
                            aws events put-rule \
                                --name ''' + "${FUNCTION_NAME}-schedule-${params.ENV}" + ''' \
                                --schedule-expression "rate(5 minutes)" \
                                --region ''' + "${REGION}" + '''

                            aws lambda add-permission \
                                --function-name ''' + "${FUNCTION_NAME}-${params.ENV}" + ''' \
                                --statement-id ''' + "${FUNCTION_NAME}-event-${params.ENV}" + ''' \
                                --action 'lambda:InvokeFunction' \
                                --principal events.amazonaws.com \
                                --source-arn arn:aws:events:''' + "${REGION}" + ''':$(aws sts get-caller-identity --query Account --output text):rule/''' + "${FUNCTION_NAME}-schedule-${params.ENV}" + '''

                            aws events put-targets \
                                --rule ''' + "${FUNCTION_NAME}-schedule-${params.ENV}" + ''' \
                                --targets "Id"="1","Arn"="$(aws lambda get-function --function-name ''' + "${FUNCTION_NAME}-${params.ENV}" + ''' --query 'Configuration.FunctionArn' --output text)"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed for ENV=${params.ENV}"
        }
    }
}
