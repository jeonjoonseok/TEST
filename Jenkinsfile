pipeline {
    agent any

    tools {
        jdk 'JDK17'
    }

    environment {
        SERVER_IP = "${DEPLOY_WAS_SERVER_IP}"
        SERVER_USER = "tomcat"
        CATALINA_HOME = "${ENVIRONMENT == 'dev' ? '/var/lib/tomcat' : '/usr/local/tomcat'}"
        DEPLOY_PATH = "${CATALINA_HOME}/webapps"
        BACKUP_PATH = "${DEPLOY_PATH}/backup"
        APP_NAME = "alleyslab"
        WAR_FILE = "${APP_NAME}.war"
        DEPLOY_FILE = "${DEPLOY_PATH}/${WAR_FILE}"
        BACKUP_FILE = "${BACKUP_PATH}/${WAR_FILE}.bak"
        GIT_BRANCH = "${ENVIRONMENT == 'dev' ? 'develop' : 'master'}"
        NGINX_STATIC_PATH = "/var/www/sopmall"
        NGINX_SERVER_IP = "${DEPLOY_WEB_SERVER_IP}"
        NGINX_USER = "${ENVIRONMENT == 'dev' ? 'dev1' : 'ubuntu'}"
//         SSH_KEY_PATH = "C:/Users/CHAMDAHAN-41/.ssh/id_rsa"
    }

    stages {

		stage('Setup parameters') {
            steps {
                script {
                    try {

						echo "Setup parameters 출력"
						echo "tomcat home here ${env.CATALINA_HOME}"
						echo "Setup parameters ${ENVIRONMENT}"
						echo "Setup parameters 출력 종료"

						if("${ENVIRONMENT}" == "dev" ) {
							properties([
		                        parameters ([
							        choice(name: 'ENVIRONMENT', choices: ['dev'], description: '배포할 어플리케이션 profile 선택'),
							        choice(name: 'DEPLOY_WAS_SERVER_IP', choices: ['172.31.26.11'], description: 'WAR 배포할 서버IP 선택'),
							        choice(name: 'DEPLOY_WEB_SERVER_IP', choices: ['172.31.25.11'], description: 'STATIC RESOURCE 배포할 서버IP 선택'),
							        string(name: 'GIT_REPO', defaultValue: 'http://172.31.18.11:3000/gcone/alleyslab.git', description: 'Git 레파지토리 URL'),
							        choice(name: 'DEPLOY_TARGET', choices: ['ALL', 'WEB_ONLY', 'WAS_ONLY'], description: '배포 대상 선택')
							    ])
		                    ])
						}else{
							properties([
		                        parameters([
							        choice(name: 'ENVIRONMENT', choices: ['stage'], description: '배포할 어플리케이션 profile 선택'),
							        choice(name: 'DEPLOY_WAS_SERVER_IP', choices: ['10.100.32.11'], description: 'WAR 배포할 서버IP 선택'),
							        choice(name: 'DEPLOY_WEB_SERVER_IP', choices: ['10.100.22.11'], description: 'STATIC RESOURCE 배포할 서버IP 선택'),
							        string(name: 'GIT_REPO', defaultValue: 'http://172.31.18.11:3000/gcone/alleyslab.git', description: 'Git 레파지토리 URL'),
							        choice(name: 'DEPLOY_TARGET', choices: ['ALL', 'WEB_ONLY', 'WAS_ONLY'], description: '배포 대상 선택')
							    ])
		                    ])
						}
                    } catch (Exception e) {
                        error "Setup parameters 실패: ${e.getMessage()}"
                    }
                }
            }
        }


        stage('Checkout') {
            steps {
                script {
                    try {
                        echo "Checking out ${env.GIT_BRANCH} branch for ${params.ENVIRONMENT} environment"
                        git branch: "${env.GIT_BRANCH}", url: "${params.GIT_REPO}"
                        echo "Checkout 성공: 브랜치 ${env.GIT_BRANCH}, 저장소 ${params.GIT_REPO}"
                    } catch (Exception e) {
                        error "Checkout 실패: ${e.getMessage()}"
                    }
                }
            }
        }



        stage('Prepare and Deploy Static Resources') {
            when {
                expression { params.DEPLOY_TARGET in ['ALL', 'WEB_ONLY'] }
            }
            steps {
                script {
                    if("${RUNNING_OS}" == "linux" ) {
                        // Static 리소스 준비 (linux 환경)
                        sh "mkdir static_resources"
                        sh "cp -r src/main/resources/static/* static_resources/"

                        // Static 리소스 압축 (static_resources 디렉토리 안의 내용만 압축)
                        sh "tar -cvf static_resources.tar -C static_resources ."
                    }else{
                        // Static 리소스 준비 (Windows 환경)
                        bat "mkdir static_resources"
                        bat "xcopy /E /I src\\main\\resources\\static\\* static_resources\\"

                        // Static 리소스 압축 (static_resources 디렉토리 안의 내용만 압축)
                        bat "tar -cvf static_resources.tar -C static_resources ."
                    }

                    // Nginx 서버로 Static 리소스 압축 파일 배포
                    sshCommand remote: [
                        name: 'Nginx Server',
                        host: "${env.NGINX_SERVER_IP}",
                        user: "${env.NGINX_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], command: "mkdir -p ${env.NGINX_STATIC_PATH}"

                    sshPut remote: [
                        name: 'Nginx Server',
                        host: "${env.NGINX_SERVER_IP}",
                        user: "${env.NGINX_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], from: 'static_resources.tar', into: "${env.NGINX_STATIC_PATH}"

                    // Nginx 서버에서 압축 해제
                    sshCommand remote: [
                        name: 'Nginx Server',
                        host: "${env.NGINX_SERVER_IP}",
                        user: "${env.NGINX_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], command: """
                        cd ${env.NGINX_STATIC_PATH}
                        tar -xvf static_resources.tar
                        rm static_resources.tar
                    """

                    echo "Static 리소스가 압축되어 Nginx 서버로 배포 및 압축 해제 완료"
                }
            }
        }



        stage('Build WAR') {
            when {
                expression { params.DEPLOY_TARGET in ['ALL', 'WAS_ONLY'] }
            }
            steps {
                script {
                    try {
                        if("${RUNNING_OS}" == "linux" ) {
                            sh 'chmod +x gradlew'
                            sh "./gradlew clean war -x test -Pprofile=${params.ENVIRONMENT}"
                        }else{
                            bat "./gradlew.bat clean build -Pprofile=${params.ENVIRONMENT}"
                        }
                        echo "WAR Build 성공"
                    } catch (Exception e) {
                        error "WAR Build 실패: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Unit Tests') {
            when {
                expression { params.DEPLOY_TARGET in ['ALL', 'WAS_ONLY'] }
            }
            steps {
                script {
                    try {
                        if("${RUNNING_OS}" == "linux" ) {
                            sh "./gradlew test"
                        }else{
                            bat "./gradlew.bat test"
                        }
                        echo "Unit Tests 성공"
                    } catch (Exception e) {
                        error "Unit Tests 실패: ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Backup WAR') {
            when {
                expression { params.DEPLOY_TARGET in ['ALL', 'WAS_ONLY'] }
            }
            steps {
                script {
                    sshCommand remote: [
                        name: 'WAS Server',
                        host: "${env.SERVER_IP}",
                        user: "${env.SERVER_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], command: """
                        if [ -f ${env.DEPLOY_FILE} ]; then
                            mkdir -p ${env.BACKUP_PATH}
                            cp ${env.DEPLOY_FILE} ${env.BACKUP_FILE}
                            rm -r ${env.CATALINA_HOME}/webapps/${env.WAR_FILE}
                            echo "기존 WAR 파일 백업 완료"
                        else
                            echo "백업할 WAR 파일이 없습니다"
                        fi
                    """
                }
            }
        }

        stage('Deploy WAR to Tomcat') {
            when {
                expression { params.DEPLOY_TARGET in ['ALL', 'WAS_ONLY'] }
            }
            steps {
                script {
                    // WAR 파일 업로드
                    sshPut remote: [
                        name: 'WAS Server',
                        host: "${env.SERVER_IP}",
                        user: "${env.SERVER_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], from: "build/libs/${env.WAR_FILE}", into: "${env.DEPLOY_FILE}"

                    // Tomcat 서비스 상태 확인 후, 실행 중일 경우 shutdown
                    sshCommand remote: [
                        name: 'WAS Server',
                        host: "${env.SERVER_IP}",
                        user: "${env.SERVER_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], command: """
                        if nc -z localhost 8005; then
                            echo "Tomcat이 실행 중입니다. 종료 중..."
                            ${env.CATALINA_HOME}/bin/shutdown.sh
                            sleep 10
                            rm -r ${env.CATALINA_HOME}/webapps/ROOT
                            sleep 10
                        else
                            echo "Tomcat이 실행 중이지 않습니다."
                        fi
                    """

                    sshCommand remote: [
                        name: 'WAS Server',
                        host: "${env.SERVER_IP}",
                        user: "${env.SERVER_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], command: """
                        sleep 10
                        ${env.CATALINA_HOME}/bin/startup.sh
                    """

                    echo "WAR Deploy 완료 (${params.ENVIRONMENT} 환경, ${env.GIT_BRANCH} 브랜치)"
                }
            }
        }
    }

    post {
        success {
            echo '배포가 성공적으로 완료되었습니다.'
        }
        failure {
            script {
                echo '배포에 실패했습니다. 롤백을 시도합니다.'
                if (params.DEPLOY_TARGET in ['ALL', 'WAS_ONLY']) {
                    // WAR 파일 롤백
                    sshCommand remote: [
                        name: 'WAS Server',
                        host: "${env.SERVER_IP}",
                        user: "${env.SERVER_USER}",
                        identityFile: "${env.SSH_KEY_PATH}",
                        allowAnyHosts: true
                    ], command: """
                        if [ -f ${env.BACKUP_FILE} ]; then
                            cp ${env.BACKUP_FILE} ${env.DEPLOY_FILE}
                            if nc -z localhost 8005; then
                                echo "Tomcat이 실행 중입니다. 종료 중..."
                                ${env.CATALINA_HOME}/bin/shutdown.sh
                                sleep 10
                            else
                                echo "Tomcat이 실행 중이지 않습니다."
                            fi
                            sleep 10
                            ${env.CATALINA_HOME}/bin/startup.sh
                            echo "WAR 파일 롤백 완료"
                        else
                            echo "백업 WAR 파일이 존재하지 않아 롤백할 수 없습니다."
                        fi
                    """
                }
                if (params.DEPLOY_TARGET in ['ALL', 'WEB_ONLY']) {
                    echo "Static 리소스 롤백이 필요합니다. 수동으로 진행해 주세요."
                }
            }
        }
        always {
            echo '파이프라인 실행이 종료되었습니다.'
            script {
                if("${RUNNING_OS}" == "linux" ) {
                    // 임시 디렉토리 정리 (linux 환경)
                    sh "rm -rf static_resources"
                }else{
                    // 임시 디렉토리 정리 (Windows 환경)
                    bat """
                    if exist static_resources (
                        rmdir /S /Q static_resources
                    )
                    """
                }
            }
        }
    }
}