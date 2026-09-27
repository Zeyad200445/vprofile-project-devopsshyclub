def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline {
    agent any
    tools {
        maven "MAVEN3.9"
        jdk "JDK17"
    }
    
    environment {
        SNAP_REPO      = 'vprofile-snapshot'
        NEXUS_USER     = 'admin'
        NEXUS_PASS     = 'admin'
        RELEASE_REPO   = 'vprofile-release'
        CENTRAL_REPO   = 'vpro-maven-central'
        NEXUSIP        = '172.31.16.229'
        NEXUSPORT      = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
        NEXUS_LOGIN    = 'Nexus-Login'
        SONARSERVER    = 'sonarserver'
        SONARSCANNER   = 'sonarscanner'
        NEXUSPASS      = credentials('nexuspass')
    }

    stages {
        stage('Build'){
            steps {
                sh 'mvn -s settings.xml -DskipTests install'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test'){
            steps {
                sh 'mvn -s settings.xml test'
            }
        }

        stage('Checkstyle Analysis'){
            steps {
                sh 'mvn -s settings.xml checkstyle:checkstyle'
            }
        }

        stage("UploadArtifact"){
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: "${env.NEXUSIP}:${env.NEXUSPORT}",
                    groupId: 'QA',
                    version: "${env.BUILD_ID}",
                    repository: "${env.RELEASE_REPO}",
                    credentialsId: "${env.NEXUS_LOGIN}",
                    artifacts: [
                        [
                            artifactId: 'vproapp',
                            classifier: '',
                            file: 'target/vprofile-v2.war',
                            type: 'war'
                        ]
                    ]
                )
            }
        }

        stage('Ansible Deploy to staging'){
            steps {
                ansiblePlaybook([
                    inventory              : 'ansible/stage.inventory',
                    playbook               : 'ansible/site.yml',
                    installation           : 'ansible',
                    colorized              : true,
                    credentialsId          : 'applogin',
                    disableHostKeyChecking : true,
                    extraVars              : [
                        USER             : "admin",
                        PASS             : "${env.NEXUSPASS}",
                        nexusip          : "172.31.16.229",
                        reponame         : "${env.RELEASE_REPO}",
                        groupid          : "QA",
                        time             : "${env.BUILD_ID}",
                        build            : "${env.BUILD_ID}",
                        artifactid       : "vproapp",
                        vprofile_version : "vproapp-${env.BUILD_ID}.war"
                    ]
                ])
            }
        }
    }

    post {
        always {
            echo 'Slack Notifications.'
            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                slackSend(
                    channel: '#jenkinscicd',
                    color: COLOR_MAP[currentBuild.currentResult] ?: 'warning',
                    message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                )
            }
        }
    }
}