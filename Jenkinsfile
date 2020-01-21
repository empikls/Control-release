#!groovy

import org.jenkinsci.plugins.pipeline.modeldefinition.Utils

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
                    println "$map.devRelease.tag"
                }
                def branchName = params.tagFromJob1
                def list = ischangeSetList()
                def values
                def map = [
                        devRelease    : [values: './dev/values.yaml', tag: 'params.tagFromJob1'],
                        qaRelease     : [values: './qa/values.yaml', tag: 'params.tagFromJob1'],
                        prodAp1Release: [values: '', tag: ''],
                        prodEu1Release: [values: '', tag: ''],
                        prodUs1Release: [values: '', tag: ''],
                        prodUs2Release: [values: '', tag: '']
                ]
                if (isMaster() || isBuildingTag()) {
                    stage('Checkout App repo') {
                        checkoutConfRepo(branchName)

                    }
                }
                stage('Deploy Dev release') {
                    if (isMaster()) {
                        confValues = list.add("./dev/values.yaml")
                        nameSpace = confValues.split('/')[1]
                        checkoutConfRepo(branchName)
                        appName = confValues.split('/')[2].split('.')[0]
                        deploy(confValues, appName, nameSpace, branchName)
                    }
                }
                stage('Deploy QA release') {
                    if (isBuildingTag()) {
                        confValues = list.add("./qa/values.yaml")
                        nameSpace = confValues.split('/')[1]
                        appName = confValues.split('/')[2].split('.')[0]
                        checkoutConfRepo(branchName)
                        deploy(confValues, appName, nameSpace, branchName)
                    }
                }
                if (list) {
                    list.each { item ->
                        stage('Deploy release for ' + item.split('/')[0]) {
                            def appName = item.split('/')[1].split(/\./)[0]
                            def nameSpace = item.split('/')[0]
                            values = readYaml file: item
                            branchName = values.image.tag
                            checkoutConfRepo(branchName)
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
    return ("${params.tagFromJob1}" ==~  /[a-z0-9]{7}/ )
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

