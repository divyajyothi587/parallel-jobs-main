def ENABLE_DEPLOY = ''


//derive settings from scm branch name
switch (BRANCH_NAME) {
    case "develop":
        ENABLE_DEPLOY = 'true'
        break
    case "main":
        ENABLE_DEPLOY = 'true'
        break
}

if (JOB_NAME == 'job-seeder/main') {
    node() {
        stage('Checkout SCM') {
            checkout scm
        }
        def scmUrl = scm.getUserRemoteConfigs()[0].getUrl()
        withEnv(["scmUrl=${scmUrl}"]) {
            stage('Update jobs') {
                sh '''#!/bin/bash
                    jobfile="$(mktemp)"
                    envsubst < Jenkinsjob > "${jobfile}"
                    echo -e "\n\n" >> "${jobfile}"
                    envsubst < ./devops/jenkinsjob-template >> "${jobfile}"
                    jenkins-jobs --conf ${JENKINS_HOME}/keystore/jenkins_jobs.ini update "${jobfile}" --delete-old
                    rm -rf "${jobfile}"
                '''
            }
        }
    }
} else {
    if (ENABLE_DEPLOY == 'true') {
        node() {
            stage('Checkout SCM') {
                checkout scm
            }
            stage('nothing') {
                echo "JOB_NAME is : ${JOB_NAME}"
            }
        }
    }
}