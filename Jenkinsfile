pipeline {
    agent { label 'e2e-pipeline' }
    environment {
        GIT_CREDENTIALS_ID = '488de81c-89ef-4c4a-be5a-79ef832e6fa3' // Your Git credentials ID
        SSH_CREDENTIALS_ID = 'ssh-user9' // Your SSH credentials ID for deployment
        MAINFRAME_HOST = '192.86.32.87' // Replace with your server's user and address
        MAINFRAME_PORT = '65522'
        DBB_PROJECT_DIR = '/u/user9/devops/dbb-zappbuild'
        venvDir = '/global/opt/pyenv/gdp'
        buildDir = '/u/user9/FullBuild/BUILD-OUTPUT'
        buildOutputDir = "${buildDir}/build.*"
    }

    stages {
        stage('Connect to Mainframe') {
            steps {
                script {
                        sshagent(credentials: ['ssh-user9']) { 
                            echo "Using SSH credentials: ssh-user9"
                            echo "Connecting to Mainframe Host at ${MAINFRAME_HOST}:${MAINFRAME_PORT}..."
                            
                            // Attempt SSH connection with verbose logging
                            try {
                                sh """
                                    ssh -v -p ${MAINFRAME_PORT} -o StrictHostKeyChecking=no user@${MAINFRAME_HOST} 'echo "SSH connection established."'
                                """
                            } catch (Exception e) {
                                echo "SSH connection failed: ${e}"
                                error("Exiting due to SSH connection failure.")
                            }
                        }
            }
                }
            }
        stage('Prepare SSH Key') {
            steps {
                script {
                    echo 'Preparing SSH key for Git...'
                    withCredentials([usernamePassword(credentialsId: GIT_CREDENTIALS_ID, passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            /usr/lpp/IBM/dbb/bin/git-jenkins.sh
                            echo "SSH Key prepared. Listing SSH keys..."
                        """
                    }
                }
            }
        }

        stage('Fetch Repo') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        echo 'Starting checkout from Git repository...'
                        sh """
                            cd ${DBB_PROJECT_DIR}
                            echo "Fetching the repository..."
                            git fetch --all
                            git checkout ${params.BRANCH_NAME} || exit 0 // Checkout branch if it exists
                        """
                        echo "Successfully fetched."
                    }
                }
            }
        }

        stage('DBB Build') {
            steps {
                script {
                    dir(DBB_PROJECT_DIR) {
                        echo "Starting DBB build..."
                        sh """
                        rm -rf /u/user9/FullBuild/BUILD-OUTPUT; mkdir -p /u/user9/FullBuild/BUILD-OUTPUT;
                        git pull --all
                        $DBB_HOME/bin/groovyz /u/user9/devops/dbb-zappbuild/build.groovy --sourceDir /u/user9/devops/dbb-zappbuild --workDir ${buildDir} --hlq USER9.TRNG --application samples/MortgageApplication --verbose --fullBuild
                        """
                    }
                }
            }
        }       
        
        stage('Wazi deploy') {
            steps {
                script {
                    
                    // Extract the directory and base name
                    def outputDir = sh(script: "ls -d ${buildOutputDir} | sort | tail -n 1", returnStdout: true).trim()
                    echo "Latest build output: ${outputDir}"
                    
                    sh """
                            set -e
                            
                            # Create Python virtual environment
                            . ${venvDir}/bin/activate
                            
                            # Run the wazideploy-package command
                            
                            $DBB_HOME/bin/groovyz /u/user9/dbb/Pipeline/PackageBuildOutputs/PackageBuildOutputs.groovy --workDir ${outputDir} -ae 
                       """     
                            // Create the full path for the .tar file
                            def baseName = outputDir.substring(outputDir.lastIndexOf('/') + 1) // Gets just the last part of the path
                            echo "base name: ${baseName}"
                            def tarFilePath = "${outputDir}/${baseName}.tar"
                            echo "Tar file path: ${tarFilePath}"
                            
                    // Run the wazideploy-generate command
                    sh """ 
                            set -e
                            
                            # Create Python virtual environment
                            . ${venvDir}/bin/activate
                    
                            wazideploy-generate \
                                -dm /u/user9/Scripts/deployment-method-static.yml \
                                -dp /u/user9/FullBuild/BUILD-OUTPUT/deployment-plan.yml \
                                -dpr /u/user9/FullBuild/BUILD-OUTPUT/deployment-plan-report.html \
                                -pif ${tarFilePath} \
                                -pof /u/user9/FullBuild/BUILD-OUTPUT/Mortgage.1.1.tar || { echo 'Error occurred while running wazideploy-generate'; exit 1; }
                        """
                    // Run the wazideploy-deploy command
                    sh """ 
                            set -e
                            
                            # Create Python virtual environment
                            . ${venvDir}/bin/activate
                            # Setting environment variables
                            export _BPXK_AUTOCVT=ON
                            export ZOAU_HOME=/usr/lpp/IBM/zoautil
                            export PATH=\${ZOAU_HOME}/bin:\$PATH
                            export LIBPATH=\${ZOAU_HOME}/lib:\$LIBPATH
                            export PATH=/VERSYSB/usr/lpp/IBM/cyp/v3r11/pyz/bin:\$PATH
                            export LIBPATH=/VERSYSB/usr/lpp/IBM/cyp/v3r11/pyz/lib:\$LIBPATH
                            export _CEE_RUNOPTS='FILETAG(AUTOCVT,AUTOTAG) POSIX(ON)'
                            export _TAG_REDIR_ERR=txt
                            export _TAG_REDIR_IN=txt
                            export _TAG_REDIR_OUT=txt
                            wazideploy-deploy \
                                -dp /u/user9/FullBuild/BUILD-OUTPUT/deployment-plan.yml \
                                -pif /u/user9/FullBuild/BUILD-OUTPUT/Mortgage.1.1.tar \
                                -ef /u/user9/Scripts/environment-zos.yml \
                                -efn /u/user9/FullBuild/BUILD-OUTPUT/evidences/evidence.yml \
                                -wf ${buildDir} || { echo 'Error occurred while running wazideploy-deploy'; exit 1; }
                        """
                }
            }
        }



     
}

   

    post {
        success {
            echo 'Pipeline successful.'
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
