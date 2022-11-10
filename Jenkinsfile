pipeline {
    agent any

    stages {
        stage("Setup") {
            steps {
                echo "Configuring Envronment Post Serverless Deployment"
                script {
                    def response = httpRequest url: "https://owl-flex-store-7157-dev.twil.io/generate-token?username=${HTTP_USERNAME}&password=${HTTP_PASSWORD}", wrapAsMultipart: false
                    println('Status: '+response.status)
                    if(response.status == 200) {
                        def jsonSlurper = new JsonSlurper() 
                        def object = jsonSlurper.parseText(response.content)
                        println('Access Token: '+ object.access_token)
                        env.TWIL_IO_ACCESS_TOKEN = object.access_token
                    }
                    def matcher = manager.getLogMatcher(".*domain.*")
                    def domain
                    if(matcher.matches()) {
                        env.ONE_CLICK_DEPLOY_BASE_URL = matcher.group(0).replaceAll("\\s+", " ").split(" ")[1]
                    }
                    env.TWILIO_WORKSPACE_SID="${TWILIO_WORKSPACE_SID}"
                }
                dir("supervisor-barge-coach-fns") {
                    script {
                        if(fileExists(".twiliodeployinfo")) {
                            def twilioDeployInfo = readJSON(file:".twiliodeployinfo");
                            env.ONE_CLICK_DEPLOY_FNS_SERVICE_SID = twilioDeployInfo[TWILIO_ACCOUNT_SID].serviceSid
                        }
                    }
                }
            }
        }
        
        stage("Serverless") {
            stages {
                stage("Notification") {
                    script {
                        def response = httpRequest customHeaders: [[name: 'Authorization', value: "Bearer ${TWIL_IO_ACCESS_TOKEN}"]], url: "https://owl-flex-store-7157-dev.twil.io/send-email-notification?emailAddress=${EMAIL}&accountSid=${TWILIO_ACCOUNT_SID}&pluginName=plugin-${JOB_NAME}&statusMessage=Building&pluginHeader=${JOB_NAME}", wrapAsMultipart: false
                        println('Status: '+response.status)
                    }
                }
                stage("Build") {
                    steps {
                        echo "Building..."
                        dir("supervisor-barge-coach-fns") {
                            echo pwd()
                            nodejs("Node-16.18.0") {
                                sh "twilio plugins:install @twilio-labs/plugin-serverless"
                                sh "npm install"
                            }
                        }
                    }
                }
                stage("Test") {
                    steps {
                        echo "Testing..."
                        dir("supervisor-barge-coach-fns") {
                            echo pwd()
                        }
                    }
                }
                stage("Deploy") {
                    steps {
                        echo "Deploying..."
                        dir("supervisor-barge-coach-fns") {
                            echo pwd()
                            nodejs("Node-16.18.0") {
                                sh "twilio serverless:deploy --username=${TWILIO_ACCOUNT_SID} --password=${TWILIO_AUTH_TOKEN}  --override-existing-project --production"
                                // sh "npm run deploy" 
                            }
                        }
                    }
                }
            }
        }
        stage("Flex") {
            stages {
                stage("Build") {
                    steps {
                        //sh "printenv | sort"
                        echo "Building..."
                        dir("plugin-supervisor-barge-coach") {
                            echo pwd()
                            nodejs("Node-16.18.0") {
                                sh "twilio plugins:install @twilio-labs/plugin-flex"
                                sh "npm install"
                                //sh "twilio flex:plugins:upgrade-plugin --install"
                                sh "twilio flex:plugins:build"
                            }
                        }
                    }
                }
                stage("Test") {
                    steps {
                        echo "Testing..."
                        dir("plugin-supervisor-barge-coach") {
                            echo pwd()
                        }
                    }
                }
                stage("Deploy") {
                    steps {
                        echo "Deploying..."
                        dir("plugin-supervisor-barge-coach") {
                            echo pwd()
                            nodejs("Node-16.18.0") {
                                sh "twilio flex:plugins:deploy --major --changelog 'One-Click-Deploy' --description 'Sample OOTB Twilio Flex Plugin'"
                                script {
                                    if(fileExists("package.json")) {
                                        packageJSON = readJSON(file:"package.json");
                                    }
                                 }
                                sh "twilio flex:plugins:release --plugin ${packageJSON.name}@${packageJSON.version} --name 'plugin-supervisor-barge-coach' --description 'Releasing Supervisor barge coach plugin'"
                            }
                        }
                    }
                }
                stage("Post Deploy") {
                    steps {
                        echo "Post Deployment"
                        script {
                            def response = httpRequest customHeaders: [[name: 'Authorization', value: "Bearer ${TWIL_IO_ACCESS_TOKEN}"]], url: "https://owl-flex-store-7157-dev.twil.io/send-email-notification?emailAddress=${EMAIL}&accountSid=${TWILIO_ACCOUNT_SID}&pluginName=plugin-${JOB_NAME}&statusMessage=Deployed&pluginHeader=${JOB_NAME}", wrapAsMultipart: false
                            println('Status: '+response.status)
                        }
                    }
                }
            }
        }
    }
}