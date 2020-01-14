#!groovy


def label = "jenkins"


podTemplate(label: label, yaml: """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-slave
  namespace: jenkins
  labels:
    component: ci
    jenkins: jenkins-agent
spec:
  serviceAccountName: jenkins
  volumes:
  - name: dind-storage
    emptyDir: {}
  containers:
  - name: git
    image: alpine/git
    command:
    - cat
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  - name: helm
    image: lachlanevenson/k8s-helm:v2.16.1
    command:
    - cat
    tty: true
"""
  )

        {

            node(label) {

                stage('Clone config repo') {
                    checkout scm
//                    def values = readYaml(file: 'values.yaml')
//                    println "tag from yaml: ${values.image.tag}"
//                    tagDockerImageFromFile = "${values.image.tag}"
                }

                stage('Clone another repo master') {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: "${branchName}"]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                }

//                    if (isChangeSet()) {
//                        def list = changeSetList()
//                        def yamlFile = list[-1]
//                        def values = readYaml file: yamlFile
//                        println "tag from yaml: ${values.image.tag}"
//                        checkout([$class           : 'GitSCM',
//                                  branches         : [[name: "${values.image.tag}"]],
//                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
//                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
//                    }
//                    if (isBuildingTag()) {
//                        checkout([$class           : 'GitSCM',
//                                  branches         : [[name: "${params.TAG}"]],
//                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
//                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
//                    } else {
//                        checkout([$class           : 'GitSCM',
//                                  branches         : [[name: "${params.COMMIT}"]],
//                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
//                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
//                    }
//

                stage('Deploy DEV release') {
                    if (isMaster()) {
                        container('helm') {
                            withKubeConfig([credentialsId: 'kubeconfig']) {
                                sh """
                         helm upgrade --install app-dev --debug --force ./App/app \
                            --namespace=dev \
                            --set image.tag="${params.COMMIT}" \
                            --values ./dev/values.yaml
                          """
                            }
                        }
                    }
                }
                stage('Deploy QA release') {
                    if (isBuildingTag()) {
                        container('helm') {
                            withKubeConfig([credentialsId: 'kubeconfig']) {
                                sh """
                         helm upgrade --install app-qa --debug --force ./App/app \
                            --namespace=qa \
                            --set image.tag="${params.TAG}" \
                            --values ./qa/values.yaml
                          """
                            }
                        }
                    }
                }
                if (Set.size() != 0) {
                    stage('Deploy PROD release') {
                        if (isChangeSet()) {
                            def list = changeSetList()
                            def yamlFile = Set[-1]
                            def dir = yamlFile.tokenize('/')
                            def stage = dir[0]
                            def appName = yamlFile.removeExtension()
                            container('helm') {
                                withKubeConfig([credentialsId: 'kubeconfig']) {
                                    sh """
                            helm upgrade --install $appName --namespace=$stage --debug --force ./App/app --values $yamlfile
                            """
                                }
                            }
                        }
                    }
                }
            }
        }
                def deploy() {
                    if (isBuildingTag()) {
                        namespace = "qa"
                    }
                    if (isChangeSet()) {
                        nemespace = "prod"
                    }
                    if (isMaster()) {
                        namespace = "dev"
                    }
                }

                boolean isMaster() {
                    return ("${params.TAG}" == "master" )
                }
                boolean isBuildingTag() {
                    return ("${params.TAG}" ==~ /^v\d+.\d+.\d+$/ || "${params.TAG}" ==~ /^\d+.\d+.\d+$/ )
                }

                def changeSetList () {
                    def list = []
                    currentBuild.changeSets.each { changeSet ->
                        changeSet.items.each { entry ->
                            entry.affectedFiles.each { file ->
                                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/*.yaml$/) {
                                    list.toSet()
                                }
                            }
                        }
                 
                    }
                        return Set
                }
                def checkout(branchName) {
                    def list = changeSetList()
                    def yamlFile = Set[-1]
                    def values = readYaml(file: yamlFile)
                    if (isMaster()) {
                        branchName = "${params.COMMIT}"
                    }
                    if (isBuildingTag()) {
                        branchName = "${params.TAG}"
                    }
                    if (changeSetList()) {
                        branchName = "${values.image.tag}"
                    }
                }


