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
) {


                node(label) {
                    def list = ischangeSetList()
                    def map = [
                            'dev'     : {['values': '', 'tag': '']},
                            'qa'      : {['values': '', 'tag': '']},
                            'prod-ap1': {['values': '', 'tag': '']},
                            'prod-eu1': {['values': '', 'tag': '']},
                            'prod-us1': {['values': '', 'tag': '']},
                            'prod-us2': {['values': '', 'tag': '']}
                    ]
                    stage('Clone config repo') {
                        checkout scm
                        echo "tag from Job1 : ${params.tagFromJob1}"
                    }
                    if (isMaster()) {
                        map['dev'] = {['values': 'dev/values.yaml', 'tag': params.tagFromJob1]}
                    }
                    if (isBuildingTag()) {
                        map['qa'] = {['values': 'qa/values.yaml', 'tag': params.tagFromJob1]}
                    }

                    if (list) {
                        list.each { item ->
                            def nameSpace = item.split('/')[0]
                            def values = readYaml file: item
                            println nameSpace
                            map[nameSpace] = {['values': item, 'tag': values.image.tag]}
                        }
                    }
                    map.each {
                        stage(it.value) {
                            println it.key
//                                if(it.key ==
                            deployStage(it.key)
                        }
                    }
                }
            }
//            /dev/values.yaml
//            /prod-eu1/values.yaml
def deployStage(list) {
    list.each {
        def nameSpace = it.values.split('/')[0]
        def appName = it.values.split('/')[1].split(/\./)[0]
        println appName
        checkoutConfRepo(it['tag'])
        deploy(it['values'], appName, nameSpace, it['tag'])
    }
}
def checkoutConfRepo(branchName) {
    checkout([$class           : 'GitSCM',
              branches         : [[name: branchName]],
              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: branchName]],
              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
    }
def deploy( confValues, appName, nameSpace, branchName ) {
    container('helm') {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh """
               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./$branchName/app --values $confValues  \
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
                if (file.path ==~ /^prod-(ap1|eu1|us1|us2)\/\w+.yaml$/ ) {
                  list.add(file.path)
                }
            }
        }
    }
        return list.toSet()
}

