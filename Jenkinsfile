@Library('jenkins-shared-library')_ 

def configMap = [
    project : "roboshop",
    component : "catalogue"
]

if( !env.BRANCH_NAME.equalsIgnoreCase('main')){
    nodejsEKSPipeline.call(configMap)
}
else {
    echo "Proceed with PROD process"
}


/* @Library('jenkins-shared-library')_ 

def configMap = [
    greeting : "Hello Jenkins"
]

samplePipeline.call(configMap) */