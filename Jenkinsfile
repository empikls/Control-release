#!groovy


def label = "jenkins"

properties([
        parameters([
                string(name: 'tagFromJob1', defaultValue: '', description: 'Short commit ID or Tag from upstream job', )
        ])
])

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
                def branchName = params.tagFromJob1
                def list = ischangeSetList()
                def values
                if (!ischangeSetList()) {
                    stage('Checkout App repo') {
                        checkoutConfRepo(branchName)
                    }
                }
                if (isMaster()) {
                    stage('Deploy DEV release') {
                        deploy("./dev/values.yaml", "app-dev", "dev", branchName)
                    }
                }
                if (isBuildingTag()) {
                    stage('Deploy QA release') {
                        deploy("./qa/values.yaml", "app-qa", "qa", branchName)
                    }
                }
                if (list) {
                    list.each { item ->

                        stage('Checkout App repo for item') {
                            values = readYaml file: item
                            branchName = values.image.tag
                            checkoutConfRepo(branchName)
                        }


                        stage('Deploy PROD release for $nameSpace') {
                            def appName = item.split('/')[1].split(/\./)[0]
                            def nameSpace = item.split('/')[0]
                            deploy(item, appName, nameSpace, values.image.tag)
                        }
                    }
                }
            }
        }
                def checkoutConfRepo(branchName) {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: branchName]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: branchName]],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    }
                def deploy(confValues, appName, nameSpace, branchName ) {
                    container('helm') {
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh """
                               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./$branchName/app --values $confValues \
                               --set image.tag=$branchName
                            """
                        }
                    }
                }

                boolean isMaster() {
                    return ("${params.tagFromJob1}" ==~  "/[a-z0-9]{7}/" )
                }
                boolean isBuildingTag() {
                    return ("${params.tagFromJob1}" ==~ /^v\d+.\d+.\d+$/ || "${params.tagFromJob1}" ==~ /^\d+.\d+.\d+$/ )
                }

                def ischangeSetList() {
                    def list = []
                    currentBuild.changeSets.each { changeSet ->
                        changeSet.items.each { entry ->
                            entry.affectedFiles.each { file ->
                                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/\w+.yaml$/) {
                                  list.add(file.path)
                                }
                            }
                        }
                    }
                        return list.toSet()
                }

