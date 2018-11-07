#!groovy

node('maven') {

  openshift.withCluster(){
    openshift.withProject("${MYPROJECT}"){
      echo "Using project ${openshift.project()}"

      stage('Deploy nexus server'){
        def APP_NAME="nexus"
        if(openshift.selector('dc', "${APP_NAME}").exists()){
          echo "${APP_NAME} already has been deployed."
        } else {
          def models = openshift.process("openshift//nexus3")
          def created = openshift.apply(models)
          def dc = created.narrow('dc')


          dc.related('pods').untilEach{
            return it.object().status.phase == 'Running'
          }

          def latestDeploymentVersion = openshift.selector('dc',"${APP_NAME}").object().status.latestVersion
          def rc = openshift.selector('rc', "${APP_NAME}-${latestDeploymentVersion}")
          rc.untilEach(1){
            def rcMap = it.object()
            return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
          }
        }
      }

      // Create Environment for DM
      stage('Create DM environment') {
        def APP_NAME="myapp-kieserver"
        if(openshift.selector('dc', "${APP_NAME}").exists()){
          echo "${APP_NAME} already has been deployed"
        }else{
          echo "Create the environment using a pre-defined template"

          def models = openshift.process("openshift//rhdm71-trial-ephemeral")
          def created = openshift.apply(models)

          // 複数のDeployment Configが同時に生成された場合の監視
          def dc = openshift.selector('dc', "${APP_NAME}")
          dc.related('pods').untilEach{
            return it.object().status.phase == 'Running'
          }


          def latestDeploymentVersion = openshift.selector('dc',"${APP_NAME}").object().status.latestVersion
          def rc = openshift.selector('rc', "${APP_NAME}-${latestDeploymentVersion}")
          rc.untilEach(1){
            def rcMap = it.object()
            return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
          }
        }
      }

      stage('Build a rule engine'){
        echo "Build rule engine"

        git url: 'ssh://git@gitlab.consulting.redhat.com:2222/hkaneko/kihonhensaiyoryoku.git', credentialsId: "git-cert"
        sh '''
         mvn -Dsample.kie.host=myapp-kieserver-sample9.apps.3f32.example.opentlc.com \
          -Dsample.kie.contextpath="/" \
          -Dsample.kie.username=adminUser \
          -Dsample.kie.password=RedHat \
          kieserver:deploy
        '''
      }


      def APP_NAME="s2i-fuse71-eap-camel-amq"
      stage('Deploy camel application with S2I'){
        echo "Deploy a camel app"

        def models = openshift.process("openshift//acom-cicd-example-template")
        def created = openshift.apply(models)

        def dc = openshift.selector('dc', "${APP_NAME}")
        dc.related('pods').untilEach{
          return it.object().status.phase == 'Running'
        }

        def latestDeploymentVersion = openshift.selector('dc',"${APP_NAME}").object().status.latestVersion
        def rc = openshift.selector('rc', "${APP_NAME}-${latestDeploymentVersion}")
        rc.untilEach(1){
          def rcMap = it.object()
          echo "${rcMap}"
          return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
        }
      }

        // Run Integration Tests in the Development Environment.
      stage('Integration Tests') {
        echo "Running Integration Tests"
        // TBD
      }

      // Copy Image to Nexus Docker Registry
      stage('Copy Image to Docker Registry') {
        echo "Copy image to Docker Registry"
        openshift.tag("${MYPROJECT}/${APP_NAME}:latest", "${DEVPROJECT}/${APP_NAME}:latest")
      }
    }

    def DEV_APP_NAME="s2i-fuse71-eap-camel-amq"
    openshift.withProject("${DEVPROJECT}"){
      // Blue/Green Deployment into Production
      // -------------------------------------
      // Do not activate the new version yet.
      def tag   = "blue"
      def altTag = "green"

      stage('Initialize'){
        // ROUTE_NAME_DEVPROJECT
        def app_route = openshift.selector("route/${DEV_APP_NAME}").object()

        def svc = app_route.spec.to.name

        if(svc == "${DEV_APP_NAME}-blue"){
            tag = "green"
            altTag = "blue"
        }

        echo "Current Active service is $svc"
      }


      stage('Blue/Green Production Deployment') {
        echo "Blue/Green Production application to ${tag}."
        def dc = openshift.selector("dc", "${DEV_APP_NAME}-${tag}")

        dc.rollout().latest()
        dc.related('pods').untilEach(1){
          return (it.object().status.phase == "Running")
        }

        def latestDeploymentVersion = dc.object().status.latestVersion
        def rc = openshift.selector('rc', "${DEV_APP_NAME}-${tag}-${latestDeploymentVersion}")
        rc.untilEach(1){
          def rcMap = it.object()
          echo "${rcMap}"
          return (rcMap.status.replicas.equals(rcMap.status.readyReplicas))
        }

      }

      stage('Deploy rule'){
        // TBD
        echo "Deploy rule into dev environment"
      }

      stage('Switch over to new Version') {
        // 注意事項 OpenShiftのユーザ名がそのままJenkinsのユーザ名ではない。
        // Jenkinsにログインした上でユーザプロファイルを確認すること
        input(
          message: 'Do you allow to deploy the latest image to the development environment?',
          ok: 'Yes',
          submitter: "user01-admin" 
        )

        echo "Switching Production application to ${tag}."
        openshift.set("route-backends", "${DEV_APP_NAME}", "${DEV_APP_NAME}-${tag}=100", "${DEV_APP_NAME}-${altTag}=0")}
    }
  }
}

// Convenience Functions to read variables from the pom.xml
// Do not change anything below this line.
// --------------------------------------------------------
def getVersionFromPom(pom) {
  def matcher = readFile(pom) =~ '<version>(.+)</version>'
  matcher ? matcher[0][1] : null
}
def getGroupIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<groupId>(.+)</groupId>'
  matcher ? matcher[0][1] : null
}
def getArtifactIdFromPom(pom) {
  def matcher = readFile(pom) =~ '<artifactId>(.+)</artifactId>'
  matcher ? matcher[0][1] : null
}

