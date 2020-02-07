def skipBranchBulds = true
if ( env.CHANGE_URL ) {
    skipBranchBulds = false
}

pipeline {
    environment {
        CLOUDSDK_CORE_DISABLE_PROMPTS = 1
        GIT_SHORT_COMMIT = sh(script: 'git describe --always --dirty', , returnStdout: true).trim()
        VERSION = "${env.GIT_BRANCH}-${env.GIT_SHORT_COMMIT}"
        AUTHOR_NAME  = sh(script: "echo ${CHANGE_AUTHOR_EMAIL} | awk -F'@' '{print \$1}'", , returnStdout: true).trim()
    }
    agent {
        label 'docker'
    }
    stages {
        stage('Prepare') {
            when {
                expression {
                    !skipBranchBulds
                }
            }
            steps {
                script {
                    if ( AUTHOR_NAME == 'null' )  {
                        AUTHOR_NAME = sh(script: "git show -s --pretty=%ae | awk -F'@' '{print \$1}'", , returnStdout: true).trim()
                    }  
                }
                withCredentials([file(credentialsId: 'PBM-AWS-S3', variable: 'PBM_AWS_S3_YML'), file(credentialsId: 'PBM-GCS-S3', variable: 'PBM_GCS_S3_YML')]) {
                    sh '''
                        sudo curl -L "https://github.com/docker/compose/releases/download/1.25.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
                        sudo chmod +x /usr/local/bin/docker-compose

                        cp $PBM_AWS_S3_YML ./e2e-tests/docker/conf/aws.yaml
                        cp $PBM_GCS_S3_YML ./e2e-tests/docker/conf/gcs.yaml

                        chmod 664 ./e2e-tests/docker/conf/aws.yaml
                        chmod 664 ./e2e-tests/docker/conf/gcs.yaml
       
                        docker-compose -f ./e2e-tests/docker/docker-compose.yaml build
                        openssl rand -base64 756 > ./e2e-tests/docker/keyFile 
                        sudo chown 1001:1001 ./e2e-tests/docker/keyFile
                        sudo chmod 400 ./e2e-tests/docker/keyFile
                    '''
                }
                sh '''
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml up --no-color -d
                    sleep 10
                    export COMPOSE_INTERACTIVE_NO_CLI=1
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml exec -T cfg01 /opt/start.sh
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml exec -T rs101 /opt/start.sh
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml exec -T rs201 /opt/start.sh
                    sleep 300
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml exec -T mongos mongo /opt/mongos_init.js
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml stop agent-cfg01 agent-cfg02 agent-cfg03  agent-rs101 agent-rs102 agent-rs103 agent-rs201 agent-rs202 agent-rs203
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml start agent-cfg01 agent-cfg02 agent-cfg03  agent-rs101 agent-rs102 agent-rs103 agent-rs201 agent-rs202 agent-rs203
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml start tests
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml exec -T tests ls -al /etc/pbm/    
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml logs -f --no-color tests
                #    sleep 180
                #    docker-compose -f ./e2e-tests/docker/docker-compose.yaml logs --no-color tests
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml ps
                    docker-compose -f ./e2e-tests/docker/docker-compose.yaml down
                '''
            }
        }
    }
    post {
        always {
            deleteDir()
        }
    }
}
