def IMAGE_NAME = "sample-webpack"
def REPOSITORY_URL = "git@github.com:nalbam/sample-webpack.git"
def REPOSITORY_SECRET = "nalbam-secret"
def SLACK_TOKEN = ""

@Library("github.com/opsnow-tools/valve-butler")
def butler = new com.opsnow.valve.Butler()
def label = "worker-${UUID.randomUUID().toString()}"
def VERSION = ""
def SOURCE_LANG = ""
def SOURCE_ROOT = ""
properties([
  buildDiscarder(logRotator(daysToKeepStr: "60", numToKeepStr: "30"))
])
podTemplate(label: label, containers: [
  containerTemplate(name: "builder", image: "quay.io/opsnow-tools/valve-builder", command: "cat", ttyEnabled: true, alwaysPullImage: true),
  containerTemplate(name: "node", image: "node:10", command: "cat", ttyEnabled: true)
], volumes: [
  hostPathVolume(mountPath: "/var/run/docker.sock", hostPath: "/var/run/docker.sock"),
  hostPathVolume(mountPath: "/home/jenkins/.draft", hostPath: "/home/jenkins/.draft"),
  hostPathVolume(mountPath: "/home/jenkins/.helm", hostPath: "/home/jenkins/.helm")
]) {
  node(label) {
    stage("Prepare") {
      container("builder") {
        butler.prepare()

        if (!SLACK_TOKEN) {
          SLACK_TOKEN = butler.slack_token
        }
      }
    }
    stage("Checkout") {
      container("builder") {
        try {
          if (REPOSITORY_SECRET) {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME, credentialsId: REPOSITORY_SECRET)
          } else {
            git(url: REPOSITORY_URL, branch: BRANCH_NAME)
          }
        } catch (e) {
          failure(SLACK_TOKEN, "Checkout", IMAGE_NAME)
          throw e
        }

        butler.scan(IMAGE_NAME, BRANCH_NAME, "nodejs")

        VERSION = butler.version
        // SOURCE_LANG = butler.source_lang
        // SOURCE_ROOT = butler.source_root
      }
    }
    stage("Build") {
      container("node") {
        try {
          butler.npm_build()
          success(SLACK_TOKEN, "Build", IMAGE_NAME, VERSION)
        } catch (e) {
          failure(SLACK_TOKEN, "Build", IMAGE_NAME)
          throw e
        }
      }
    }
    if (BRANCH_NAME == "master") {
      stage("Build Image") {
        parallel(
          "Build Docker": {
            container("builder") {
              try {
                butler.build_image(IMAGE_NAME, VERSION)
              } catch (e) {
                failure(SLACK_TOKEN, "Build Docker", IMAGE_NAME)
                throw e
              }
            }
          },
          "Build Charts": {
            container("builder") {
              try {
                butler.build_chart(IMAGE_NAME, VERSION)
              } catch (e) {
                failure(SLACK_TOKEN, "Build Charts", IMAGE_NAME)
                throw e
              }
            }
          }
        )
      }
      stage("Deploy DEV") {
        container("builder") {
          try {
            butler.helm_install(IMAGE_NAME, VERSION, "dev", "dev.opsnow.com", "dev")
            success(SLACK_TOKEN, "Deploy DEV", IMAGE_NAME, VERSION, "dev", "dev.opsnow.com")
          } catch (e) {
            failure(SLACK_TOKEN, "Deploy DEV", IMAGE_NAME)
            throw e
          }
        }
      }
      stage("Proceed STAGE") {
        container("builder") {
          proceed(SLACK_TOKEN, "Deploy STAGE", IMAGE_NAME, VERSION, "stage")
          timeout(time: 60, unit: "MINUTES") {
            input(message: "$IMAGE_NAME $VERSION to stage")
          }
        }
      }
      stage("Deploy STAGE") {
        container("builder") {
          try {
            butler.helm_install(IMAGE_NAME, VERSION, "stage", "dev.opsnow.com", "dev")
            success(SLACK_TOKEN, "Deploy STAGE", IMAGE_NAME, VERSION, "stage", "dev.opsnow.com")
          } catch (e) {
            failure(SLACK_TOKEN, "Deploy STAGE", IMAGE_NAME)
            throw e
          }
        }
      }
    }
  }
}
def failure(token = "", type = "", name = "") {
  slack("$token", "danger", "$type Failure", "`$name`", "$JOB_NAME <$RUN_DISPLAY_URL|#$BUILD_NUMBER>")
}
def success(token = "", type = "", name = "", version = "", namespace = "", base_domain = "", cluster = "") {
  if (cluster) {
    def link = "https://$name-$namespace.$base_domain"
    slack("$token", "good", "$type Success", "`$name` `$version` :satellite: `$namespace` :earth_asia: `$cluster`", "$JOB_NAME <$RUN_DISPLAY_URL|#$BUILD_NUMBER> : <$link|$name-$namespace>")
  } else if (base_domain) {
    def link = "https://$name-$namespace.$base_domain"
    slack("$token", "good", "$type Success", "`$name` `$version` :satellite: `$namespace`", "$JOB_NAME <$RUN_DISPLAY_URL|#$BUILD_NUMBER> : <$link|$name-$namespace>")
  } else if (namespace) {
    slack("$token", "good", "$type Success", "`$name` `$version` :rocket: `$namespace`", "$JOB_NAME <$RUN_DISPLAY_URL|#$BUILD_NUMBER>")
  } else {
    slack("$token", "good", "$type Success", "`$name` `$version` :heavy_check_mark:", "$JOB_NAME <$RUN_DISPLAY_URL|#$BUILD_NUMBER>")
  }
}
def proceed(token = "", type = "", name = "", version = "", namespace = "") {
  slack("$token", "warning", "$type Proceed?", "`$name` `$version` :rocket: `$namespace`", "$JOB_NAME <$RUN_DISPLAY_URL|#$BUILD_NUMBER>")
}
def slack(token = "", color = "", title = "", message = "", footer = "") {
  try {
    // butler.slack("$token", "$color", "$title", "$message", "$footer")
    sh """
      curl -sL repo.opsnow.io/valve-ctl/slack | bash -s -- --token='$token' \
        --footer='$footer' --footer_icon='https://jenkins.io/sites/default/files/jenkins_favicon.ico' \
        --color='$color' --title='$title' '$message'
    """
  } catch (ignored) {
  }
}
