#!groovy
import groovy.text.SimpleTemplateEngine
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
                'dev'     : ['values': ''],
                'qa'      : ['values': ''],
                'prod-ap1': ['values': ''],
                'prod-eu1': ['values': ''],
                'prod-us1': ['values': ''],
                'prod-us2': ['values': '']
        ]
        stage('Clone config repo') {
            checkout scm
            echo "tag from Job1 : ${params.tagFromJob1}"
        }
        if (isMaster()) {
            map['dev'] = ['values': 'dev/values.yaml']
        }
        if (isBuildingTag()) {
            map['qa'] = ['values': 'qa/values.yaml']
        }
        if (list) {
            list.each { item ->
                def nameSpace = item.split('/')[0]
                def appName = item.split('/')[1].split(/\./)[0]
                map[nameSpace] = ['values': item]
            }
        }
        map.each {
            stage("Deploy release for " + it.key) {
//                            it.key = 'dev' , 'qa', 'prod-ap1','prod-eu1','prod-us1','prod-us2'

                deployStage(it.value.values)
//                            it.value = 'values':'dev/values.yaml','values':'qa/values.yaml','values':'prod-ap1/*.yaml','values':'prod-eu1/*.yaml','values':'prod-us1/*.yaml','values':'prod-us2/*.yaml'
//                              it.value.values = 'dev/values.yaml','qa/values.yaml','prod-ap1/*.yaml','prod-eu1/*.yaml','prod-us1/*.yaml','prod-us2/*.yaml'
            }
        }
    }
}
def deployStage(list) {
    def tag = params.tagFromJob1
    list.each { item ->
        if (isMaster() || isBuildingTag()) {
            tag = params.tagFromJob1
        }
        else {
            dockerTag = readTaml file: item
            tag = item.value.tag
        }
        def nameSpace = item.split('/')[0]
        def appName = item.split('/')[1].split(/\./)[0]
        checkoutConfRepo(tag)
        deploy(nameSpace, appName, item ,tag)
    }
}
def checkoutConfRepo(tag) {
    checkout([$class           : 'GitSCM',
              branches         : [[name: tag]],
              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: tag]],
              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
}
def deploy( nameSpace, appName, confValues, tag ) {
    container('helm') {
        withKubeConfig([credentialsId: 'kubeconfig']) {
            sh """
               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./$tag/app --values $confValues  \
               --set image.tag=$tag
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
