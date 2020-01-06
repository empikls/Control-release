#!groovy

def label = "jenkins"
env.DOCKERHUB_IMAGE = "devops53/hello-world"

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
    
    stage('Checkout SCM') {
        checkout scm
        sh 'git rev-parse HEAD > GIT_COMMIT'
        shortCommit = readFile('GIT_COMMIT').take(7)
    }
    
   
  
      
      
    stage('Deploy to production') {
      container('helm') {
          echo "Deploy app name: app-prod"
        withKubeConfig([credentialsId: 'kubeconfig']) {
          sh """
         helm upgrade --install app-prod --debug --force https://github.com/empikls/node.is/tree/master/app \
            --namespace="prod" \
            --set image.tag="app-prod" \
            --set ingress.hostName="prod-184-173-46-252.nip.io" \
            --set-string ingress.tls[0].hosts[0]="prod-184-173-46-252.nip.io" \
            --set-string ingress.tls[0].secretName=acme-prod-tls 
          """
        }
      }
    }
