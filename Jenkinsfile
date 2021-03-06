def handleCheckout = {
  if (env.gitlabMergeRequestId) {
    sh "echo 'Merge request detected. Merging...'"
    checkout ([
      $class: 'GitSCM',
      branches: [[name: "${env.gitlabSourceNamespace}/${env.gitlabSourceBranch}"]],
      extensions: [
        [$class: 'PruneStaleBranch'],
        [$class: 'CleanCheckout'],
        [
          $class: 'PreBuildMerge',
          options: [
            fastForwardMode: 'NO_FF',
            mergeRemote: env.gitlabTargetNamespace,
            mergeTarget: env.gitlabTargetBranch
          ]
        ]
      ],
      userRemoteConfigs: [
                [
          credentialsId: credentialsId,
          name: env.gitlabTargetNamespace,
          url: env.gitlabTargetRepoHttpUrl
        ],
        [
          credentialsId: credentialsId,
          name: env.gitlabSourceNamespace,
          url: env.gitlabSourceRepoHttpUrl
        ]
      ]
    ])
  } else {
    sh "echo 'No merge request detected. Checking out current branch'"
    checkout([$class: 'GitSCM', branches: [[name: Branch]], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[credentialsId: credentialsId, url: Git_Url]]])
  }
}
def deploy (Namespace) { 
    sh '''
    ls -lrth
    echo "Creating namespace ${Namespace} if needed"
    [ ! -z \"\$(kubectl get ns ${Namespace} -o name 2>/dev/null)\" ] || kubectl apply -f namespace.yml
    kubectl apply -f deployment.yml
    kubectl apply -f service.yml
    '''
}
def build_status (state) {
    gitLabConnection('Jenkins')
    updateGitlabCommitStatus(name: 'build', state: "${state}")
}
def buid_pass_git () {
    script{
        if (currentBuild.currentResult == "ABORTED") {
            build_status ('canceled')
        } else if ( currentBuild.currentResult == "FAILURE"){
            build_status ('failed')
            build_status ('success')
        }
    }
}

pipeline {
        agent any
        options {
                buildDiscarder(logRotator(numToKeepStr: '10', artifactNumToKeepStr: '10'))
                preserveStashes()
                disableConcurrentBuilds()
        }
        environment {
            Namespace = 'test-namespace'
            credentialsId = 'eb402e14-8fee-482f-9c81-d97b9ea64481'
            Git_Url = 'http://gitlab/jerry/My_test_webapp.git'
            Branch = '*/ITT'
            Git_Url_Template = 'http://192.168.56.102/jerry/Templates.git'
            Branch_Template = 'master'
            Image_Name = 'jjjje/directory_build_itt_branch'
            branch = '*/Development'
        }
        tools {
                maven 'Local_Maven_3.5.4'
                jdk 'Local_jdk_1.8.0'
        }
        stages {
                stage('Git Checkout') {
                    steps {
                        build_status ('running')
                        deleteDir ()
                        script {
                            handleCheckout ()
                        }
                    }
                }
                stage('Unit testing') {
                    when {
                        branch 'Development'
                    }
                    steps {
                        sh 'mvn clean test'
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/**/*.xml'
                        }
                    }
                }
                stage('Maven build, Unit test, JaCoCo code coverage & Sonar code quality Anlysis') {
                    when {
                        branch 'ITT'
                    }
                    steps { 
                        withSonarQubeEnv('sonarenv') {
                        sh 'mvn package sonar:sonar'
                        }
                        jacoco ()
                    }
                    post {
                        always {
                            junit 'target/surefire-reports/**/*.xml'
                            script {
                                stash includes: 'target/*.jar', name: 'targetfiles'
                            }
                        }
                    }
                }
                stage('Sonar Quality Gate') {
                    when {
                        branch 'ITT'
                    }
                    steps {
                        timeout(time: 1, unit: 'HOURS') {
                        // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                        // true = set pipeline to UNSTABLE, false = don't
                        // Requires SonarQube Scanner for Jenkins 2.7+
                            script {
                                def qg = waitForQualityGate ()
                                if (qg.status != 'OK') {
                                    error "Pipeline aborted due to quality gate failure: ${qg.status}"
                                }
                            }
                        }    
                    }    
                }
                stage("Docker image Creation") {
                    when {
                        branch 'ITT'
                    }
                    steps {
                        git branch: Branch_Template,
                        credentialsId: credentialsId,
                        url: Git_Url_Template
                        dir('openjdk8') {
                            script {
                                unstash 'targetfiles'
                                docker.withTool("local") { 
                                    withDockerRegistry([credentialsId: "Dockerhub", url: ""]) {
                                        def img = docker.build(Image_Name)
                                        img.push()
                                        img.push('${BUILD_NUMBER}')
                                    }
                                }        
                            }
                        }
                    }
                    post {
                        always {
                            sh 'docker rmi -f ${Image_Name} ${Image_Name}:${BUILD_NUMBER}'
                        }
                    }
                }
                stage('deploy') {
                    when {
                        branch 'ITT'
                    }
                    steps {
                        dir('openjdk8') {
                            echo "Deploying"
                            deploy (namespace)
                        }
                    }
                }
                stage('Performance Tests') {
                    when {
                        branch 'ITT'
                    }
                    steps {
                            dir ('openjdk8') {
                                sh 'ls -lrth'
                                sh 'bzt test.yml'
                                perfReport 'TaurusResult/perf_result_csv.xml'
                            }    
                    }
                }
        }
        post {
            always {
                buid_pass_git ()  
            }       
        }
}
