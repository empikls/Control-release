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

                def values

                    stage('Checkout App repo') {
                        if (isMaster()) {
                            checkoutConfRepo(branchName)
                        }
                        if (isBuildingTag()) {
                            checkoutConfRepo(branchName)
                        }
                    }
                if (isMaster()) {
                    def list
                    stage('Deploy DEV release') {
                        confValues = list.add("./dev/values.yaml")
                        deploy(confValues, "app-dev", "dev", branchName)
                    }
                }
                if (isBuildingTag()) {
                    def list
                    stage('Deploy QA release') {
                        confValues = list.add("./qa/values.yaml")
                        deploy(confValues, "app-qa", "qa", branchName)
                    }
                }
                list.each { item ->
                    if (list) {
                        stage('Checkout App repo') {
                            values = readYaml(file: item)
                            branchName = values.image.tag
                            checkoutConfRepo(branchName)
                        }
                    }
                    if (list) {
                        stage('Deploy PROD release') {
                            def appName = item.split('/')[1].split( /\./ )[0]
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
                               echo appVersion: $branchName >> ${params.tagFromJob1}/app/Chart.yaml
                                echo "values : $confValues"
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

