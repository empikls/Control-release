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
  - name: nodejs
    image: node:latest
    command:
    - cat
    tty: true
  - name: kubectl
    image: bitnami/kubectl:latest
    command:
    - cat
    tty: true
  - name: docker
    image: docker:19.03.3-git
    command:
    - cat
    tty: true
    env:
    - name: DOCKER_HOST
      value: tcp://docker-dind:2375
    volumeMounts:
    - mountPath: /var/lib/docker
      name: dind-storage 
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
                    def values = readYaml(file: 'values.yaml')
                    println "tag from yaml: ${values.image.tag}"
                    tagDockerImageFromFile = "${values.image.tag}"
                }

                stage('Clone another repo master') {
                    def values = readYaml(file: 'values.yaml')
                    if (isChangeSet()) {
                        checkout([$class           : 'GitSCM',
                                  branches         : [[name: "${tagDockerImageFromFile}"]],
                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    }
                    else {
                        checkout([$class           : 'GitSCM',
                                  branches         : [[name: "${params.COMMIT}"]],
                                  extensions       : [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'App']],
                                  userRemoteConfigs: [[url: "https://github.com/empikls/node.is"]]])
                    }
                }
                echo "${params.TAG}"
                echo "${params.COMMIT}"


                stage('Deploy DEV release') {
                    sh """
                    shortCommit = "${echo "${params.COMMIT}" | cut -c1-7}"
                       """
                    if (isMaster()) {
                        nameStage = "app-dev"
                        namespace = "dev"
                        tagDockerImage = "${shortCommit}"
                        hostname = "dev-173-193-112-65.nip.io"
                        deploy(nameStage, namespace, tagDockerImage, hostname)
                    }
                }
                stage('Deploy QA release') {
                    if (isBuildingTag()) {
                        nameStage = "app-qa"
                        namespace = "qa"
                        tagDockerImage = "${params.TAG}"
                        hostname = "qa-173-193-112-65.nip.io"
                        deploy(nameStage, namespace, tagDockerImage, hostname)
                    }
                }
                stage('Deploy PROD release') {
                    if (isChangeSet()) {
                        container('helm') {
                            withKubeConfig([credentialsId: 'kubeconfig']) {
                                sh """
                            helm upgrade --install prod --debug ./App/app --values ./values.yaml
                            """
                            }
                        }
                    }
                }
            }
        }


                def tagDockerImage
                def nameStage
                def hostname


                boolean isMaster() {
                    return ("${params.TAG}" == "master" )
                }
                boolean isBuildingTag() {
                    return ("${params.TAG}" ==~ /^v\d.\d.\d$/ || "${params.TAG}" ==~ /^\d.\d.\d$/ )
                }

                boolean isChangeSet() {

//                currentBuild.changeSets.any { changeSet ->
//                    changeSet.items.any { entry ->
//                        entry.affectedFiles.any { file ->
//                            if (file.path.equals("values.yaml")) {
//                                return true
//                                }
//                            }
//                        }
//                    }
//                }
                    def changeLogSets = currentBuild.changeSets
                    for (int i = 0; i < changeLogSets.size(); i++) {
                        def entries = changeLogSets[i].items
                        for (int j = 0; j < entries.length; j++) {
                            def files = new ArrayList(entries[j].affectedFiles)
                            for (int k = 0; k < files.size(); k++) {
                                def file = files[k]
                                if (file.path.equals("values.yaml")) {
                                    return true
                                }
                            }
                        }
                    }
                }
                def deploy( appName, namespace, tagName, hostName ) {
                    container('helm') {
                        echo "Release image: $shortCommit"
                        echo "Deploy app name: $appName"
                        withKubeConfig([credentialsId: 'kubeconfig']) {
                            sh """
                         helm upgrade --install $appName --debug --force ./App/app \
                            --namespace=$namespace \
                            --set image.tag="$tagName" \
                            --set ingress.hostName=$hostName \
                            --set-string ingress.tls[0].hosts[0]="$hostName" \
                            --set-string ingress.tls[0].secretName=acme-$appName-tls
                          """
                        }
                    }
              }