#!groovy


def label = "jenkins"

properties([
        parameters([
                string(name: 'tagFromJob1', defaultValue: 'Null', description: 'Short commit ID or Tag from upstream job', )
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
                echo "list is $list "
                    if ( list                    ) {
                        values = readYaml(file: list[0])
                        branchName = values.image.tag
                    }
                    if (isBuildingTag()) {
                        branchName = params.tagFromJob1
                        confValues = list.add("./qa/values.yaml")
                    }
                    if (isMaster()) {
                        branchName = params.tagFromJob1
                        confValues = list.add("./dev/values.yaml")
                    }
                stage('Checkout App repo') {
                    checkout([$class           : 'GitSCM',
                              branches         : [[name: branchName]],
                              extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: branchName]],
                              userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    sh 'ls'
                }
                if (isMaster()) {
                    stage('Deploy DEV release') {
                        confValues = list.add("./dev/values.yaml")
                        appName = "app-dev"
                        nameSpace = "dev"
                        dockerTag = params.tagFromJob1
                        deploy(confValues, appName, nameSpace, dockerTag)
                    }
                }
                if (isBuildingTag()) {
                    stage('Deploy QA release') {
                        confValues = list.add("./qa/values.yaml")
                        appName = "app-qa"
                        nameSpace = "qa"
                        dockerTag = params.tagFromJob1
                        deploy(confValues, appName, nameSpace, dockerTag)
                    }
                }
                if (ischangeSetList()) {
                    stage('Deploy PROD release') {
                        confValues = ischangeSetList()
                        def appName
                        def nameSpace
                        list.each { file ->
                            appName = file.split('/')[1]
                            nameSpace = file.split('/')[0]
                        }
                        dockerTag = values.image.tag
                        deploy(confValues, appName, nameSpace, dockerTag)
                    }
                }
            }
        }
                def deploy(confValues, appName, nameSpace, dockerTag ) {
                    container('helm') {
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh """
                               helm upgrade --install $appName --namespace=$nameSpace --debug --force ./$dockerTag/app --values $confValues \
                               --set image.tag=$dockerTag
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

                def ischangeSetList () {
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

