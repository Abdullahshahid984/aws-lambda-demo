pipeline {
    agent any

    parameters {
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose deployment environment')
    }

    environment {
        FUNCTION_NAME = 'hello-world'
        REGION = 'us-east-1'
    }

    stages {
        // stage('Sonar') {
        //     steps {
        //         withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
        //             sh '''
        //                 sonar-scanner \
        //                   -Dsonar.projectKey=${FUNCTION_NAME}-${ENV} \
        //                   -Dsonar.sources=. \
        //                   -Dsonar.host.url=https://sonarcloud.io \
        //                   -Dsonar.login=$SONAR_TOKEN
        //             '''
        //         }
        //     }
        // }

        stage('Deploy Lambda') {
            steps {
                script {
                    def awsCred = ""
                    if (params.ENV == "dev") {
                        awsCred = "aws-cred-dev"
                    } else if (params.ENV == "stage") {
                        awsCred = "aws-cred-stage"
                    } else if (params.ENV == "prod") {
                        awsCred = "aws-cred-prod"
                    }

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh """
                            zip function.zip index.js
                            aws lambda create-function --function-name ${FUNCTION_NAME}-${ENV} \
                                --runtime nodejs18.x \
                                --role arn:aws:iam::123456789012:role/lambda-ex \
                                --handler index.handler \
                                --zip-file fileb://function.zip || \
                            aws lambda update-function-code --function-name ${FUNCTION_NAME}-${ENV} \
                                --zip-file fileb://function.zip
                        """
                    }
                }
            }
        }

        stage('Setup CloudWatch Schedule') {
            steps {
                script {
                    def awsCred = ""
                    if (params.ENV == "dev") {
                        awsCred = "aws-cred-dev"
                    } else if (params.ENV == "stage") {
                        awsCred = "aws-cred-stage"
                    } else if (params.ENV == "prod") {
                        awsCred = "aws-cred-prod"
                    }

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh """
                            aws events put-rule --name ${FUNCTION_NAME}-${ENV}-schedule \
                                --schedule-expression "rate(5 minutes)" || true
                            aws lambda add-permission --function-name ${FUNCTION_NAME}-${ENV} \
                                --statement-id ${FUNCTION_NAME}-${ENV}-schedule \
                                --action 'lambda:InvokeFunction' \
                                --principal events.amazonaws.com \
                                --source-arn arn:aws:events:${REGION}:123456789012:rule/${FUNCTION_NAME}-${ENV}-schedule || true
                            aws events put-targets --rule ${FUNCTION_NAME}-${ENV}-schedule \
                                --targets "Id"="1","Arn"="$(aws lambda get-function --function-name ${FUNCTION_NAME}-${ENV} --query 'Configuration.FunctionArn' --output text)"
                        """
                    }
                }
            }
        }
    }
}
