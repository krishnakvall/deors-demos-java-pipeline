#!groovy

pipeline {
    agent {
        label "jenkins maven"
    }

    environment {
        ORG_NAME = "deors"
        APP_NAME = "deors-demos-java-pipeline"
        APP_VERSION = "1.0-SNAPSHOT"
        APP_CONTEXT_ROOT = "/"
        APP_LISTENING_PORT = "8080"
        TEST_CONTAINER_NAME = "ci-appname-buildnumber"
        DOCKER_HUB = credentials("orgname-docker-hub")
    }

    stages {
        stage('Compile') {
            steps {
                echo "-=- compiling project -=-"
                sh "./mvnw clean compile"
            }
        }

        stage('Unit tests') {
            steps {
                echo "-=- execute unit tests -=-"
                sh "./mvnw test org.jacoco:jacoco-maven-plugin:report"
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
            }
        }

        stage('Mutation tests') {
            steps {
                echo "-=- execute mutation tests -=-"
                sh "./mvnw org.pitest:pitest-maven:mutationCoverage"
            }
        }

        stage('Package') {
            steps {
                echo "-=- packaging project -=-"
                sh "./mvnw package -DskipTests"
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build Docker image') {
            steps {
                echo "-=- build Docker image -=-"
                container(name: 'kaniko', shell: '/busybox/sh') {
                    ansiColor('xterm') {
                        sh '''#!/busybox/sh
                            /kaniko/executor -f `pwd`/Dockerfile.run -c `pwd` --cache=true --destination=${env.IMAGE}
                        '''
                    }
                }
                // sh "docker build -t ${env.ORG_NAME}/${env.APP_NAME}:${env.APP_VERSION} -t ${env.ORG_NAME}/${env.APP_NAME}:latest ."
            }
        }

        stage('Run Docker image') {
            steps {
                echo "-=- run Docker image -=-"
                sh "docker run --name ${env.TEST_CONTAINER_NAME} --detach --rm --network ci --expose 6300 --env JAVA_OPTS='-javaagent:/jacocoagent.jar=output=tcpserver,address=*,port=6300' ${env.ORG_NAME}/${env.APP_NAME}:latest"
            }
        }

        stage('Integration tests') {
            steps {
                echo "-=- execute integration tests -=-"
                sh "curl --retry 5 --retry-connrefused --connect-timeout 5 --max-time 5 http://${env.TEST_CONTAINER_NAME}:${env.APP_LISTENING_PORT}/${env.APP_CONTEXT_ROOT}/actuator/health"
                sh "./mvnw failsafe:integration-test failsafe:verify -DargLine=\"-Dtest.selenium.hub.url=http://selenium-hub:4444/wd/hub -Dtest.target.server.url=http://${env.TEST_CONTAINER_NAME}:${env.APP_LISTENING_PORT}/${env.APP_CONTEXT_ROOT}\""
                sh "java -jar target/dependency/jacococli.jar dump --address ${env.TEST_CONTAINER_NAME} --port 6300 --destfile target/jacoco-it.exec"
                sh "mkdir target/site/jacoco-it"
                sh "java -jar target/dependency/jacococli.jar report target/jacoco-it.exec --classfiles target/classes --xml target/site/jacoco-it/jacoco.xml"
                junit 'target/failsafe-reports/*.xml'
                jacoco execPattern: 'target/jacoco-it.exec'
            }
        }

        stage('Performance tests') {
            steps {
                echo "-=- execute performance tests -=-"
                sh "./mvnw jmeter:configure jmeter:jmeter jmeter:results -Djmeter.target.host=${env.TEST_CONTAINER_NAME} -Djmeter.target.port=${env.APP_LISTENING_PORT} -Djmeter.target.root=${env.APP_CONTEXT_ROOT}"
                perfReport sourceDataFiles: 'target/jmeter/results/*.csv', errorUnstableThreshold: 0, errorFailedThreshold: 5, errorUnstableResponseTimeThreshold: 'default.jtl:100'
            }
        }

        stage('Web page performance analysis') {
            steps {
                echo "-=- execute web page performance analysis -=-"
                sh "apt-get update"
                sh "apt-get install -y gnupg"
                sh "echo \"deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main\" | tee -a /etc/apt/sources.list.d/google.list"
                sh "curl -sL https://dl.google.com/linux/linux_signing_key.pub | apt-key add -"
                sh "curl -sL https://deb.nodesource.com/setup_10.x | bash -"
                sh "apt-get install -y nodejs google-chrome-stable"
                sh "npm install -g lighthouse@5.6.0"
                sh "lighthouse http://${env.TEST_CONTAINER_NAME}:${env.APP_LISTENING_PORT}/${env.APP_CONTEXT_ROOT}/hello --output=html --output=csv --chrome-flags=\"--headless --no-sandbox\""
                archiveArtifacts artifacts: '*.report.html'
                archiveArtifacts artifacts: '*.report.csv'
            }
        }

        stage('Dependency vulnerability tests') {
            steps {
                echo "-=- run dependency vulnerability tests -=-"
                sh "./mvnw dependency-check:check"
                dependencyCheckPublisher failedTotalHigh: 2, unstableTotalHigh: 2, failedTotalMedium: 5, unstableTotalMedium: 5
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                echo "-=- run code inspection & check quality gate -=-"
                withSonarQubeEnv('ci-sonarqube') {
                    sh "./mvnw sonar:sonar"
                }
                timeout(time: 10, unit: 'MINUTES') {
                    //waitForQualityGate abortPipeline: true
                    script  {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK' && qg.status != 'WARN') {
                            error "Pipeline aborted due to quality gate failure: ${env.qg.status}"
                        }
                    }
                }
            }
        }

        stage('Push Docker image') {
            steps {
                echo "-=- push Docker image -=-"
                withDockerRegistry([ credentialsId: "${env.ORG_NAME}-docker-hub", url: "" ]) {
                    sh "docker push ${env.ORG_NAME}/${env.APP_NAME}:${env.APP_VERSION}"
                    sh "docker tag ${env.ORG_NAME}/${env.APP_NAME}:${env.APP_VERSION} ${env.ORG_NAME}/${env.APP_NAME}:latest"
                }
            }
        }
    }

    post {
        always {
            echo "-=- remove deployment -=-"
            sh "docker stop ${env.TEST_CONTAINER_NAME}"
        }
    }
}
