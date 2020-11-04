#!groovy
// execute this before anything else, including requesting any time on an agent
if (currentBuild.rawBuild.getCauses().toString().contains('BranchIndexingCause')) {
  print "INFO: Build skipped due to trigger being Branch Indexing"
  return
}
pipeline {
    agent any
    triggers {
        githubPush()
    }
    environment {
        BUILD_VERSION = "build.${currentBuild.number}"
        CLIENT_ID = credentials('anypoint.platform.clientId')
        CLIENT_SECRET = credentials('anypoint.platform.clientSecret')
        ANYPOINT_PLATFORM_CREDS = credentials('anypoint.platform.account')
        ASSET_VERSION = "v1"
    }
    stages {
        stage('Compile and Test') {
            steps {
                echo "Starting Compile and Test..."
                withMaven(
                    // Default Maven installation declared in the Jenkins "Global Tool Configuration"
                    maven: 'Maven-3.6.3',
                    // Maven settings.xml file defined with the Jenkins Config File Provider Plugin
                    mavenSettingsConfig: '5b8a4952-9763-4685-9fd0-e17521e77ac1') {
                        sh """ mvn clean compile test """
                    }
                echo "Compile and Test completed: ${currentBuild.currentResult}"
            }
            post {
                success {
                    echo "...Compile and Test Succeeded for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                } 
                failure {
                    echo "...Build and Test Failed for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                }
            }
        }
        stage('Deploy to Development') {
            when {
                not { branch 'master' }
            }
            steps {
                script {
                    echo "Starting Deployment to Development"
                    def pom = readMavenPom file: 'pom.xml'
                    print "POM artifactId: " + pom.artifactId
                    print "POM version: " + pom.version
                    withMaven(
                        // Default Maven installation declared in the Jenkins "Global Tool Configuration"
                        maven: 'Maven-3.6.3',
                        // Maven settings.xml file defined with the Jenkins Config File Provider Plugin
                        mavenSettingsConfig: '5b8a4952-9763-4685-9fd0-e17521e77ac1') {
                            sh """ mvn --batch-mode package mule:deploy \
                                    -Dmule.env=DEV \
                                    -Danypoint.username=$ANYPOINT_PLATFORM_CREDS_USR \
                                    -Danypoint.password=$ANYPOINT_PLATFORM_CREDS_PSW \
                                    -Dcloudhub.application.name=${pom.artifactId}-dev-$ASSET_VERSION \
                                    -Dcloudhub.environment=Development \
                                    -Dbusiness.group.name=MuleSoft \
                                    -Dartifact.path=target/${pom.artifactId}-${pom.version}-mule-application.jar \
                                    -Dcloudhub.workers=1 \
                                    -Dcloudhub.worker.type=MICRO \
                                    -Dcloudhub.region=ap-southeast-2 \
                                    -Danypoint.platform.client.id=$CLIENT_ID \
                                    -Danypoint.platform.client.secret=$CLIENT_SECRET """
                    }
                    echo "Deployment to Development completed: ${currentBuild.currentResult}"
                }
            }    
            post {
                success {
                    echo "...Deploy to Development Succeeded for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                } 
                failure {
                    echo "...Deploy to Development Failed for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                }
            }
        }
        stage('Maven Release') {
            when {
                allOf {
                    branch 'master'
                    expression{ env.BUILD_VERSION != 'build.1' }
                }
            }
            steps {
                echo "Starting Maven Release..."
                withMaven(
                    // Default Maven installation declared in the Jenkins "Global Tool Configuration"
                    maven: 'Maven-3.6.3',
                    // Maven settings.xml file defined with the Jenkins Config File Provider Plugin
                    mavenSettingsConfig: '5b8a4952-9763-4685-9fd0-e17521e77ac1') {
                        sh """ mvn --batch-mode release:clean release:prepare release:perform """
                    }
                echo "Maven Release completed: ${currentBuild.currentResult}"
            }
            post {
                success {
                    echo "...Maven Release Succeeded for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                } 
                failure {
                    echo "...Maven Release Failed for ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
                }
            }
        }
    }
    post {
        success {
            echo "All Good: ${env.BUILD_VERSION}: ${currentBuild.currentResult}"
        }
        failure {
            echo "Not So Good: ${env.BUILD_VERSION}:? ${currentBuild.currentResult}"
        }      
        always {
            echo "Pipeline result: ${currentBuild.result}"
            echo "Pipeline currentResult: ${currentBuild.currentResult}"
        }
    }
}
