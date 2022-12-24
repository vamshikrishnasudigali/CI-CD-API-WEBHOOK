pipeline {
    agent any
    tools { 
        maven 'Maven-360' 
        jdk 'JAVA-11' 
    }
    /*environment {
        PATH = "$PATH:/opt/apache-maven-3.8.2/bin"
    }*/

    stages {
        stage('Clone the Code') {
            steps {
                git 'https://github.com/futuretechdevops/hello-world.git'
            }
        }
        stage('MVN Build and Publish the Unit Test Results') {
            steps {
                sh 'mvn clean install'
            }
            post {
                always {
                    junit(
                        allowEmptyResults: true,
                        testResults: '**/target/surefire-reports/*.xml'
                    )
                }
            }
        }
        stage ('Code Quality') {
            environment {
                scannerHome = tool 'sonar-scanner-4.7'
            }
            steps {
                withSonarQubeEnv('SonarQube-Server-CE-9.8') {
                    sh "${scannerHome}/bin/sonar-scanner \
                    -D sonar.projectKey=cicd-demo \
                    -D sonar.exclusions=vendor/**,resources/**,**/*.java"
                }
            }
        }
        stage('SonarQube Quality Gates Check'){
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        stage('Artifact-Upload') {
            steps {
                echo "build-version ${env.POM_VERSION}"
                script {
                    env.POM_VERSION = sh(returnStdout: true, script: "cat pom.xml  |grep 'version' |head -1 |awk -F '[><]' '{print \$3}'").trim()
                    env.RELEASE_VERSION = sh(returnStdout: true, script: "cat pom.xml  |grep 'version' |head -1 |awk -F '[><]' '{print \$3}' |awk -F '[-]' '{print \$2}'").trim()
                    sh '[[ -d webapp/target2 ]] && echo "target2 directory exist" || mkdir webapp/target2'
                    sh '[[ $(ls -A webapp/target2) ]] && rm -rf webapp/target2/* || echo "target2 empty directory"'
                    sh 'cp webapp/target/webapp.war webapp/target2/$BUILD_NUMBER-$RELEASE_VERSION-webapp.war' 
                    
                    if (env.RELEASE_VERSION == 'RELEASE') {
                        env.RELEASE_VERSION = "Release"
                        echo "${env.RELEASE_VERSION}"
                    }
                    else {
                        env.RELEASE_VERSION = "Snapshot"
                        echo "${env.RELEASE_VERSION}"
                    }
                    rtServer (
                        id: "jfrog", 
                        url: "http://10.10.0.5:8082",
                        credentialsId: "4fe16cb0-df33-4b63-a117-68c127eacd40"
                    )
                    rtUpload (
                        serverId: 'jfrog',
                        spec: '''{
                            "files": [
                                {
                               "pattern": "webapp/target2/*.war",
                               "target": "artifactory/\$RELEASE_VERSION/com/example/maven-project/webapp/\$POM_VERSION/"
                                }
                            ]
                        }''',
                        buildName: "$JOB_NAME",
                        buildNumber: "$BUILD_NUMBER"
                    )
                }
                stash includes: '**/target/*.war', name: "$JOB_NAME-WARfile"
                stash includes: 'Dockerfile', name: "$JOB_NAME-Dockerfile"
            }
        }
        stage('docker-build/tag/push') {
            agent {
                label 'docker'
            }
            steps {
                unstash "$JOB_NAME-WARfile"
                unstash "$JOB_NAME-Dockerfile"
                //sh 'printenv'
                //sh 'hostname -i && pwd && ls && whoami'
                sh 'docker build -t futuretechdevops/tomcat:\$BUILD_NUMBER-\$RELEASE_VERSION . -f Dockerfile'
                sh 'docker push futuretechdevops/tomcat:\$BUILD_NUMBER-\$RELEASE_VERSION'
            }
        }
        stage('docker-run || Deployment') {
            agent {
                label 'docker'
            }
            steps {
                script {
                    env.STABLE_IMAGE = sh(script: 'docker ps -a --format \"table {{.Image}}\\t{{.Names}}\" |grep tomcat |awk \'{print \$1}\'', returnStdout: true).trim()
                    echo "${env.STABLE_IMAGE}"
                    sh 'docker stop tomcat || true && docker rm tomcat || true'
                    sh 'docker run -d --name tomcat -p 8090:8090 futuretechdevops/tomcat:\$BUILD_NUMBER-\$RELEASE_VERSION'
                }
            }
        }
        stage('API || Sanity Testing') {
            agent {
                label 'docker'
            }
            steps {
                script {
                    try {
                        echo "${env.STABLE_IMAGE}"
                        sh 'sleep 45'
                        def code = sh(script: 'curl -o /dev/null -s -w "%{http_code}\n" http://127.0.0.1:8090/webapp/', returnStdout: true).trim()
                        echo "HTTP response status code: $code"
                        def response = sh(script: 'curl -i http://127.0.0.1:8090/webapp/', returnStdout: true)
                        
                        if (code != '200') {
                            echo response
                            //currentBuild.result = 'UNSTABLE'
                            //currentStage.result = 'UNSTABLE'
                            sh 'exit 1'
                        }
                    }
                    catch (Exception e) {
                        sh 'docker stop $(docker ps -a |grep tomcat |awk "{print \\$1}")'
                        sh 'docker rm $(docker ps -a |grep tomcat |awk "{print \\$1}")'
                        sh 'echo "tomcat container STOPPED and REMOVED"'
                        
                        //sh 'printenv'
                        
                        sh 'docker run -d --name tomcat -p 8090:8090 $STABLE_IMAGE'
                        sh 'echo "Started tomcat container with STABLE_IMAGE: $STABLE_IMAGE"'
                        currentBuild.result = 'FAILED'
                        urrentStage.result = 'FAILED'
                        sh 'exit 1'
                    }
                    finally {
                        sh 'echo "tomcat Container final status"'
                        sh 'docker ps -a |grep tomcat'
                    }
                }
            }
        }
    }
    post { 
        always { 
            echo 'Email Notification sent to : always receipents'
        }
        success {
            echo 'Email Notification sent to : Succeess receipents --> I Succeeded!'
        }
        unstable {
            echo 'Email Notification sent to : DEV/DEVOPS receipents --> I unstable :/'
        }
        failure {
            echo 'Email Notification sent to : Failure receipents --> I failed :('
        }
        changed {
            echo 'Email Notification sent to : changed receipents --> Things were different before...'
        }
    }
}
