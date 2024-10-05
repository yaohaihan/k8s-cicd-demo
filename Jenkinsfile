pipeline {
  agent {
    node {
      label 'maven'
    }

  }
  stages {
    stage('clone code') {
      steps {
        container('maven') {
          git(url: 'http://192.168.110.121:28080/root/demo1.git', credentialsId: 'gitlab-user-pass', branch: '$BRANCH_NAME', changelog: true, poll: false)
        }

      }
    }

    stage('unit test') {
      steps {
        container('maven') {
          sh 'mvn clean test'
        }

      }
    }

    stage('sonarqube analysis') {
      agent none
      steps {
        withCredentials([string(credentialsId : 'sonarqube' ,variable : 'SONAR_TOKEN' ,)]) {
          withSonarQubeEnv('sonar') {
            container('maven') {
              sh '''mvn sonar:sonar -Dsonar.projectKey=$APP_NAME
echo "mvn sonar:sonar -Dsonar.projectKey=$APP_NAME"'''
            }

          }

          timeout(unit: 'MINUTES', activity: true, time: 5) {
            waitForQualityGate 'true'
          }

        }

      }
    }

    stage('build & push') {
      steps {
        container('maven') {
          sh 'mvn clean package -DskipTests'
          sh 'docker build -f Dockerfile -t $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER .'
          withCredentials([usernamePassword(credentialsId : 'harbor-user-pass' ,passwordVariable : 'DOCKER_PASSWORD' ,usernameVariable : 'DOCKER_USERNAME' ,)]) {
            sh '''echo "$DOCKER_PASSWORD" | docker login $REGISTRY -u "$DOCKER_USERNAME" --password-stdin
docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER'''
          }

        }

      }
    }

    stage('push latest') {
      when {
        branch 'master'
      }
      steps {
        container('maven') {
          sh 'docker tag $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
          sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:latest'
        }

      }
    }

    stage('deploy to dev') {
      steps {
        container('maven') {
          input(id: 'deploy-to-dev', message: 'deploy to dev?')
          withCredentials([kubeconfigContent(credentialsId : 'kubeconfig-id' ,variable : 'ADMIN_KUBECONFIG' ,)]) {
            sh 'mkdir -p ~/.kube/'
            sh 'echo "$ADMIN_KUBECONFIG" > ~/.kube/config'
            sh '''sed -i\'\' "s#REGISTRY#$REGISTRY#" deploy/cicd-demo-dev.yaml
sed -i\'\' "s#DOCKERHUB_NAMESPACE#$DOCKERHUB_NAMESPACE#" deploy/cicd-demo-dev.yaml
sed -i\'\' "s#APP_NAME#$APP_NAME#" deploy/cicd-demo-dev.yaml
sed -i\'\' "s#BUILD_NUMBER#$BUILD_NUMBER#" deploy/cicd-demo-dev.yaml
kubectl apply -f deploy/cicd-demo-dev.yaml'''
          }

        }

      }
    }

    stage('push with tag') {
      agent none
      when {
        expression {
          params.TAG_NAME =~ /v.*/
        }

      }
      steps {
        input(message: 'release image with tag?', submitter: '')
        withCredentials([usernamePassword(credentialsId : 'gitlab-user-pass' ,passwordVariable : 'GIT_PASSWORD' ,usernameVariable : 'GIT_USERNAME' ,)]) {
          sh 'git config --global user.email "liugang@wolfcode.cn" '
          sh 'git config --global user.name "xiaoliu" '
          sh 'git tag -a $TAG_NAME -m "$TAG_NAME" '
          sh 'git push http://$GIT_USERNAME:$GIT_PASSWORD@$GIT_REPO_URL/$GIT_ACCOUNT/k8s-cicd-demo.git --tags --ipv4'
        }

        container('maven') {
          sh 'docker tag $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:SNAPSHOT-$BUILD_NUMBER $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME'
          sh 'docker push $REGISTRY/$DOCKERHUB_NAMESPACE/$APP_NAME:$TAG_NAME'
        }

      }
    }

    stage('deploy to production') {
      agent none
      when {
        expression {
          params.TAG_NAME =~ /v.*/
        }

      }
      steps {
        input(message: 'deploy to production?', submitter: '')
        container('maven') {
          sh '''sed -i\'\' "s#REGISTRY#$REGISTRY#" deploy/cicd-demo.yaml
sed -i\'\' "s#DOCKERHUB_NAMESPACE#$DOCKERHUB_NAMESPACE#" deploy/cicd-demo.yaml
sed -i\'\' "s#APP_NAME#$APP_NAME#" deploy/cicd-demo.yaml
sed -i\'\' "s#TAG_NAME#$TAG_NAME#" deploy/cicd-demo.yaml

kubectl apply -f deploy/cicd-demo.yaml'''
        }

      }
    }

  }
  environment {

    DOCKER_CREDENTIAL_ID = 'harbor-user-pass'
    GIT_REPO_URL = '192.168.110.121:28080'
    GIT_CREDENTIAL_ID = 'gitlab-user-pass'
    KUBECONFIG_CREDENTIAL_ID = '3123e385-8143-430f-b38b-674d0c913638'
    REGISTRY = '192.168.110.122:8858'
    DOCKERHUB_NAMESPACE = 'docker_username'
    GITHUB_ACCOUNT = 'root'
    APP_NAME = 'demo1'

  }
  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'master', description: '请选择要发布的分支')
    string(name: 'TAG_NAME', defaultValue: 'snapshot', description: '标签名称，必须以 v 开头，例如：v1、v1.0.0')
  }
}
