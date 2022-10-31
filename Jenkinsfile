pipeline {
    agent any

    // Set project parameters.
    parameters {
        booleanParam(name: 'CLEAN_WS',
                     defaultValue: false,
                     description: 'If enabled the whole workspace will be cleaned before the pipeline starts.')
        string(name: 'PACKAGES',
               defaultValue: 'd3f_app RepoTrend Limex JSONForUG4',
               description: 'Packages to install (use spaces to separate multiple packages).')
        string(name: 'CXX_FLAGS',
               defaultValue: '-fprofile-arcs -ftest-coverage -fPIC',
               description: 'C++ flags for compiling ug4.')
        string(name: 'CC',
               defaultValue: 'ccache clang',
               description: 'C compiler to use.')
        string(name: 'CXX',
               defaultValue: 'ccache clang++',
               description: 'C++ compiler to use.')
        string(name: 'TEST_APPS',
               defaultValue: '',
               description: 'Apps to test (use spaces to separate multiple apps). If empty all installed apps will be searched for XML-test-files and tested.')
        string(name: 'CMAKE_OPTIONS',
               defaultValue: '-DUSE_JSON=ON -DINTERNAL_BOOST=ON -DDEBUG=OFF -DENABLE_ALL_PLUGINS=ON -DUGTest=OFF -DSUPERLU_PATH=/usr/local/SuperLU/4.3 -GNinja',
               description: 'Options to pass to cmake when building ug4.')
        string(name: 'CMAKE_TESTSUITE_OPTIONS',
               defaultValue: '-DUSE_JSON=OFF -DINTERNAL_BOOST=ON -DDEBUG=OFF -GNinja',
               description: 'Options to pass to cmake when building the testsuite.')
        string(name: 'TEST_NAME',
               defaultValue: "${env.JOB_NAME}",
               description: 'This will be the name used in the report file.')

        // TODO Add support for emails
        /*string(name: 'NOTIFY_ON_FAILURE',
               defaultValue: 'tests',
               description: 'Email addresses (use spaces to separate multiple ).')*/

    }

    // Environment variable for all stages.
    environment {
    PATH = "${env.WORKSPACE}/ughub:$PATH"
    }

    stages {
        stage('Clean workspace') {
            when{expression { params.CLEAN_WS }}

            steps {
                echo 'Stage: Clean workspace'
                cleanWs()
            }
        }

        // Install ug4 for testing.
        stage('Install UG4') {
            steps {
                echo 'Stage: Install'

                sh 'git clone https://github.com/UG4/ughub.git || true'
                sh 'mkdir -p $WORKSPACE/ug4'
                dir("$WORKSPACE/ug4") {
                    // Setup ug4 environment.
                    sh 'ughub init || true'
                    sh 'ughub updatesources || true'

                    // Install all packages to test:
                    sh 'ughub install ${PACKAGES}'

                    // Legacy testsuite is required for testing.
                    sh 'ughub install unit_tests'
                    sh 'ughub git submodule init'
                    sh 'ughub git submodule update'
                }
            }
        }

        stage('Init UG4 submodules') {
            steps {
                echo 'Stage: Init submodules'

                dir("$WORKSPACE/ug4") {
                    sh 'ughub git submodule init'
                    sh 'ughub git submodule update'
                }
            }
        }

        // Build ug4 with installed apps.
        stage('Build UG4') {
            // Set environment variables for build.
            environment {
                CXX_FLAGS="${CXX_FLAGS}"
                LD_FLAGS="-lboost_system -lboost_filesystem -lboost_serialization -lug4" //TODO rename to LDFLAGS?
            }

            steps {
                echo 'Stage: Build UG4'

                // Build ug4.
                sh 'rm -rf $WORKSPACE/ug4/build/'
                sh 'mkdir -p $WORKSPACE/ug4/build'
                dir("$WORKSPACE/ug4/build") {
                    sh 'CC=${CC} CXX=${CXX} cmake .. ${CMAKE_OPTIONS}'
                    sh 'ninja -j3'
                }
            }
        }

        stage('Test'){
            environment {
                BUILD_NO="${env.BUILD_NUMBER}"
            }

            stages {
                // Build testsuite.
                stage('Build testsuite') {
                    // Set environment variables for build.
                    environment {
                        CXX_FLAGS="${CXX_FLAGS}"
                        LD_FLAGS="-lboost_system -lboost_filesystem -lboost_serialization -lug4" //TODO rename to LDFLAGS?
                    }

                    steps {
                        echo 'Stage: Test'

                        // Build Testsuite.
                        sh 'rm -rf $WORKSPACE/ug4/build-testsuite/'
                        sh 'mkdir -p $WORKSPACE/ug4/build-testsuite'
                        dir("$WORKSPACE/ug4/build-testsuite") {
                            sh 'CC=${CC} CXX=${CXX} cmake $WORKSPACE/ug4/apps/unit_tests ${CMAKE_OPTIONS}'
                            sh 'ninja -j3'
                        }
                    }
                }

                stage('Run tests'){
                    // Perform tests on ug4 build.
                    steps{
                        sh 'mkdir -p $WORKSPACE/reports/$BUILD_NO/'
                        dir("$WORKSPACE/reports/${env.BUILD_NUMBER}") {
                            // Check if TEST_APPS option is empty and run accordingly.
                            sh '''
                                if [ -z "${TEST_APPS}"]
                                then
                                    command $WORKSPACE/ug4/bin/testsuite --build-info=yes --log_level=all --log_format=XML -- --name ${TEST_NAME}|| true
                                else
                                    command $WORKSPACE/ug4/bin/testsuite --build-info=yes --log_level=all --log_format=XML -- --name ${TEST_NAME} --apps ${TEST_APPS} || true
                                fi
                            '''
                            sh 'mkdir -p $WORKSPACE/reports/current/'
                            sh 'cp ug_test_numprocs_1.log "$WORKSPACE/reports/current/report.xml"'
                            sh 'rm ug_test_numprocs_1.log'
                        }
                    }
                }

                // Analyze report.
                stage('Report') {
                    steps{
                        // Invoke xUnit on reports.
                        xunit([BoostTest(
                                        deleteOutputFiles: true,
                                        failIfNotNew: true,
                                        pattern: "reports/current/report.xml",
                                        skipNoTestFiles: false,
                                        stopProcessingIfError: true
                                        )])
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Stage: Deploy'
            }
        }
    }

    /*post {
        always {
            //cleanWs()
        }
    }*/
}
