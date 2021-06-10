import groovy.json.JsonSlurper
import groovy.json.JsonOutput; 

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurper().parseText(json)
}
def toJSON(def json) {
    new groovy.json.JsonOutput().toJson(json)
}

String INSTANCEARN = ""
String FLOWID = ""
String TRAGETINSTANCEARN = ""
String TARGETFLOWID = ""
String TARGETJSON = ""
String PRIMARYQC = ""
String TARGETQC = ""
String PRIMARYQUEUES = ""
String TARGETQUEUES = ""
String PRIMARYUSERS = ""
String TARGETUSERS = ""
String PRIMARYCFS = ""
String TARGETCFS = ""
String PRIMARYHOP = ""
String TARGETHOP = ""
String PRIMARYPROMPTS = ""
String TARGETPROMPTS = ""

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
                        //TARGETFLOWID = instanceMapping.targetFlowId
                    }
                }
            }
        }
        stage('read all resources') {
            steps {
                echo 'Reading all the resources for ARN resolution'
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                        PRIMARYUSERS =  sh(script: "aws connect list-users --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYUSERS
                        TARGETUSERS =  sh(script: "aws connect  list-users --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETUSERS
                        
                        PRIMARYQUEUES =  sh(script: "aws connect list-queues --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYQUEUES
                        TARGETQUEUES =  sh(script: "aws connect list-queues --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETQUEUES
                        
                        PRIMARYQC =  sh(script: "aws connect list-quick-connects --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYQC
                        TARGETQC =  sh(script: "aws connect list-quick-connects --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETQC 
                        
                        PRIMARYCFS =  sh(script: "aws connect list-contact-flows --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYCFS
                        TARGETCFS =  sh(script: "aws connect list-contact-flows --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETCFS
                        
                        PRIMARYHOP = sh(script: "aws connect list-hours-of-operations --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYHOP
                        TARGETHOP = sh(script: "aws connect list-hours-of-operations --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETHOP

                        PRIMARYPROMPTS = sh(script: "aws connect list-prompts --instance-id ${INSTANCEARN}", returnStdout: true).trim()
                        echo PRIMARYPROMPTS
                        TARGETPROMPTS = sh(script: "aws connect list-prompts --instance-id ${TRAGETINSTANCEARN}", returnStdout: true).trim()
                        echo TARGETPROMPTS
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
                            def flow = jsonParse(di)
                            def content = jsonParse(flow.ContactFlow.Content)
                            TARGETJSON = flow.ContactFlow.Content    
                            def flowId = getFlowId(PRIMARYCFS, flow.ContactFlow.Arn,TARGETCFS).split("/")
                            TARGETFLOWID = flowId[3]
                            String arn = ""
                            echo "Need to update flowId : ${TARGETFLOWID}"
                            for(int i =0; i < content.Actions.size(); i++ )
                            {
                                def obj = content.Actions[i]
                                if(obj.Parameters.equals('MessageParticipant')) {
                                    //handle prompts
                                    arn = getPromptId (PRIMARYPROMPTS, obj.Parameters.PromptId, TARGETPROMPTS)
                                    TARGETJSON = TARGETJSON.replaceAll(obj.Parameters.PromptId, arn)
                                } else if(obj.Parameters.equals('ConnectParticipantWithLexBot')) {
                                    //handle lex box
                                    
                                } else if(obj.Parameters.equals('UpdateContactTargetQueue')) {
                                    //handle queues
                                    arn = getQueueId (PRIMARYQC, obj.Parameters.QueueId, TARGETQC)
                                    TARGETJSON = TARGETJSON.replaceAll(obj.Parameters.QueueId, arn)
                                    
                                } else if(obj.Parameters.equals('UpdateContactEventHooks')) {
                                    //handle flows
                                    arn = getFlowId (PRIMARYCFS, obj.Parameters.QueueId, TARGETCFS)
                                    TARGETJSON = TARGETJSON.replaceAll(obj.Parameters.QueueId, arn)
                                    
                                } else if(obj.Parameters.equals('InvokeLambdaFunction')) {
                                    //handle lambda
                                    
                                } else if(obj.Parameters.equals('TransferToFlow')) {
                                    //handle flows
                                    arn = getFlowId (PRIMARYCFS, obj.Parameters.QueueId, TARGETCFS)
                                    TARGETJSON = TARGETJSON.replaceAll(obj.Parameters.QueueId, arn)
                                    
                                } else if(obj.Parameters.equals('CheckHoursOfOperation')) {
                                    //handle hours of operation
                                    arn = getHOPId (PRIMARYHOP, obj.Parameters.QueueId, TARGETHOP)
                                    TARGETJSON = TARGETJSON.replaceAll(obj.Parameters.QueueId, arn)
                                    
                                } else {
                                    //handle any other resource
                                    echo "No handling for ${obj.Parameters}"
                                }
                            }
                            String json = toJSON(TARGETJSON)
                            json = toJSON(json)
                            TARGETJSON = json.toString()
                     }
                }
            }
        }

                           
        stage('publish flow after arn resolution') {
            steps {
                echo "deploy flow content "
                withAWS(credentials: '71b568ab-3ca8-4178-b03f-c112f0fd5030', region: 'us-east-1') {
                    script {
                        def di =  sh(script: "aws connect update-contact-flow-content --instance-id ${TRAGETINSTANCEARN} --contact-flow-id ${TARGETFLOWID} --content ${TARGETJSON}", returnStdout: true).trim()
                        echo di
                    }
                }
            }
        }
        
    }
}


def getFlowId (primary, flowId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for flowId : $flowId"
    for(int i = 0; i < pl.ContactFlowSummaryList.size(); i++){
        def obj = pl.ContactFlowSummaryList[i]    
        if (obj.Arn.equals(flowId)) {
            fName = obj.Name
            println "Found flow name : $fName"
            break
        }
    }
    println "Searching for flow name : $fName"        
    for(int i = 0; i < tl.ContactFlowSummaryList.size(); i++){
        def obj = tl.ContactFlowSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Arn
            println "Found flow id : $rId"
            break
        }
    }
    return rId
}

def getQueueId (primary, queueId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for queueId : $queueId"
    for(int i = 0; i < pl.QueueSummaryList.size(); i++){
        def obj = pl.QueueSummaryList[i]    
        if (obj.Arn.equals(queueId)) {
            fName = obj.Name
            println "Found queue name : $fName"
            break
        }
    }
            
    for(int i = 0; i < tl.QueueSummaryList.size(); i++){
        def obj = tl.QueueSummaryList[i]    
        if (obj.Name.equals(fName)) {
            rId = obj.Arn
            println "Found flow id : $rId"
            break
        }
    }
    return rId
    
}

def getUserId (primary, userId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for userId : $userId"
    for(int i = 0; i < pl.UserSummaryList.size(); i++){
        def obj = pl.UserSummaryList[i]    
        if (obj.Arn.equals(userId)) {
            fName = obj.Username
            println "Found user name : $fName"
            break
        }
    }
    println "Searching for userId for : $fName"        
    for(int i = 0; i < tl.UserSummaryList.size(); i++){
        def obj = tl.UserSummaryList[i]    
        if (obj.Username.equals(fName)) {
            rId = obj.Arn
            println "Found flow id : $rId"
            break
        }
    }
    return rId    
}

def getHOPId (primary, hopId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for userId : $userId"
    for(int i = 0; i < pl.HoursOfOperationSummaryList.size(); i++){
        def obj = pl.HoursOfOperationSummaryList[i]    
        if (obj.Arn.equals(hopId)) {
            fName = obj.Name
            println "Found user name : $fName"
            break
        }
    }
    println "Searching for hopId for : $fName"        
    for(int i = 0; i < tl.HoursOfOperationSummaryList.size(); i++){
        def obj = tl.HoursOfOperationSummaryList[i]    
        if (obj.Username.equals(fName)) {
            rId = obj.Arn
            println "Found flow id : $rId"
            break
        }
    }
    return rId    
}

def getPromptId (primary, searchId, target) {
    def pl = jsonParse(primary)
    def tl = jsonParse(target)
    String fName = ""
    String rId = ""
    println "Searching for userId : $userId"
    for(int i = 0; i < pl.PromptSummaryList.size(); i++){
        def obj = pl.PromptSummaryList[i]    
        if (obj.Arn.equals(searchId)) {
            fName = obj.Name
            println "Found user name : $fName"
            break
        }
    }
    println "Searching for hopId for : $fName"        
    for(int i = 0; i < tl.PromptSummaryList.size(); i++){
        def obj = tl.PromptSummaryList[i]    
        if (obj.Username.equals(fName)) {
            rId = obj.Arn
            println "Found flow id : $rId"
            break
        }
    }
    return rId    
}

