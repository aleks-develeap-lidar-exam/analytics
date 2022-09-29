pipeline {
    options {
        disableConcurrentBuilds()
        gitLabConnection gitLabConnection: 'gitlab', jobCredentialId: 'gitlab_apitoken'
        timeout(activity: true, time: 5)
    }
    
    tools {
        jdk 'jdk8'
        maven '3.6.2'
    }
    agent any

        triggers {
        gitlab(
            triggerOnPush: true,
            branchFilterType: 'All'
        )
    }

    stages {

    stage('Calculate version'){
        when {
                branch "release/*"
            }
        steps {
            script {
            branchNumber = env.BRANCH_NAME.split("/")[1]
            sh "git fetch --tags"
            oldTag = sh(script: "git describe --tags --abbrev=0 || true", returnStdout: true).trim()
            if (!oldTag) {
                        finalNum = "0"
                    } else {
                        finalNum = (oldTag.tokenize(".")[2].toInteger() + 1).toString()
                    }

                    env.VERSION = branchNumber + "." + finalNum
                    echo env.VERSION
                    sh "mvn versions:set -DnewVersion=$env.VERSION"
            }
        }
    }


    stage('Build') {
        when {
            branch "main"
        }
      steps {
        sh "mvn package -Dskip.unit-tests=true"
      }
    }

    stage('Build and unit test'){
        when {
            anyOf{
                branch 'feature/*'
                branch 'release/*'
            } 
        }
        steps{
            configFileProvider([configFile(fileId: 'exam_maven_settings', variable: 'SETTINGS')]) {
                sh "mvn package"
            }
    
        }
        }
    
    stage('E2E test'){
        when {
            anyOf {
                branch 'main'
                branch 'release/*'
                allOf {
                    environment name: 'E2E_TEST', value: 'true'
                    branch 'feature/*'
                }
            }     
        } 
        steps{
            sh "mkdir test"
            //add different x.y ver here
            withCredentials([usernamePassword(credentialsId: 'aleks_jfrog', passwordVariable: 'password', usernameVariable: 'myUser')]) {
                sh "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT/simulator-99-20220929.101554-1.jar --output test/simulator.jar"
                sh "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-release-local/com/lidar/telemetry/1.0.2/telemetry-1.0.2.jar --output test/telemetry.jar"
        
            }
            withCredentials([string(credentialsId: 'testing_api', variable: 'token')]) {
                sh "curl --header 'PRIVATE-TOKEN: $token' http://gitlab/api/v4/projects/8/repository/files/tests-sanity.txt/raw?ref=main --output test/tests.txt"
            }
            


        }  

    }
 


    stage('Publish') {
        when {
            anyOf{
                branch 'main'
                branch 'release/*'
            }  
        }
        steps {
            configFileProvider([configFile(fileId: 'exam_maven_settings', variable: 'SETTINGS')]) {
            sh "mvn deploy -s $SETTINGS -Dmaven.test.skip"
            }
        }
    }
        stage('Tag'){
        when {
                branch "release/*"
            }
        steps{
            script{
                    sh "git clean -f -x"
                    sh "git tag -a ${env.VERSION} -m 'version ${env.VERSION}'"
                    sh "git push --tag"
            }
        }


    }

    
    }
}

