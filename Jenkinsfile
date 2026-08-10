```groovy
pipeline {

    agent {
        docker {
            // Maven 3.9.6 with JDK 17
            image 'maven:3.9.6-eclipse-temurin-17'

            // Run container as root and use host Maven cache
            args '-u root:root -v $HOME/.m2:/root/.m2'

            reuseNode true
        }
    }

    environment {
        // Maven configuration
        MAVEN_OPTS = '-Dmaven.repo.local=/root/.m2/repository'
        MVN_CMD = 'mvn -B -ntp'

        // Application configuration
        APP_NAME = 'my-app'
        DOCKER_REPO = 'myregistry/my-app'
    }

    parameters {

        string(
            name: 'GIT_BRANCH',
            defaultValue: 'master',
            description: 'Branch to build'
        )

        booleanParam(
            name: 'DEPLOY_PROD',
            defaultValue: false,
            description: 'Deploy to production?'
        )

        choice(
            name: 'LOG_LEVEL',
            choices: ['INFO', 'DEBUG', 'WARN'],
            description: 'Logging level'
        )
    }

    stages {

        /*
         * DO NOT add a manual git checkout here.
         *
         * Jenkins Declarative Pipeline automatically performs:
         * "Declarative: Checkout SCM"
         *
         * It will use the repository configured in your Jenkins job.
         */

        stage('Build + Test') {

            steps {

                sh 'java -version'

                sh 'mvn -version'

                sh "${MVN_CMD} clean test"
            }

            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Parallel Tests') {

            failFast false

            parallel {

                stage('Unit Tests') {

                    steps {
                        sh 'mvn test -Dtest=*Unit* -Dsurefire.failIfNoSpecifiedTests=false'
                    }

                    post {
                        always {
                            junit 'target/surefire-reports/*.xml'
                        }
                    }
                }

                stage('Integration Tests') {

                    steps {
                        sh 'mvn verify -Dtest=*IT* -Dsurefire.failIfNoSpecifiedTests=false'
                    }
                }
            }
        }

        stage('Package') {

            steps {
                sh "${MVN_CMD} package"
            }
        }

        stage('Archive Artifacts') {

            steps {

                archiveArtifacts(
                    artifacts: 'target/*.war, target/*.jar',
                    fingerprint: true,
                    allowEmptyArchive: true
                )
            }
        }

        stage('Docker Build & Push') {

            /*
             * Run Docker stage only for:
             * - master branch
             * - release branches
             * - Git tags
             */

            when {

                anyOf {

                    branch 'master'

                    branch pattern: 'release/*', comparator: 'GLOB'

                    buildingTag()
                }
            }

            steps {

                script {

                    def dockerTag

                    if (env.TAG_NAME) {
                        dockerTag = env.TAG_NAME
                    } else if (env.BRANCH_NAME) {
                        dockerTag = env.BRANCH_NAME.replace('/', '-')
                    } else {
                        dockerTag = 'latest'
                    }

                    env.DOCKER_TAG = dockerTag

                    echo "Docker image tag: ${dockerTag}"

                    sh "docker build -t ${DOCKER_REPO}:${dockerTag} ."

                    sh "docker push ${DOCKER_REPO}:${dockerTag}"
                }
            }
        }

        stage('Approve Production Deploy') {

            /*
             * Production deployment is allowed only when:
             *
             * 1. Building master branch
             * 2. DEPLOY_PROD parameter is true
             */

            when {

                allOf {

                    branch 'master'

                    expression {
                        return params.DEPLOY_PROD
                    }
                }
            }

            steps {

                timeout(
                    time: 30,
                    unit: 'MINUTES'
                ) {

                    input(
                        message: 'Deploy to Production?',
                        ok: 'Deploy Now',
                        submitter: 'admin,devops'
                    )
                }

                echo 'Production deployment approved.'
            }
        }
    }

    post {

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check the Jenkins console output.'
        }

        always {
            echo "Build result: ${currentBuild.currentResult}"
        }
    }
}
```
