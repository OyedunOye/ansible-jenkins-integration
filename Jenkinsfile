pipeline {

agent any
    tools {
        maven 'maven-3.9'
    }

    stages {
        
        stage('copy files to ansible server') {            
            steps {
                script {
                    echo "copying all necessary files to ansible control node"
                    sshagent (['ansible-server-key']) {
                        sh "scp -o StrictHostKeyChecking=no ansible/* root@172.104.233.61:/root"
                        withCredentials([sshUserPrivateKey(credentialsId: 'ansible-configured-ec2-server-key', keyFileVariable: 'keyfile', usernameVariable: 'user')]) {
                            sh 'scp $keyfile root@172.104.233.61:/root/ssh-key.pem'
                        }
                    }
                }
            }
        }

        
    }
}