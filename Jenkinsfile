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

    stage("Set E2E flag") {
            steps {
                script{
                    GIT_MESSAGE = sh (
                    script: "git log --format=%B -n 1 $GIT_COMMIT",
                    returnStdout: true
                    ).trim()
                    if ("${GIT_MESSAGE}".contains('#e2e')){
                        env.E2E_TEST = true
                        
                    }
                }

            }
        }



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
            sh "mkdir -p test"            
            withCredentials([usernamePassword(credentialsId: 'aleks_jfrog', passwordVariable: 'password', usernameVariable: 'myUser')]) {
                script{
            //get newest telemetry version
            TELEMETRY_VERSION = sh(returnStdout: true, script: "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-release-local/com/lidar/telemetry/maven-metadata.xml | grep '<version>' | tail -1 | grep -o '[0-9].[0-9].[0-9]'").trim()
            }
                
                sh "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-snapshot-local/com/lidar/simulator/99-SNAPSHOT/simulator-99-20220929.101554-1.jar --output test/simulator.jar"

            sh "curl -u $myUser:$password http://artifactory:8082/artifactory/exam-libs-release-local/com/lidar/telemetry/${TELEMETRY_VERSION }/telemetry-${TELEMETRY_VERSION}.jar --output test/telemetry.jar"
        
            }
            //curl gitlab for the test file
            withCredentials([string(credentialsId: 'testing_api', variable: 'token')]) {
                sh "curl --header 'PRIVATE-TOKEN: $token' http://gitlab/api/v4/projects/8/repository/files/tests-sanity.txt/raw?ref=main --output test/tests.txt"
            }
            script {
            if( env.BRANCH_NAME != 'release/*' ){
                sh "cp target/analytics-99-SNAPSHOT.jar test/analytics.jar"
            }
            else{
                sh "cp target/analytics-{env.VERSION}.jar test/analytics.jar"
            }
            }

            dir('test'){
                sh "java -cp simulator.jar:analytics.jar:telemetry.jar com.lidar.simulation.Simulator"
            }
            sh "rm -r test"


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

