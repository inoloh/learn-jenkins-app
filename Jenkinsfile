pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = credentials('site_id')
        NETLIFY_AUTH_TOKEN = credentials('netlify_pat')
    }

    stages {
        stage('Build') {
        agent {
            docker {
                image 'node:18-alpine'
                reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Run tests') {
            parallel {
                stage('Unit') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                            }
                        }
                    steps {
                        sh '''
                            test -f build/index.html
                            npm test

                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                            }
                        }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            sleep 10
                            npx playwright test --reporter=html
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'E2E local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        // stage('Deploy staging') {
        //     agent {
        //         docker {
        //             image 'node:18-alpine'
        //             reuseNode true
        //             }
        //         }
        //     steps {
        //         sh '''
        //             npm install netlify-cli node-jq
        //             node_modules/.bin/netlify --version
        //             node_modules/.bin/netlify status
        //             node_modules/.bin/netlify deploy --dir=build --json > staging-deploy.txt
        //             node_modules/.bin/node-jq -r .deploy_url staging-deploy.txt
        //         '''

        //         script {
        //             env.STAGING_URL = sh(script: "node_modules/.bin/node-jq -r '.deploy_url' staging-deploy.txt", returnStdout: true)
        //         }
        //     }
        // }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                    }
                }
            environment {
                CI_ENVIRONMENT_URL = "STAGING_URL_TO_BE_SET"
            }
            steps {
                sh '''
                    npm install netlify-cli node-jq
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --json > staging-deploy.json
                    node_modules/.bin/node-jq -r .deploy_url staging-deploy.json
                    CI_ENVIRONMENT_URL=$(node_modules/.bin/node-jq -r '.deploy_url' staging-deploy.json)
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'stage E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    input message: 'Are you ready to rumble?', ok: 'YES SIR'
                }
            }
        }

        stage('Deploy Prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                    }
                }
            environment {
                CI_ENVIRONMENT_URL = credentials('netlify_site_url')
            }
            steps {
                sh '''
                    node --version
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                    npx playwright test --reporter=html
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
