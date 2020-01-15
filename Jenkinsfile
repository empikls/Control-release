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
                    echo "tag from Job1 : ${params.tagFromJob1}"
                }

//                stage('Clone another repo master') {
//                    checkout([$class           : 'GitSCM',
//                              branches         : [[name: $branchName]],
//                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
//                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
//                }
                def branchName = params.tagFromJob1
                if (ischangeSetList()) {
                    branchName = "${values.image.tag}"
                    }
                stage('Checkout App repo') {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: branchName]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                }
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

                stage('Deploy PROD release') {
                    if (ischangeSetList()) {
                        def list = changeSetList()
                        def yamlFile = list[-1]
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

                boolean isMaster() {
                    return ("${params.TAG}" == "master" )
                }
                boolean isBuildingTag() {
                    return ("${params.TAG}" ==~ /^v\d+.\d+.\d+$/ || "${params.TAG}" ==~ /^\d+.\d+.\d+$/ )
                }

                def ischangeSetList () {
                    def list
                    currentBuild.changeSets.each { changeSet ->
                        changeSet.items.each { entry ->
                            entry.affectedFiles.each { file ->
                                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/values.yaml$/) {
                                    list.add(file.path)
                                }
                            }
                        }
                    }
                    return list
                }
                def checkoutAppRepo(branchName) {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: $branchName]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                }


