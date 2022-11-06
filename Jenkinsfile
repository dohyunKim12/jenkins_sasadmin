pipeline {
    agent any 
    environment {
        gitUrl = "192.168.1.150:10081/superobject/super-app-admin.git"
        gitCred = "Rbxxb7pBtyw6D_KqbNWa"
        gitBranch = "master"
        version = "${params.version}"
    }
    stages {
        stage('Git Clone') {
            steps {
                deleteDir()
                sh 'ls -al'
                sh 'rm -rf *'
                sh 'rm -rf .g*'
                script {
                    echo "${env.WORKSPACE}"
                    credId = 'gitlab_token'
                    }
                    git([branch: "${gitBranch}", credentialsId:"${credId}", url: "http://${gitCred}@${gitUrl}"])
                }
        }
        stage('Git Tagging') {
            steps {
                sh 'git fetch --all'
               script {
                    echo "****************************************This is master!*********************************"
                    sh "git checkout -b release-${version}"
                    commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                    commitId = commitId.substring(1)
                    tagName = "release-${version}"
                    sh "git tag -a ${tagName} -m 'Version ${version} update'"
                }
            }
        }
        stage('Build Jar') {
            steps {
               echo "${version}"
               sh 'chmod +x ./gradlew'
               sh "./gradlew clean build jenkins -PbuildVersion=${version} -PcommitId=${commitId}"
           }
        }
        stage('Upload to ftp server') {
            steps {
                sh "sudo scp -r ./bin/libs/sasctl-${version}.jar ck-ftp@192.168.1.150:~/binary/super-app-runtime/super-app-runtime-${version}/sasctl-${version}.jar"
                sh "sudo scp -r ./bin/libs/sasctl-${version}.jar root@192.168.9.12:/root/apache2-data/binary/super-app-runtime/super-app-runtime-${version}/sasctl-${version}.jar"
           }
        }
        stage('Git Push') {
            steps {
                echo "pushing..."
                script {
                    commitMsg = "Release commit - version ${version}"
                    sh "git add -A"
                    sh "git commit -m \"${commitMsg}\" || true"
                    sh "git remote rm origin"
                    sh "git remote add origin http://dohyun_kim5:ehgus0303!@${gitUrl}"
                    sh "git remote -v"
                    sh "git push origin refs/tags/${tagName}:refs/tags/${tagName}"
                    sh "git push origin refs/heads/${gitBranch}:refs/heads/${gitBranch}"
                }
            }
        }
        stage('Cleaning...') {
            steps {
                echo "All work Done. Cleaning..."
                echo "Check the @tmp directory for confirm."
                script {
                    sh "sudo echo ${WORKSPACE}"
                    sh "sudo rsync -a ${env.WORKSPACE} ${env.WORKSPACE}@tmp"
                    sh "sudo echo ${WORKSPACE}"
                    sh "cd ${WORKSPACE}"
                    sh "sudo chown -R jenkins ."
                }
                deleteDir()
            }
        }
    }
}