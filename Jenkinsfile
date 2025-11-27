pipeline {
    agent any

    environment {
        SONARQUBE_ENV = "SonarQubeServer"

        CC_API_TOKEN = credentials('CC_API_TOKEN')
        CC_API_URL   = "https://sap-commerce-cloud-ccv2-mock.free.beeceptor.com/v2/subscriptions/111111111111"

        PLATFORM_ZIP  = "/opt/platform_zip/sap-commerce-suite-2211.zip"
        HYBRIS_DIR    = "core-customize/hybris"
        PLATFORM_DIR  = "core-customize/hybris/bin/platform"

        TEST_PACKAGES = "de.hybris.platform.spartacussampledata.*"
    }

    stages {

        stage('Checkout Source') {
            steps {
                echo "Checking out source code..."
                git branch: 'main', url: 'https://github.com/SAP-samples/cloud-commerce-sample-setup.git'
            }
        }

        stage('Prepare Platform') {
            steps {
                echo "Preparing SAP Commerce Platform..."

                    sh """
                        if [ ! -d "${HYBRIS_DIR}/bin/platform" ] || [ ! -d "${HYBRIS_DIR}/bin/modules" ]; then
                            echo "Platform missing → Unzipping platform..."
                            unzip -o -q ${PLATFORM_ZIP} 'hybris/*' -d ./core-customize
                        else
                            echo "Platform already exists. Skipping unzip."
                        fi
                    """
            }
        }

        stage('Build Platform') {
            steps {
                dir("${PLATFORM_DIR}") {
                    sh """
                        . ./setantenv.sh
                        ant clean all | tee ${env.WORKSPACE}/build.log
                    """
                }
            }
        }

        stage('Run SonarQube') {
            steps {
                script {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        dir("${PLATFORM_DIR}") {
                            sh """
                                . ./setantenv.sh
                                ant sonarcheck
                            """
                        }
                    }
                }
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo "Running unit tests only for custom extensions..."

                dir("${PLATFORM_DIR}") {
                    sh """
                        . ./setantenv.sh
                        ant yunitinit
                        ant alltests -Dtestclasses.packages="${TEST_PACKAGES}"
                    """
                }
            }
        }

        stage('Trigger Build - CCV2') {
            steps {
                script {
                    echo "Triggering CCV2 Build..."

                    sh """
                        curl -X POST ${CC_API_URL}/builds \
                             -o build_response.json
                    """

                    env.BUILD_ID = sh(
                        script: "grep -oP '(?<=\"code\": \")[^\"]*' build_response.json",
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Monitor Build - CCV2') {
            steps {
                script {
                    echo "Monitoring CCV2 build status..."

                    sh '''
                        TIMEOUT=600
                        INTERVAL=10
                        ELAPSED=0

                        while [ $ELAPSED -lt $TIMEOUT ]; do
                            STATUS=$(
                                curl -s ${CC_API_URL}/builds/${BUILD_ID}/progress \
                                -H "Authorization: Bearer ${CC_API_TOKEN}" \
                                | grep -oP '(?<="buildStatus": ")[^"]+'
                            )

                            echo "Build status: $STATUS"

                            if [ "$STATUS" = "SUCCESS" ]; then
                                echo "Build completed successfully!"
                                exit 0
                            fi

                            sleep $INTERVAL
                            ELAPSED=$((ELAPSED + INTERVAL))
                        done

                        echo "ERROR: Build timed out"
                        exit 1
                    '''
                }
            }
        }

        stage('Trigger Deployment - CCV2') {
            steps {
                script {
                    echo "Triggering CCV2 Deployment..."

                    sh """
                        curl -X POST ${CC_API_URL}/deployments \
                             -o deploy_response.json
                    """

                    env.DEPLOY_ID = sh(
                        script: "grep -oP '(?<=\"code\": \")[^\"]*' deploy_response.json",
                        returnStdout: true
                    ).trim()
                }
            }
        }
		
		stage('ChatGPT Analysis') {
			when { expression { true } }
			steps {
                script {
                    sh 'apt-get update && apt-get install -y make'
                    sh 'make build'
                }
            }
		}
    }

    post {
        success {
            echo "✔ SUCCESS — Build and Deployment completed!"
        }
        failure {
            echo "✘ FAILURE — See Jenkins logs."
             // Automatically explain errors when build fails
            explainError()
        }
    }
}
