import groovy.json.JsonSlurper
import groovy.json.JsonOutput; 

def CONTACTFLOW = ""

def INSTANCEARN = ""
def FLOWID = ""

String TRAGETINSTANCEARN = ""
String TARGETFLOWID = ""
String TARGETJSON = ""

pipeline {
    agent any
    stages {
        
        
        stage('git repo & clean') {
            steps {
                   script{
                    try{
                      sh(script: "rm -r ac-flows-via-api", returnStdout: true)
                    }catch(Exception e){
                      println ("Exception occured " + e.toString())
                    }
                   sh(script: "git clone https://github.com/ramprasadsv/ac-flows-via-api.git", returnStdout: true)
                   sh(script: "ls -ltr", returnStatus: true)
               }
            }
        }
        
        stage('read config from git') {
            steps{
                echo 'Reading the contact flow content '
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                        def params = sh(script: 'cat parameters.json', returnStdout: true).trim()    
                        echo params                      
                        def instanceMapping = jsonParse(params)
                        INSTANCEARN = instanceMapping.primaryInstance
                        FLOWID = instanceMapping.flowId
                        TRAGETINSTANCEARN = instanceMapping.targetInstance
                        TARGETFLOWID = instanceMapping.targetFlowId
                    }
                }
            }
        }        
        stage('read flow from api') {
            steps {
                    echo 'Reading the contact flow content via api'
                    withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                        script {
                            def di =  sh(script: "aws connect describe-contact-flow --instance-id ${INSTANCEARN} --contact-flow-id ${FLOWID}", returnStdout: true).trim()
                            echo di
                            def data2 = sh(script: 'cat arnmapping.json', returnStdout: true).trim()    
                            def flow = jsonParse(di)
                            def arnmapping = jsonParse(data2)
                            String content = flow.ContactFlow.Content    
                            for(i = 0; i < arnmapping.size(); i++){
                                content = content.replaceAll(arnmapping[i].sourceARN, arnmapping[i].targetARN)
                            }
                            String json = toJSON(content)
                            TARGETJSON = json.toString()
                     }
                }
            }
        }
        
        stage('deploy updated flow after api') {
            steps {
                echo "Updating contact flow after reading from api "
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                        def di =  sh(script: "aws connect update-contact-flow-content --instance-id ${TRAGETINSTANCEARN} --contact-flow-id ${TARGETFLOWID2} --content ${TARGETJSON}", returnStdout: true).trim()
                        echo di
                    }
                }
            }
        }
        
    }
}

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurper().parseText(json)
}
def toJSON(def json) {
    new groovy.json.JsonOutput().toJson(json)
}

