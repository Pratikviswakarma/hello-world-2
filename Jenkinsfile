pipeline {
    agent {
        docker {
            // Maven image with JDK 17 (change to 11/8 if your project requires)
            image 'maven:3.9.6-eclipse-temurin-17'
            // Run as root to avoid permission issues writing to workspace/.m2.
            args '-u root:root -v $HOME/.m2:/root/.m2'
            reuseNode true
        }
    }

     environment {
        // Optional: speed up Maven, reduce noise
        MAVEN_OPTS = '-Dmaven.repo.local=/root/.m2/repository'
        MVN_CMD    = 'mvn -B -ntp'
        APP_NAME    = 'my-app'
        DOCKER_REPO = 'myregistry/my-app'

    }
    


    parameters {
         string(name: 'GIT_BRANCH', defaultValue: 'master', description: 'Branch to build')
        booleanParam(name: 'DEPLOY_PROD', defaultValue: false,
                     description: 'Deploy to production?')
        choice(name: 'LOG_LEVEL', choices: ['INFO','DEBUG','WARN'],
               description: 'Logging level')
    }

    stages {
 stage('Build + Test') {
            steps {
                sh '${MVN_CMD} clean test'
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
                    steps { sh 'mvn test -Dtest=*Unit* -Dsurefire.failIfNoSpecifiedTests=false' }
                    post { always { junit 'target/surefire-reports/*.xml' } }
                }
                stage('Integration Tests') {
                    steps { sh 'mvn verify -Dtest=*IT* -Dsurefire.failIfNoSpecifiedTests=false' }
                }
               
            }
        }
        stage('Package') {
            steps {
                sh '${MVN_CMD} package'
            }
        }

        stage('Archive Artifacts') {
            steps {
                // This project is packaging=war, so archive WAR (also keep JAR if produced)
                archiveArtifacts artifacts: 'target/*.war, target/*.jar', fingerprint: true
            }
        }
        stage('Docker Build & Push') {
            when {
                anyOf { branch 'master'; branch 'release/*';
                      }
            }
            steps {
                script {
                    def tag = env.TAG_NAME ?: env.BRANCH_NAME.replace('/', '-')
                    env.DOCKER_TAG = tag
                    sh "docker build -t ${DOCKER_REPO}:${tag} ."
                    sh "docker push ${DOCKER_REPO}:${tag}"
                }
            }
        }
           stage('Approve Production Deploy') {
    when {
        allOf {
            branch 'main'
            expression { return params.DEPLOY_PROD }
        }
    }
    steps {
        timeout(time: 30, unit: 'MINUTES') {
            input(
                message: 'Deploy to Production?',
                ok: 'Deploy Now',
                submitter: 'admin,devops'
            )
        }
    }
}

    }
   
}
