pipeline {
    agent any 
    environment {
      // gitlab
        gitUrl = "192.168.1.150:10081/superobject/super-object.git"
        gitCred = "Rbxxb7pBtyw6D_KqbNWa"

        // github - dohyun
        // gitUrl = "github.com/dohyunKim12/SASNewStruct.git"
        // gitCred = "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"

        // gitBranch = "${params.gitBranch}"
        gitBranch = "master"
        // version = "${gitBranch.tokenize('-')[1]}"
        version = "${params.version}"
        prev_version = "${params.prev_version}"
        dockerRegistry = "192.168.9.12:5000"
        publishUrl = "http://192.168.9.12:8081/repository/maven-releases"
        repoUser = "root"
        repoPassword = "tmax@23"
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
                    if ("${gitUrl.tokenize('/')[0]}" == 'github.com') {
                        credId = 'dohyun_github'
                    } else {
                        credId = 'gitlab_token'
                    }
                    git([branch: "${gitBranch}", credentialsId:"${credId}", url: "http://${gitCred}@${gitUrl}"])
                }
            }
        }
        stage('Git Tagging') {
            steps {
                sh 'git fetch --all'
               script {
                   if ("${gitBranch}" == 'master') {
                       echo "****************************************This is master!*********************************"

                        if ("${prev_version}" == "default") {
                            // prev_version = version.tokenize('.')[0] + "." + version.tokenize('.')[1] + "." + (version.tokenize('.')[2].toInteger() -1.toInteger())
                            prev_version = sh(script:"sudo git describe --tags --abbrev=0", returnStdout: true).tokenize('-')[1]
                            echo "${prev_version}"
                        }

                       sh "git checkout -b release-${version}"
                       commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                       commitId = commitId.substring(1)
                       tagName = "release-${version}"
                       sh "git tag -a ${tagName} -m 'Version ${version} update'"
                   } else {
                       echo "****************************************${gitBranch}!***********************************"
                       commitId = sh(returnStdout: true, script: "git log | head -1 | cut -b 7-15")
                       commitId = commitId.substring(1)
                       tagName = "release-${version}"
                       sh "git tag -a ${tagName} -m 'Version ${version} update'"
                   }
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
       stage('Upload Jar') {
           steps {
               sh "./gradlew publish -PbuildVersion=${version} -PpublishUrl=${publishUrl} -PrepoUser=${repoUser} -PrepoPassword=${repoPassword}"
           }
       }
       stage('Build Package & Upload to ftp server') {
           steps {
                sh "sudo sh ./scripts/packaging.sh"
           }
       }
       stage ('Build and Upload Docker Image') {
           steps {
               script {
                   def dockerImage = docker.build("${dockerRegistry}/super-app-server:${version}", "--build-arg version=${version} .")
                   docker.withRegistry('', 'dockercred') {
                       dockerImage.push()
                   }
                //    sh "docker rmi ${dockerRegistry}/super-app-server:${version}"
               }
           }
       }
       stage('Edit ChangeLog') {
            steps {
                script {

                    def gitDomain = "${gitUrl}".tokenize('/')[0]
                    def changelogString = gitChangelog returnType: 'STRING',
                           from: [type: 'REF', value: "tags/release-${prev_version}"],
                            to: [type: 'REF', value: "tags/release-${version}"],
                            template:
"""
  {{#tags}}
# {{name}}
 {{#issues}}
 
     {{#ifContainsType commits type='feat'}}
## Features

    {{#commits}}
      {{#ifCommitType . type='feat'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}* 
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
  
     {{#ifContainsType commits type='mod'}}
## Refactor

    {{#commits}}
      {{#ifCommitType . type='mod'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}*
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
  
     {{#ifContainsType commits type='fix'}}
## Bug Fixes

    {{#commits}}
      {{#ifCommitType . type='fix'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}*
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
  
     {{#ifContainsType commits type='etc'}}
## OTHERS

    {{#commits}}
      {{#ifCommitType . type='etc'}}
**{{#eachCommitScope .}} {{.}} {{/eachCommitScope}}{{{commitDescription .}}}**  ([{{hash}}](http://${gitDomain}/{{ownerName}}/{{repoName}}/commit/{{hash}})) *{{authorName}} {{commitTime}}*

{{#messageBodyItems}}
  *{{.}}* 
{{/messageBodyItems}}

      {{/ifCommitType}}
    {{/commits}}
  {{/ifContainsType}} 
 {{/issues}}
{{/tags}}
"""
                    writeFile file: "tmp/CHANGELOG_new", text: changelogString
                    currentBuild.description = changelogString
                    sh "mv CHANGELOG.md tmp/tmpfile"
                    sh "cat tmp/CHANGELOG_new > CHANGELOG.md"
                    sh "cat tmp/tmpfile >> CHANGELOG.md"
                    sh "ls -al"
                }
            }
        }
//         stage('Send Email') {
//             steps {
//                 emailext (
//                         attachmentsPattern: 'CHANGELOG.md',
//                         subject: "[super-app-server] Release Notes - super-app-server:${version}",
//                         body:
//                                 """
//  안녕하세요. ck1-2팀 김도현입니다.

// 금주 배포된 super-app-server:${version} release 버전에 대한 안내 및 가이드 메일 드립니다.

// ${version}의 개선 및 추가된 사항은 첨부된 CHANGELOG.md 파일을 확인 부탁드립니다.

// ===

// 안내드린 SAS Runtime Web Server에 DB table create ddl문과 k8s Manifest yaml파일을 포함하여 배포하였습니다.
// 참고해주시길 부탁드립니다.

// ===

// ※ SuperApp 서비스 예제 프로젝트:
// http://gitlab.ck:10081/superobject/super-app-service-example
// 해당 프로젝트를 참조하여 AbstractServiceObject 를 상속받아 슈퍼앱 서비스를 구현하고,
// super-app-runtime.jar 런타임을 실행시키면 테스트가 가능합니다.

// 구체적인 설치 및 서비스 개발, 그리고 테스트 가이드에 대한 내용은 해당 WIKI 가이드 참고 부탁드립니다.
// http://gitlab.ck:10081/superobject/super-object/wikis/home

// SuperApp Server 관련된 문의사항 있으실 경우 메일 혹은 WAPL TF를 통해 문의해주시면 바로 대응하도록 하겠습니다.

// 감사합니다.


// - 김도현 드림.


// ※ SuperApp Server Runtime :
// http://192.168.9.12/binary/super-app-runtime/super-app-runtime-${version}

// ※ SuperApp Server Maven Repository :
// http://192.168.9.12:8081/#browse/browse:maven-releases:com%2Ftmax%2Fsuper-app-server%2F0.0.5%2Fsuper-app-server-${version}.jar

// ※ SuperApp Server Project :
// http://gitlab.ck:10081/superobject/super-object/tree/release-${version}

// ※ gitlab.ck:10081 접속 방법 :
// Default DNS 192.168.1.150 로 설정

// """,
//                         to: "dohyun_kim5@tmax.co.kr; ck1@tmax.co.kr; ck2@tmax.co.kr; cqa1@tmax.co.kr;",
//                         // to: "dohyun_kim5@tmax.co.kr;",
//                         from: "dohyun_kim5@tmax.co.kr"
//                 )
//             }
//         }
//         stage('Git Push') {
//             steps {
//                 echo "pushing..."
//                 script {
//                     commitMsg = "Release commit - version ${version}"
//                     sh "git add -A"
//                     sh "git commit -m \"${commitMsg}\" || true"
//                     sh "git remote rm origin"
//                     sh "git remote add origin http://dohyun_kim5:ehgus0303!@${gitUrl}"
//                     sh "git remote -v"
//                     sh "git push origin refs/tags/${tagName}:refs/tags/${tagName}"
//                     sh "git push origin refs/heads/${gitBranch}:refs/heads/${gitBranch}"
//                 }
//             }
//         }
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