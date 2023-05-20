//********************************************************************
// Updated on 29 April 2022
//**************************Change History****************************
// Jyoti Choudhury     2022-03-24     Adoption of Jenkins Enterprise
// Jyoti Choudhury     2022-04-29     Capability to update pom version
//                                    and creating release for prod
//********************************************************************
// Version 1.0.1-SNAPSHOT
//********************************************************************

boolean success(String message) {
    currentBuild.result = 'SUCCESS'
    echo message
    return false
}
boolean shouldRun() {
    return true
}
String getRepoName(String repoUrl) {
    return repoUrl.tokenize('/')[3].split('\\.')[0]
}
String getCommonRepoUrl() {
    return env.APIGW_COMMON_REPO
}
String getFTRepoUrl() {
    return env.APIGW_FT_REPO
}
String getCommonRepoCred() {
    return env.APIGW_COMMON_REPO_CREDID
}
String getEnvrionmentName() {
    if (env.BRANCH_NAME == 'dev') {
        return "DEV"
    } else if (env.BRANCH_NAME == 'qa') {
        return "QA2"
    } else if (env.BRANCH_NAME == 'release') {
        return "PET"
    } else if (env.BRANCH_NAME == 'master') {
        return "PROD"
    }
}
void toolSh(String command) {
    container('java-maventools') {
        sh command
    }
}
String getVersionFromPom() {
    container('java-maventools') {
        return sh(label: 'Get Version From POM', script: 'maven-pom-version.sh', returnStdout: true).trim()
    }
}
void pushVersionUpdate(String newVersion) {
    toolSh 'maven-push-pom.sh'
}
void updatePomFile(String newVersion) {
    toolSh "maven-update-version.sh ${newVersion}"
}
void updateVersion() {
    String currentVersion = getVersionFromPom()
    String prefix = ""
    if(currentVersion.indexOf("SNAPSHOT",0) != -1) {
        prefix = currentVersion.substring(0,(currentVersion.indexOf("SNAPSHOT",0) + 8)) + "-"
    }
    else {
        prefix = currentVersion + "-SNAPSHOT-"
    }
    String nextVersion = prefix + GIT_COMMIT
    updatePomFile(nextVersion)
    pushVersionUpdate(nextVersion)
}
pipeline {
    agent {
    	kubernetes {
            yamlFile 'jenkins-agent.yml'
            podRetention never()
        }
    }
    environment {
        ARTIFACTORY_CREDENTIALS = credentials('artifactory_serv_svc_dawsdev')
        ARTIFACTORY_URL = 'https: //artifactory.rogers.com:443/artifactory'
        ARTIFACTORY_VIRTUAL_REPO_KEY = 'rogers-maven-sandbox-virtual'
        GITHUB_CREDENTIALS = credentials('github_apigwebu_svc_account')
    }
    stages {
        stage('Run CI?') { 
            when { expression { shouldRun() } }
            stages {
                stage('Setup') {
                    // 'Setup' stage set to run in all branches
                    parallel {
                        stage('Environment Variables') {
                            steps { sh 'env | sort'}
                        }
                        stage('Credentials') {
                            steps { 
                                toolSh 'artifactory-maven-credentials.sh'
                                toolSh 'github-credentials.sh'
                            }
                        }
                        stage('Checkout') {
                            steps { 
                                toolSh 'artifactory-maven-credentials.sh'
                                script {
                                    checkout(
                                        [$class: 'GitSCM', branches: [[name: '*/master']], 
                                        doGenerateSubmoduleConfigurations: false, 
                                        extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: getRepoName(getCommonRepoUrl())]], 
                                        submoduleCfg: [], 
                                        userRemoteConfigs: [[credentialsId: getCommonRepoCred(), url: getCommonRepoUrl()]]]
                                    )
                                }

                                sh "ls ${WORKSPACE}"
                                toolSh "cp ./jenkins-mule-log/settings.xml \${WORKSPACE}"
                                toolSh "cp ./jenkins-mule-log/settings-security.xml \${WORKSPACE}"

                                sh "ls ${WORKSPACE}"
                                toolSh "cp ${WORKSPACE}/settings.xml \$MAVEN_USER_HOME"
                                toolSh "cp ${WORKSPACE}/settings-security.xml \$MAVEN_USER_HOME"
                            }
                        }
                    }
                }
                stage('Update POM Version') {
                    when {
                        expression { 
                            getEnvrionmentName() == "DEV"
                        }
                    }
                    steps {
                        echo "Update POM Version - Release to DEV"
                        script { updateVersion() }
                    }
                }
                stage('Release to PROD') {
                    when { 
                        expression {
                            getEnvrionmentName() == "PROD"
                        }				
                    }
                    steps {
                        echo "Creating Production Release Version"
                        //toolSh 'maven-semantic-release-no-push.sh'
                        toolSh 'maven-semantic-release.sh'
                    }
                }
                stage('MUnit') {
                    when { 
                        expression {
                            //getEnvrionmentName() == "DEV" ||
                            //getEnvrionmentName() == "QA2" ||
                            getEnvrionmentName() == "PET"
                        }				
                    }
                    steps {
                        echo 'MUnit'
                        toolSh 'mvn clean test'
                    }
                }
                stage('Sonarqube') {
                    when { 
                        expression { 
                            getEnvrionmentName() == "DEV" ||
                            getEnvrionmentName() == "QA2" ||
                            getEnvrionmentName() == "PET" ||
                            getEnvrionmentName() == "PROD"
                        }				
                    }
                    steps {
                        echo 'Sonarqube'
                        toolSh 'mvn sonar:sonar -Dsonar.host.url=https://sonarqube.rogers.com -Dsonar.login=3eec97107b3e065acab1829bfe1a871931f21ab1 -Dsonar.mule4.file.suffixes=xml -Dsonar.xml.file.suffixes=.xmlobs -Dsonar.projectVersion=$BUILD_NUMBER -Dsonar.language=mule4 -Dsonar.branch.name=master'
                    }
                }
                stage('Exchange Publish') {
                    when { 
                        expression { 
                            getEnvrionmentName() == "DEV" ||
                            getEnvrionmentName() == "PROD"
                        }				
                    }
                    steps {
                        echo 'Exchange Publish'
                        toolSh 'mvn -DskipTests deploy -Pexch'
                    }
                }
                stage('DEV Deploy') {
                    when { expression { getEnvrionmentName() == "DEV" } }
                    steps {
                        echo 'Deploying to DEV'
                        toolSh 'mvn -DskipTests -Pdev deploy -DmuleDeploy'
                    }
                }
                stage('QA2 Deploy') {
                    when { expression { getEnvrionmentName() == "QA2" } }
                    steps {
                        echo 'Deploying to QA2'
                        toolSh 'mvn -DskipTests -Pqa2 deploy -DmuleDeploy'
                    }
                }
                stage('PET Deploy') {
                    when { expression { getEnvrionmentName() == "PET" } }
                    steps {
                        echo 'Deploying to PET'
                        toolSh 'mvn -DskipTests -Ppet deploy -DmuleDeploy'
                    }
                }
                stage('PROD Deploy') {
                    when { expression { getEnvrionmentName() == "PROD" } }
                    steps {
                        echo 'Deploying to PROD'
                        toolSh 'mvn -DskipTests -Pprod deploy -DmuleDeploy'
                    }
                }
                stage('Functional Testing') {
                    when { 
                        expression { 
                            getEnvrionmentName() == "QA2" ||
                            getEnvrionmentName() == "PET"
                        }				
                    }
                    steps {
                        echo 'Functional Testing'

                        toolSh 'artifactory-maven-credentials.sh'
                        script {
                            checkout(
                                [$class: 'GitSCM', branches: [[name: '*/main']], 
                                doGenerateSubmoduleConfigurations: false, 
                                extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: getRepoName(getFTRepoUrl())]], 
                                submoduleCfg: [], 
                                userRemoteConfigs: [[credentialsId: getCommonRepoCred(), url: getFTRepoUrl()]]]
                            )
                            def repoName = getRepoName(env.GIT_URL)
                            def ftRepoName = getRepoName(getFTRepoUrl())
                            sh "ls ${WORKSPACE}/${ftRepoName}/script"
                            toolSh "Gradle clean -Dsuits"
                        }
                    }
                }
            }
        }
    }
    post { 
        always {
            emailext attachLog: true, body: "Hello ${env.BUILD_USER}, \n\nPlease find the latest build result of Pipeline job you ran.\n\nJob Name: ${env.JOB_NAME} Build# ${env.BUILD_NUMBER} \n\nBuild Status: ${currentBuild.currentResult} \n\nYou can see more details at ${env.BUILD_URL} \n\nThanks and regards \nAPIGW Team", subject: "Jenkins Build - ${env.JOB_NAME} - ${env.BUILD_NUMBER} - ${currentBuild.currentResult}", to: "${env.BUILD_USER_EMAIL}"
        }
    }
}
