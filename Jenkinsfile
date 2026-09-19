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

        stage('execute ansible playbook') {
            steps {
                script {
                    echo "calling ansible playbook to configure ec2 instances"
                    def remote = [:]
                    remote.name = "my-ansible-server"
                    remote.host = "172.104.233.61"
                    remote.allowAnyHosts = true
                    
                    withCredentials([sshUserPrivateKey(credentialsId: 'ansible-server-key', keyFileVariable: 'keyfile', usernameVariable: 'user')]) {
                        remote.user = user
                        remote.identityFile = keyfile
                        sshCommand remote: remote, command: "ansible-playbook my-playbook.yaml"
                    }
                }
            }
        }
        
    }
}