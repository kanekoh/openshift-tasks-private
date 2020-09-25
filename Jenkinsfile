// Set your project Prefix using your GUID
def prefix      = "user27"

// Set variable globally to be available in all stages
// Set Maven command to always include Nexus Settings
def mvnCmd      = "mvn -s ./nexus_openshift_settings.xml"
// Set Development and Production Project Names
def devProject  = "${prefix}-tasks-dev"
def prodProject = "${prefix}-tasks-prod"
// Set the tag for the development image: version + build number
def devTag      = "0.0-0"
// Set the tag for the production image: version
def prodTag     = "0.0"
def destApp     = "tasks-green"
def activeApp   = ""

pipeline {
  agent {
    // Using the Jenkins Agent Pod that we defined earlier
    label "maven-appdev"
  }
  stages {
    // Checkout Source Code and calculate Version Numbers and Tags
    stage('Checkout Source') {
      steps {
        // TBD: Get code from protected Git repository
        withCredentials([usernamePassword(credentialsId: 'gogs-protected', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]){
            git  url: "http://${GIT_USERNAME}:${GIT_PASSWORD}@gogs-gogs.${prefix}-gogs.svc.cluster.local:3000/CICDLabs/openshift-tasks-private.git"
        }

       script {
          def pom = readMavenPom file: 'pom.xml'
          def version = pom.version

          // TBD: Set the tag for the development image: version + build number.
          // Example: def devTag  = "0.0-0"
          devTag = "${version}-${BUILD_NUMBER}"

          // TBD: Set the tag for the production image: version
          // Example: def prodTag = "0.0"
          prodTag = "${version}"

        }
      }
    }

    // Using Maven build the war file
    // Do not run tests in this step
    stage('Build War File') {
      steps {
        echo "Building version ${devTag}"

        // TBD
        sh "${mvnCmd} clean package -DskipTests=true"
      }
    }

    // Using Maven run the unit tests
    stage('Unit Tests') {
      steps {
        echo "Running Unit Tests"

        // TBD
        sh "${mvnCmd} test"
      }
    }

    //Using Maven call SonarQube for Code Analysis
    stage('Code Analysis') {
      steps {
        echo "Running Code Analysis"

        // TBD
        // def SONARQUBE_ROUTE = sh (
            // script: "oc get route sonarqube -n ${GUID}-sonarqube --template='{{ .spec.host }}'",
            // returnStdout: true).trim()
        sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube.${prefix}-sonarqube.svc.cluster.local:9000 -Dsonar.projectName=${JOB_BASE_NAME} -Dsonar.projectVersion=${devTag}"

      }
    }

    // Publish the built war file to Nexus
    stage('Publish to Nexus') {
      steps {
        echo "Publish to Nexus"

        // TBD
        sh "${mvnCmd} deploy -DskipTests=true -DaltDeploymentRepository=nexus::default::http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases"
      }
    }

    // Build the OpenShift Image in OpenShift and tag it.
    stage('Build and Tag OpenShift Image') {
      steps {
        echo "Building OpenShift container image tasks:${devTag}"
        script {
            openshift.withCluster(){
                openshift.withProject("${devProject}"){
    
                    // TBD: Start binary build in OpenShift using the file we just published.
                    // Either use the file from your
                    // workspace (filename to pass into the binary build
                    // is openshift-tasks.war in the 'target' directory of
                    // your current Jenkins workspace).
                    // OR use the file you just published into Nexus:
                    // "--from-file=http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war"
                    def bc = openshift.selector("bc", "tasks")
                    
                    def buildSelector = bc.startBuild("--from-file=http://nexus.${prefix}-nexus.svc.cluster.local:8081/repository/releases/org/jboss/quickstarts/eap/tasks/${prodTag}/tasks-${prodTag}.war", "--wait")
                    buildSelector.logs()
                   
                    #if ( buildSelector.status.phase != "Complete" ) {
                    #    currentBuild.result = 'ABORTED'
		    #	error('Stopping build due to error')
                    #} 
                    //timeout(10) { // wiat for 10 minutes
                    //    buildSelector.untilEach(1){
                    //        return it.object().status.phase == "Complete"
                    //    }
                    //}
                    
                    // TBD: Tag the image using the devTag.
                    openshift.tag("${devProject}/tasks:latest", "${devProject}/tasks:${devTag}")
                }
            }
        }
      }
    }

    // Deploy the built image to the Development Environment.
    stage('Deploy to Dev') {
      steps {
        echo "Deploy container image to Development Project"
        script {
            openshift.withCluster(){
                openshift.withProject("${devProject}"){
                    // TBD: Deploy the image
                    // 1. Update the image on the dev deployment config
                    // 2. Recreate the config maps with the potentially changed properties files
                    // 3. Redeploy the dev deployment
                    // 4. Wait until the deployment is running
                    //    The following code will accomplish that by
                    //    comparing the requested replicas
                    //    (rc.spec.replicas) with the running replicas
                    //    (rc.status.readyReplicas)
                    openshift.set("image", "dc/tasks", "tasks=image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${devTag}")
                    openshift.selector("cm","tasks-config").delete()
                    openshift.create("cm", "tasks-config",
                        '--from-file="./configuration/application-users.properties"',
                        '--from-file="./configuration/application-roles.properties"')

                    def dc = openshift.selector("dc", "tasks")
                    dc.rollout().latest()
                    dc.rollout().status()

                }
            }
        }
      }
    }

    // Run Integration Tests in the Development Environment.
    stage('Integration Tests') {
      steps {
        echo "Running Integration Tests"
        script {
          def status = "000"

          // Create a new task called "integration_test_1"
          echo "Creating task"
          // The next bit works - but only after the application
          // has been deployed successfully
           status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'redhat:redhat1' -H 'Content-Length: 0' -X POST http://tasks.${devProject}.svc.cluster.local:8080/ws/tasks/integration_test_1").trim()
           echo "Status: " + status
           if (status != "201") {
               error 'Integration Create Test Failed!'
           }

          sleep 5
          echo "Retrieving tasks"
          // TBD: Implement check to retrieve the task
           status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'redhat:redhat1' -H 'Content-Length: 0' -X GET http://tasks.${devProject}.svc.cluster.local:8080/ws/tasks/1").trim()
           echo "Status: " + status
           if (status != "200") {
               error 'Integration Create Test Failed!'
           }

          sleep 5
          echo "Deleting tasks"
          // TBD: Implement check to delete the task
           status = sh(returnStdout: true, script: "curl -sw '%{response_code}' -o /dev/null -u 'redhat:redhat1' -H 'Content-Length: 0' -X DELETE http://tasks.${devProject}.svc.cluster.local:8080/ws/tasks/1").trim()
           echo "Status: " + status
           if (status != "204") {
               error 'Integration Create Test Failed!'
           }

        }
      }
    }

    // Copy Image to Nexus Container Registry
    stage('Copy Image to Nexus Container Registry') {
      steps {
        echo "Copy image to Nexus Container Registry"
        // TBD. Use skopeo to copy
        withCredentials([
                usernamePassword(credentialsId: 'nexus', passwordVariable: 'NEXUS_PASSWORD', usernameVariable: 'NEXUS_USERNAME'),
                usernamePassword(credentialsId: 'openshift', passwordVariable: 'OCP_PASSWORD', usernameVariable: 'OCP_USERNAME'),
            ]){
            sh "skopeo copy --src-creds ${OCP_USERNAME}:${OCP_PASSWORD} --src-tls-verify=false --dest-creds ${NEXUS_USERNAME}:${NEXUS_PASSWORD} --dest-tls-verify=false docker://image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${devTag} docker://nexus-registry.${prefix}-nexus.svc.cluster.local:5000/tasks:${prodTag}"
        }

        // TBD: Tag the built image with the production tag.
        script {
            openshift.withCluster(){
                openshift.withProject("${devProject}"){
                    openshift.tag("${devProject}/tasks:${devTag}","${devProject}/tasks:${prodTag}")
                }
            }
        }
      }
    }

    // Blue/Green Deployment into Production
    // -------------------------------------
    // Do not activate the new version yet.
    stage('Blue/Green Production Deployment') {
      steps {
        echo "Blue/Green Deployment"

        // TBD: 1. Determine which application is active
        //      2. Update the image for the other application
        //      3. Deploy into the other application
        //      4. Recreate Config maps for other application
        //      5. Wait until application is running
        //         See above for example code
        script {
            openshift.withCluster(){
                openshift.withProject("${prodProject}"){
                    activeApp = openshift.selector("route", "tasks").object().spec.to.name
                    if( activeApp == "${destApp}"){
                        destApp = "tasks-blue"
                    }
                    
                    echo "Current Active service is ${activeApp}"
                    openshift.set("image", "dc/${destApp}", "${destApp}=image-registry.openshift-image-registry.svc:5000/${devProject}/tasks:${prodTag}")
                    openshift.selector("cm","${destApp}-config").delete()
                    openshift.create("cm", "${destApp}-config",
                        '--from-file="./configuration/application-users.properties"',
                        '--from-file="./configuration/application-roles.properties"')

                    def dc = openshift.selector("dc", "${destApp}")
                    dc.rollout().latest()
                    dc.rollout().status()
                }
            }
        }

      }
    }

    stage('Switch over to new Version') {
      steps {
        // TBD: Stop for approval
        timeout(time: 30, unit: 'DAYS') {
                input message: "Ready to switch over to new Version ?"
            }

        echo "Executing production switch"
        // TBD: After approval execute the switch
        script {
            openshift.withCluster(){
                openshift.withProject("${prodProject}"){
                    openshift.set("route-backends", "tasks", "${destApp}=100", "${activeApp}=0")
                }
            }
        }
      }
    }
  }
}

