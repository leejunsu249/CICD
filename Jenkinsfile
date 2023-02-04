def branch = 'master'
def commitMsg = 'CICD-pipeline'
def githubEmail = '75758585@naver.com'
def githubKey = 'github-key'
def githubURL = 'https://github.com/leejunsu249/CICD.git'
def imageTag = 'red'
def registry = '10.60.200.120:5000'
def target_host = "root@10.60.200.120"

podTemplate(label: 'docker-build',
  containers: [
    containerTemplate(
      name: 'podman',
      image: '10.60.200.120:5000/c-podman',
      command: 'cat',
      ttyEnabled: true,
      privileged: true
    ),
    containerTemplate(
      name: 'argo',
      image: 'argoproj/argo-cd-ci-builder:latest',
      command: 'cat',
      ttyEnabled: true
    ),
  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/containerd/containerd.sock', hostPath: '/var/run/containerd/containerd.sock'),
  ]
) 
{
  node('docker-build') {
    stage('Checkout') {
      container('podman') {
        checkout scm
      }
    }

    stage('Podman Build') {
      dir(path: 'container') {
        container(name:'podman', shell:'/bin/bash') {
             sh """
                #!/bin/bash

                # Construct Image Name
                IMAGE=${registry}/test:${imageTag}

                podman build -t \${IMAGE} --build-arg COLOR=${imageTag} .
                """
        }
      }
    }

    stage('Push image') {
      container(name:'podman', shell:'/bin/bash') {
        withCredentials([usernamePassword(credentialsId: podmankey, usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]){
                sh """
                    #!/bin/bash

                    # Construct Image Name
                    IMAGE=${registry}/test:${imageTag}

                    podman login -u ${USERNAME} -p ${PASSWORD} ${registry} --tls-verify=false

                    podman push \${IMAGE} --tls-verify=false
                    """
        }
      }
    }

    stage('Image Scan') {
      container(name: 'podman', shell:'/bin/bash') {
          
          sh """
             #!/bin/bash

             IMAGE=${registry}/test:${imageTag}

             trivy image --severity HIGH,CRITICAL --insecure=true --format json \${IMAGE}
             """
      }
    }

    stage('Sign Image') {
      sshagent (credentials: ['ssh-agent']) {
                sh """
                    ssh -o StrictHostKeyChecking=no ${target_host}
                    IMAGE=${registry}/test:${imageTag}

                    "cosign sign --insecure-skip-verify --allow-insecure-registry  --key k8s://image-sign/cosignkey \${IMAGE}

                """
      }
    }

    stage('Deploy') {
      container('argo') {
        checkout scm
        withCredentials([usernamePassword(credentialsId: github, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]){
                sh """
                    #!/bin/bash

                    git config --global user.email ${githubEmail}
                    git config --global user.name leejunsu249
                    git checkout ${branch}
                    cd  helm-charts
                    sed -i 's/tag:.*/tag: ${imageTag}/g' values.yaml
                    git commit -a -m ${commitMsg}
                    git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/leejunsu249/CICD.git HEAD:master
                    """
        }
      }
    }
  }
}

