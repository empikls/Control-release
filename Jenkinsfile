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
                    def tag =  params.tagFromJob1
                    def list = ischangeSetList()
                    def map = [
                            'dev'     : ['values': '','tag':''],
                            'qa'      : ['values': '','tag':''],
                            'prod-ap1': ['values': '','tag':''],
                            'prod-eu1': ['values': '','tag':''],
                            'prod-us1': ['values': '','tag':''],
                            'prod-us2': ['values': '','tag':'']
                    ]
                    stage('Clone config repo') {
                        checkout scm
                        echo "tag from Job1 : ${params.tagFromJob1}"
                    }
                    if (isMaster()) {
                        map['dev'] = ['values': 'dev/values.yaml','tag':params.tagFromJob1]
                    }
                    if (isBuildingTag()) {
                        map['qa'] = ['values': 'qa/values.yaml','tag':params.tagFromJob1]
                    }
                    if (list) {
                        list.each { item ->
                            def nameSpace = item.split('/')[0]
                            def appName = item.split('/')[1].split(/\./)[0]
                            dockerTag = readYaml file: item
                            tag = item.value.tag
                            map[nameSpace] = ['values': item,'tag':tag]
                        }
                    }
                    map.each {
                        stage("Deploy release for " + it.key) {
//                            it.key = 'dev' , 'qa', 'prod-ap1','prod-eu1','prod-us1','prod-us2'

                            deployStage(it.value.values, it.value.tag)
//                            it.value = 'values':'dev/values.yaml','tag':sometag'
//                              it.value.values = 'dev/values.yaml'
//                                it.value.tag = 'sometag'
                        }
                    }
                }
            }
def deployStage(file, tag) {
        file.each { item ->
            def nameSpace = item.split('/')[0]
            def appName = item.split('/')[1].split(/\./)[0]
            checkoutConfRepo(tag)
            deploy(nameSpace, appName, item, tag)
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

