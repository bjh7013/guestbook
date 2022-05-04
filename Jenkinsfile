import java.text.SimpleDateFormat

def TODAY = (new SimpleDateFormat("yyyyMMddHHmmss")).format(new Date())

pipeline {
    agent any
    environment {
        strDockerTag = "${TODAY}_${BUILD_ID}"
        strDockerImage ="bjh7013/cicd_guestbook:${strDockerTag}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url:'https://github.com/bjh7013/guestbook.git'
            }
        }
        stage('Build') {
            steps {
                sh './mvnw clean package'
            }
        }
        stage('Unit Test') {
            steps {
                sh './mvnw test'
            }
            
            post {
                always {
                    junit '**/target/surefire-reports/TEST-*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps{
                echo 'SonarQube Analysis'
                /*
                withSonarQubeEnv('SonarQube-Server'){
                    sh '''
                        ./mvnw sonar:sonar \
                        -Dsonar.projectKey=guestbook \
                        -Dsonar.host.url=http://172.31.0.120:9000 \
                        -Dsonar.login=24b9cdbb32a942bb11fd3c930f27f02e74046923
                    '''
                }
                */
            }
        }
        stage('SonarQube Quality Gate'){
            steps{
                echo 'SonarQube Quality Gate'
                /*
                timeout(time: 1, unit: 'MINUTES') {
                    script{
                        def qg = waitForQualityGate()
                        if(qg.status != 'OK') {
                            echo "NOT OK Status: ${qg.status}"
                            error "Pipeline aborted due to quality gate failure: ${qg.status}"
                        } else{
                            echo "OK Status: ${qg.status}"
                        }
                    }
                }
                */
            }
        }
        stage('Docker Image Build') {
            steps {
                script {
                    //oDockImage = docker.build(strDockerImage)
                    oDockImage = docker.build(strDockerImage, "--build-arg VERSION=${strDockerTag} -f Dockerfile .")
                }
            }
        }
        stage('Docker Image Push') {
            steps {
                script {
                    docker.withRegistry('', 'DockerHub_bjh7013') {
                        oDockImage.push()
                    }
                }
            }
        }
        stage('Staging Deploy') {
            steps {
                sshagent(credentials: ['Staging-PrivateKey']) {
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@172.31.0.110 docker container rm -f guestbookapp"
                    sh "ssh -o StrictHostKeyChecking=no ec2-user@172.31.0.110 docker container run \
                                        -d \
                                        -p 38080:80 \
                                        --name=guestbookapp \
                                        -e MYSQL_IP=220.95.250.70 \
                                        -e MYSQL_PORT=3306 \
                                        -e MYSQL_DATABASE=guestbook \
                                        -e MYSQL_USER=root \
                                        -e MYSQL_PASSWORD=education \
                                        ${strDockerImage} "
                }
            }
        }
        stage ('JMeter LoadTest') {
            steps { 
                sh '~/lab/sw/jmeter/bin/jmeter.sh -j jmeter.save.saveservice.output_format=xml -n -t src/main/jmx/guestbook_loadtest.jmx -l loadtest_result.jtl' 
                perfReport filterRegex: '', showTrendGraphs: true, sourceDataFiles: 'loadtest_result.jtl' 
            } 
        }
    }
    post { 
        always { 
            emailext (attachLog: true, body: '본문', compressLog: true
                    , recipientProviders: [buildUser()], subject: '제목', to: 'bjh7013@gmail.com')

        }
        success { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'good'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 성공적으로 끝났습니다. Details: (<${BUILD_URL} | here >)")
        }
        failure { 
            slackSend(tokenCredentialId: 'slack-token'
                , channel: '#교육'
                , color: 'danger'
                , message: "${JOB_NAME} (${BUILD_NUMBER}) 빌드가 실패하였습니다. Details: (<${BUILD_URL} | here >)")
    }
  }
}

