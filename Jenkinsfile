pipeline {
    agent any

    parameters {
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Enter the Git branch to build from')
        choice(name: 'ENV', choices: ['dev', 'stage', 'prod'], description: 'Choose the environment to deploy to')
    }

    environment {
        FUNCTION_NAME = "hello-world-lambda"
        REGION        = "us-east-1"

        // Map Jenkins credentials for AWS environments
        AWS_DEV    = "aws-cred-dev"
        AWS_STAGE  = "aws-cred-stage"
        AWS_PROD   = "aws-cred-prod"
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

        // -------------------------
        // stage('SonarQube Analysis') {
        //     environment {
        //         SONAR_TOKEN = credentials('SONAR_TOKEN')
        //     }
        //     steps {
        //         withSonarQubeEnv('SonarQube') {
        //             sh """
        //                 sonar-scanner \
        //                   -Dsonar.projectKey=${env.FUNCTION_NAME} \
        //                   -Dsonar.sources=. \
        //                   -Dsonar.host.url=\${SONAR_HOST_URL} \
        //                   -Dsonar.login=\${SONAR_TOKEN}
        //             """
        //         }
        //     }
        // }
        // -------------------------

        stage('Deploy Lambda') {
            steps {
                script {
                    def awsCred = ""
                    if (params.ENV == "dev") {
                        awsCred = env.AWS_DEV
                    } else if (params.ENV == "stage") {
                        awsCred = env.AWS_STAGE
                    } else if (params.ENV == "prod") {
                        awsCred = env.AWS_PROD
                    }

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh """
                            echo "Zipping Lambda function..."
                            zip -r function.zip lambda_function.py

                            echo "Deploying Lambda function..."
                            aws lambda update-function-code \
                              --function-name ${env.FUNCTION_NAME}-${params.ENV} \
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
                        awsCred = env.AWS_DEV
                    } else if (params.ENV == "stage") {
                        awsCred = env.AWS_STAGE
                    } else if (params.ENV == "prod") {
                        awsCred = env.AWS_PROD
                    }

                    withAWS(credentials: awsCred, region: env.REGION) {
                        sh """
                            echo "Creating/Updating CloudWatch Event Rule..."
                            aws events put-rule \
                              --name ${env.FUNCTION_NAME}-${params.ENV}-schedule \
                              --schedule-expression "rate(5 minutes)"

                            echo "Adding Lambda target to CloudWatch Rule..."
                            aws events put-targets \
                              --rule ${env.FUNCTION_NAME}-${params.ENV}-schedule \
                              --targets "Id"="1","Arn"="$(aws lambda get-function --function-name ${env.FUNCTION_NAME}-${params.ENV} --query 'Configuration.FunctionArn' --output text)"

                            echo "Adding permission for events to invoke Lambda..."
                            aws lambda add-permission \
                              --function-name ${env.FUNCTION_NAME}-${params.ENV} \
                              --statement-id ${env.FUNCTION_NAME}-${params.ENV}-invoke \
                              --action 'lambda:InvokeFunction' \
                              --principal events.amazonaws.com \
                              --source-arn \$(aws events describe-rule --name ${env.FUNCTION_NAME}-${params.ENV}-schedule --query 'Arn' --output text) || true
                        """
                    }
                }
            }
        }
    }
}
