pipeline {
    options {
        // set a timeout of 60 minutes for this pipeline
        timeout(time: 60, unit: 'MINUTES')
    }
    agent any 
        tools {maven "Maven"}

    environment {
        //TODO: Customize these variables for your environment
		DEV_PROJECT = "project-dev"
        STAGE_PROJECT = "project-stage"
        APP_GIT_URL = "https://github.com/Omar-Openshift/java1"
        NEXUS_SERVER = "https://nexus-project-devops.dte-ocp46-wkmkvn-915b3b336cabec458a7c7ec2aa7c625f-0000.us-east.containers.appdomain.cloud/repository/maven-public"

        // DO NOT CHANGE THE GLOBAL VARS BELOW THIS LINE
        APP_NAME = "movies"
    }


    stages {

        stage('Compilation Check') {
            steps {
                echo '### Checking for compile errors ###'
                sh '''
                        mvn -s settings.xml -B clean compile
                   '''
            }
        }

        /*stage('Run Unit Tests') {
            steps {
                echo '### Running unit tests ###'
                sh '''
                        mvn -s settings.xml -B clean test
                   '''
            }
        }*/
		stage ('SonarQube Analysis') {
			steps {
				  withSonarQubeEnv('sonar') {
					 sh 'mvn -U clean install sonar:sonar'
				  }
				}
		}

        stage('Static Code Analysis') {
            steps {
                echo '### Running pmd on code ###'
                sh '''
                        mvn -s settings.xml -B clean pmd:check
                   '''
            }
        }
		
		


        stage('Create fat JAR') {
            steps {
                echo '### Creating fat JAR ###'
                sh '''
                        mvn -s settings.xml -B clean package -DskipTests=true
                   '''
            }
        }

        stage('Launch new app in DEV env') {
            steps {
                echo '### Cleaning existing resources in DEV env ###'
                sh '''
                        oc delete all -l app=${APP_NAME} -n ${DEV_PROJECT}
                        oc delete all -l build=${APP_NAME} -n ${DEV_PROJECT}
                        sleep 5
                        oc new-build java:8 --name=${APP_NAME} --binary=true -n ${DEV_PROJECT}
                   '''

                echo '### Creating a new app in DEV env ###'
                script {
                    openshift.withCluster() {
                      openshift.withProject(env.DEV_PROJECT) {
                        openshift.selector("bc", "${APP_NAME}").startBuild("--from-file=target/${APP_NAME}.jar", "--wait=true", "--follow=true")
                      }
                    }
                }
                sh '''
					oc new-app --as-deployment-config ${APP_NAME}:latest -n ${DEV_PROJECT}
					oc expose svc/${APP_NAME} -n ${DEV_PROJECT}
				'''
            }
        }

        stage('Wait for deployment in DEV env') {
            steps {
                script {
                    openshift.withCluster() {
                        openshift.withProject( "${DEV_PROJECT}" ) {
                            openshift.selector("dc", "${APP_NAME}").related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                    }
                }
            }
        }

        stage('Promote to Staging Env') {
            steps {
                timeout(time: 60, unit: 'MINUTES') {
                    input message: "Promote to Staging?"
                }
                script {
                    openshift.withCluster() {
						openshift.tag("${DEV_PROJECT}/${APP_NAME}:latest", "${STAGE_PROJECT}/${APP_NAME}:stage")
                    }
                }
            }
        }

        stage('Deploy to Staging Env') {
            steps {
                echo '### Cleaning existing resources in Staging ###'
                sh '''
                        oc project ${STAGE_PROJECT}
                        oc delete all -l app=${APP_NAME}
                        sleep 5
                   '''

                echo '### Creating a new app in Staging ###'
                sh '''
					oc new-app --as-deployment-config \
					--name ${APP_NAME} \
					-i ${APP_NAME}:stage \
					-n ${STAGE_PROJECT}
					
					oc expose svc/${APP_NAME}
				'''
            }
        }

        stage('Wait for deployment in Staging') {
            steps {
                sh "oc get route ${APP_NAME} -n ${STAGE_PROJECT} -o jsonpath='{ .spec.host }' --loglevel=4 > routehost"

                script {
                    routeHost = readFile('routehost').trim()

                    openshift.withCluster() {
                        openshift.withProject( "${STAGE_PROJECT}" ) {
                            def deployment = openshift.selector("dc", "${APP_NAME}").rollout()
                            openshift.selector("dc", "${APP_NAME}").related('pods').untilEach(1) {
                                return (it.object().status.phase == "Running")
                            }
                        }
                        echo "Deployment to Staging env is complete. Access the API endpoint at the URL http://${routeHost}/movies."
                    }
                }
            }
        }
    }
}
