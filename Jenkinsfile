pipeline {
    agent {
        label {
            label ""
            // JOB_NAME will be "<github_repo_name>/<branch>", e.g acme-project/master
            customWorkspace "/tmp/jenkins-workspace_${JOB_NAME.replace('/', '_')}"
        }
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()

        preserveStashes(buildCount: 20)
        buildDiscarder(
                logRotator(numToKeepStr: '10', daysToKeepStr: '14'))

        // Do not create too many builds based on frequent SCM changes; set as a number of seconds
        quietPeriod(3 * 3600)

        retry(2)    // overcome some random issues in the build
    }

    environment {
        // Each value must be an interpolated String, cannot use variable directly inside 'environment'

        TARGET_WEDEPLOY_REMOTE = "us-west-1.liferay.cloud"
        TARGET_WEDEPLOY_PROJECT_ROOT = "liferaygssdlc"

        TARGET_WEDEPLOY_PROJECT_DEV = "liferaygssdlc-dev"             // The WD project where the main dev branch ("master") should be deployed
        DEVELOPMENT_BRANCH_NAME = "develop"                        // The GitHub branch where currently the main development happens


        // Credentials set up using 'dxpcloud-jenkins' Docker image -> init.groovy.d/dxpcloud-4-setup-secrets.groovy
        // vars like *_USR and *_PSW are also exposed, e.g. LIFERAY_COM_CREDS_USR and LIFERAY_COM_CREDS_PSW,
        // see https://jenkins.io/doc/book/pipeline/jenkinsfile/#usernames-and-passwords
        LIFERAY_COM_CREDS = credentials('liferay-com')
        WEDEPLOY_CREDS = credentials('wedeploy')

        // A file in workspace where the create build's UID (server-side build number) will be recorded
        WEDEPLOY_BUILD_NUMBER_FILE = 'build/wedeploy_buildGroupUid.txt'
    }

    stages {
        stage('Build Workspace') {
            steps {
                echo "Building services using Gradle..."

                lock('execution-gradle') {
                    sh './gradlew distWeDeploy --no-daemon --stacktrace'
                }

                // Make sure secrets written in the files are not recorded as artifacts, by
                // adding e.g.:
                //      excludes: '**/download-patches-username.txt,**/download-patches-password.txt'
                archiveArtifacts(
                        artifacts: 'build/wedeploy/**')

                // But they are stashed, since they are used in Dockerfile for WD build
                stash(
                        name: 'artifacts',
                        includes: 'build/wedeploy/**')
            }
        }

        stage('Create DXP Cloud Build') {
            steps {
                unstash('artifacts')

                echo "Pushing built services into: ${env.TARGET_WEDEPLOY_REMOTE} / ${env.TARGET_WEDEPLOY_PROJECT_ROOT}"

                // Create a new build using WD CLI
                lock('execution-we') {
                    sh '''\
                            #!/bin/bash
                            
                            set -euxo pipefail
                        
                            # cannot 'we update' as regular user, so just show the version
                            we version
                        
                            # The path must match where your workspace produces the files, using './gradlew distWeDeploy'
                            cd build/wedeploy
                        
                            we remote set $TARGET_WEDEPLOY_REMOTE $TARGET_WEDEPLOY_REMOTE
                            we remote default $TARGET_WEDEPLOY_REMOTE
                            we logout
                            echo $WEDEPLOY_CREDS_USR $WEDEPLOY_CREDS_PSW | we login
                        
                            # Deploy into project itself (no environment), since we have an 'universal' build
                            we deploy --only-build --quiet --project $TARGET_WEDEPLOY_PROJECT_ROOT
                                                            
                            we logout
                        '''.stripIndent().trim()
                }

                // Capture the ID of the newly created build for eventual deploy (see below)
                sh '''\
                        #!/bin/bash
                        
                        set -euxo pipefail
                        
                        API_URL="https://api.$TARGET_WEDEPLOY_REMOTE"
                        
                        # get the number of the latest build, as produced in stage 'Build DXP Cloud' above
                        latest_build_buildGroupUid=$(\
                            curl --silent --show-error --fail \
                                    -u $WEDEPLOY_CREDS_USR:$WEDEPLOY_CREDS_PSW \
                                    "$API_URL/projects/$TARGET_WEDEPLOY_PROJECT_ROOT/builds?limit=1" \
                                | python -c "import sys, json; print(json.load(sys.stdin)[0]['buildGroupUid'])" \
                        )
                        
                        echo $latest_build_buildGroupUid >> $WEDEPLOY_BUILD_NUMBER_FILE
                    '''.stripIndent().trim()

                archiveArtifacts(
                        artifacts: env.WEDEPLOY_BUILD_NUMBER_FILE)

                stash(
                        name: 'wedeploy-metadata',
                        includes: env.WEDEPLOY_BUILD_NUMBER_FILE)

            }
        }

        stage("Deploy to 'dev'") {
            when {
                branch env.DEVELOPMENT_BRANCH_NAME
            }
            steps {
                unstash('wedeploy-metadata')

                echo "Deploying the build ${readFile(env.WEDEPLOY_BUILD_NUMBER_FILE).trim()} " +
                        "to ${env.TARGET_WEDEPLOY_REMOTE} / ${env.TARGET_WEDEPLOY_PROJECT_DEV}"

                sh '''\
                        #!/bin/bash
                        
                        set -euxo pipefail
                        
                        API_URL="https://api.$TARGET_WEDEPLOY_REMOTE"
                        CONSOLE_URL="https://console.$TARGET_WEDEPLOY_REMOTE"
                       
                        buildGroupUid=$(cat $WEDEPLOY_BUILD_NUMBER_FILE)
                       
                        http_code=$(\
                            curl -s -o deploy_out.html -w '%{http_code}' \
                                    -u $WEDEPLOY_CREDS_USR:$WEDEPLOY_CREDS_PSW \
                                    -X POST "$API_URL/projects/$TARGET_WEDEPLOY_PROJECT_DEV/deploy" \
                                    -H "Content-Type: application/json" \
                                    -d '{ "buildGroupUid": "'$buildGroupUid'" }' \
                        )
                        
                        if [ "$http_code" = 200 ]; then
                            echo "Deployment of build $buildGroupUid to '$TARGET_WEDEPLOY_PROJECT_DEV' started, please check the result on $CONSOLE_URL/projects/$TARGET_WEDEPLOY_PROJECT_ROOT/deployments"
                        else
                            echo "Deployment of build $buildGroupUid to '$TARGET_WEDEPLOY_PROJECT_DEV' could not be started, remote returned:"
                            cat deploy_out.html
                            
                            exit 1 
                        fi
                        
                        # TODO implement waiting to be able to fail the pipeline?
                        # Wait for deploy to succeed; fail on error or after timeout
                                                      
                    '''.stripIndent().trim()
            }
        }
    }
}