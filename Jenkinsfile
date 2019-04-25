pipeline {
  agent {
    label "jenkins-go"
  }
  environment {
    ORG = 'kuflink'
    APP_NAME = 'demo-app'
    FRONTEND_APP_NAME = 'frontend-app'
    APP_DIR = '/home/jenkins/go/src/github.com/kuflink/demo-app'
    CHARTMUSEUM_CREDS = credentials('jenkins-x-chartmuseum')
  }
  stages {
    stage('CI Build and push snapshot') {
      when {
        branch 'PR-*'
      }
      environment {
        PREVIEW_VERSION = "0.0.0-$BRANCH_NAME-$BUILD_NUMBER"
        PREVIEW_NAMESPACE = "$FRONTEND_APP_NAME-$BRANCH_NAME".toLowerCase()
        HELM_RELEASE = "$PREVIEW_NAMESPACE".toLowerCase()

      }
      steps {
        container('go') {
          // @todo - make PHP container
          // @todo - make nginx container
          // @todo - make api/admin ? container

          // Build the GO app
          dir(env.APP_DIR) {
            // @todo - find out what checkout scm does
            checkout scm
            // Make the GO app on linux distry
            sh "make linux"
            // Do a skaffold (docker) build
            // @todo - withEnv() to put these vars in the shell env
            sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold_frontend.yaml"
            // Post build image - assuming we push here
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:$PREVIEW_VERSION"
          }
          // Make a preview environemnt
          dir("${env.APP_DIR}/charts/preview") {
            sh "make preview_frontend"
            sh "jx preview --app $APP_NAME --dir ../.."
          }
        }

        // @todo - make this container template inside of jenkins
        // container('php_72') {
        container('go') {
          // @todo - make PHP container
          // @todo - make nginx container
          // @todo - make api/admin ? container

          dir('/home/jenkins/go/src/github.com/kuflink/demo-app') {
            // @todo - find out what checkout scm does in this specific context (dir)
            checkout scm

            // @todo - withEnv() to put these vars in the shell env
            // Do a skaffold (docker) build
            // @todo - figure out how skaffold's "docker: {}" section can be customised to use a different Dockerfile path. (for frontendapp)

            // @todo - composer install here - re-using the image
            // Make the GO app on linux distry
            sh "make linux"

            sh "export VERSION=$PREVIEW_VERSION && skaffold build -f skaffold_frontend.yaml"

            // Post build image - assuming we push here
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$FRONTEND_APP_NAME:$PREVIEW_VERSION"
          }

          // Make a preview environemnt
          dir("${env.APP_DIR}/charts/preview") {
            sh "make preview_frontend"
            sh "jx get apps"
            sh "jx preview --app $FRONTEND_APP_NAME --dir ../.."
          }
        }

      }
    }
    stage('Build Release') {
      when {
        branch 'develop'
      }
      steps {
        container('go') {
          dir(env.APP_DIR) {
            checkout scm

            // ensure we're not on a detached head
            sh "git checkout develop"
            sh "git config --global credential.helper store"
            sh "jx step git credentials"

            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "jx step tag --version \$(cat VERSION)"
            sh "make build"
            sh "export VERSION=`cat VERSION` && skaffold build -f skaffold.yaml"
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$APP_NAME:\$(cat VERSION)"
          }
        }

        container('php_72') {
          dir(env.APP_DIR) {
            checkout scm

            // ensure we're not on a detached head
            sh "git checkout develop"
            sh "git config --global credential.helper store"
            sh "jx step git credentials"

            // so we can retrieve the version in later steps
            sh "echo \$(jx-release-version) > VERSION"
            sh "jx step tag --version \$(cat VERSION)"

            // @todo - composer install here - re-using the image

            sh "export VERSION=`cat VERSION` && skaffold build -f skaffold_frontend.yaml"
            sh "jx step post build --image $DOCKER_REGISTRY/$ORG/$FRONTEND_APP_NAME:\$(cat VERSION)"
          }
        }

      }
    }
    stage('Promote to Environments') {
      when {
        branch 'develop'
      }
      steps {
        container('go') {
          dir("${env.APP_DIR}/charts/demo-app") {
            sh "jx step changelog --version v\$(cat ../../VERSION)"

            // release the helm chart
            sh "jx step helm release"

            // promote through all 'Auto' promotion Environments
            sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
          }
        }

        container('php_72') {
          dir("${env.APP_DIR}/charts/frontend_app") {
            sh "jx step changelog --version v\$(cat ../../VERSION)"

            // release the helm chart
            sh "jx step helm release"

            // promote through all 'Auto' promotion Environments
            sh "jx promote -b --all-auto --timeout 1h --version \$(cat ../../VERSION)"
          }
        }

      }
    }
  }
}
