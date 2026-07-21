pipeline {
    agent {
        label 'salesforce'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 90, unit: 'MINUTES')
    }

    environment {
        SF_DISABLE_TELEMETRY = 'true'
        SF_USE_GENERIC_UNIX_KEYCHAIN = 'true'
        DEPLOY_MANIFEST = 'manifest/package.xml'
        RESULTS_DIR = 'test-results'
    }

    stages {
        stage('Checkout') {
            steps {
                deleteDir()
                checkout scm

                sh '''
                    set -eu
                    echo "Commit: ${GIT_COMMIT}"
                    echo "Branch: ${BRANCH_NAME}"
                    sf version
                '''
            }
        }

        stage('Determine Pipeline Action') {
            steps {
                script {
                    env.IS_PULL_REQUEST = env.CHANGE_ID ? 'true' : 'false'

                    if (env.CHANGE_TARGET == 'main') {
                        env.DEPLOY_ENVIRONMENT = 'production'
                        env.PIPELINE_ACTION = 'validate'
                    } else if (env.CHANGE_TARGET == 'develop') {
                        env.DEPLOY_ENVIRONMENT = 'integration'
                        env.PIPELINE_ACTION = 'validate'
                    } else if (env.BRANCH_NAME == 'main') {
                        env.DEPLOY_ENVIRONMENT = 'production'
                        env.PIPELINE_ACTION = 'deploy'
                    } else if (env.BRANCH_NAME == 'develop') {
                        env.DEPLOY_ENVIRONMENT = 'integration'
                        env.PIPELINE_ACTION = 'deploy'
                    } else if (env.BRANCH_NAME.startsWith('release/')) {
                        env.DEPLOY_ENVIRONMENT = 'preprod'
                        env.PIPELINE_ACTION = 'deploy'
                    } else if (env.BRANCH_NAME.startsWith('hotfix/')) {
                        env.DEPLOY_ENVIRONMENT = 'preprod'
                        env.PIPELINE_ACTION = 'validate'
                    } else {
                        env.DEPLOY_ENVIRONMENT = 'integration'
                        env.PIPELINE_ACTION = 'validate'
                    }

                    echo """
                    Salesforce environment: ${env.DEPLOY_ENVIRONMENT}
                    Pipeline action: ${env.PIPELINE_ACTION}
                    Pull request: ${env.IS_PULL_REQUEST}
                    """
                }
            }
        }

        stage('Repository Checks') {
            steps {
                sh '''
                    set -eu

                    test -f sfdx-project.json
                    test -f "${DEPLOY_MANIFEST}"

                    # Add project-specific checks here:
                    # npm ci
                    # npm run lint
                    # npm test
                    # sf scanner run --target force-app
                '''
            }
        }

        stage('Authenticate') {
            steps {
                script {
                    Map configuration = [
                        integration: [
                            usernameCredential: 'sf-int-username',
                            clientCredential:   'sf-int-client-id',
                            instanceUrl:         'https://ka-pc-consent--int.sandbox.my.salesforce.com/',
                            alias:               'devops-integration'
                        ],
                        preprod: [
                            usernameCredential: 'sf-staging-username',
                            clientCredential:   'sf-staging-client-id',
                            instanceUrl:         'https://ka-pc-consent--staging.sandbox.my.salesforce.com/',
                            alias:               'devops-preprod'
                        ],
                        production: [
                            usernameCredential: 'sf-prod-username',
                            clientCredential:   'sf-prod-client-id',
                            instanceUrl:         'https://ka-pc-consent.my.salesforce.com/',
                            alias:               'devops-production'
                        ]
                    ]

                    Map target = configuration[env.DEPLOY_ENVIRONMENT]

                    if (target == null) {
                        error("Unknown Salesforce environment: ${env.DEPLOY_ENVIRONMENT}")
                    }

                    env.SF_INSTANCE_URL = target.instanceUrl
                    env.SF_ORG_ALIAS = target.alias

                    withCredentials([
                        file(
                            credentialsId: 'sf-jwt-key',
                            variable: 'SF_JWT_KEY_FILE'
                        ),
                        string(
                            credentialsId: target.usernameCredential,
                            variable: 'SF_USERNAME'
                        ),
                        string(
                            credentialsId: target.clientCredential,
                            variable: 'SF_CLIENT_ID'
                        )
                    ]) {
                        sh '''
                            set +x
                            sf org login jwt \
                                --client-id "$SF_CLIENT_ID" \
                                --jwt-key-file "$SF_JWT_KEY_FILE" \
                                --username "$SF_USERNAME" \
                                --instance-url "$SF_INSTANCE_URL" \
                                --alias "$SF_ORG_ALIAS"
                        '''
                    }
                }
            }
        }

        stage('Validate') {
            when {
                expression {
                    env.PIPELINE_ACTION == 'validate'
                }
            }
            steps {
                script {
                    if (env.DEPLOY_ENVIRONMENT == 'production') {
                        sh '''
                            set -eu
                            mkdir -p "${RESULTS_DIR}"

                            sf project deploy validate \
                                --manifest "${DEPLOY_MANIFEST}" \
                                --target-org "${SF_ORG_ALIAS}" \
                                --test-level RunLocalTests \
                                --wait 90 \
                                --results-dir "${RESULTS_DIR}" \
                                --junit
                        '''
                    } else {
                        sh '''
                            set -eu
                            mkdir -p "${RESULTS_DIR}"

                            sf project deploy start \
                                --dry-run \
                                --manifest "${DEPLOY_MANIFEST}" \
                                --target-org "${SF_ORG_ALIAS}" \
                                --test-level RunLocalTests \
                                --wait 90 \
                                --results-dir "${RESULTS_DIR}" \
                                --junit
                        '''
                    }
                }
            }
        }

        stage('Production Approval') {
            when {
                allOf {
                    branch 'main'
                    expression {
                        env.PIPELINE_ACTION == 'deploy'
                    }
                }
            }
            steps {
                input(
                    message: 'Deploy this commit to DevOps Production?',
                    ok: 'Deploy',
                    submitter: 'devops-release-managers'
                )
            }
        }

        stage('Deploy') {
            when {
                expression {
                    env.PIPELINE_ACTION == 'deploy'
                }
            }
            steps {
                sh '''
                    set -eu
                    mkdir -p "${RESULTS_DIR}"

                    sf project deploy start \
                        --manifest "${DEPLOY_MANIFEST}" \
                        --target-org "${SF_ORG_ALIAS}" \
                        --test-level RunLocalTests \
                        --wait 90 \
                        --results-dir "${RESULTS_DIR}" \
                        --junit
                '''
            }
        }

        stage('Verify') {
            when {
                expression {
                    env.PIPELINE_ACTION == 'deploy'
                }
            }
            steps {
                sh '''
                    set -eu
                    sf org display \
                        --target-org "${SF_ORG_ALIAS}" \
                        --verbose
                '''
            }
        }
    }

    post {
        always {
            junit(
                testResults: 'test-results/**/*.xml',
                allowEmptyResults: true
            )

            archiveArtifacts(
                artifacts: 'test-results/**/*',
                allowEmptyArchive: true,
                fingerprint: true
            )

            sh '''
                set +e
                sf org logout \
                    --target-org "${SF_ORG_ALIAS:-}" \
                    --no-prompt
                rm -rf .sf
            '''

            deleteDir()
        }

        success {
            echo "Salesforce pipeline completed successfully."
        }

        failure {
            echo "Salesforce pipeline failed. Review the deployment and Apex test results."
        }
    }
}