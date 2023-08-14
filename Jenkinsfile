pipeline  {

    agent {
        label "system42-builder"
    }

    environment {
        
        ANYPOINT_ORG = credentials("${env.BRANCH_NAME}-anypoint-organization-name")
        ANYPOINT_ENV = credentials("${env.BRANCH_NAME}-anypoint-environment-name")

        ANYPOINT_CLIENT = credentials("${env.BRANCH_NAME}-anypoint-cicd-service-account")
        ANYPOINT_CLIENT_ID = "${ANYPOINT_CLIENT_USR}"
        ANYPOINT_CLIENT_SECRET = "${ANYPOINT_CLIENT_PSW}"
        
        
        ANYPOINT_DEPLOYER = credentials("${env.BRANCH_NAME}-anypoint-api-deployer-account")

        
        API_SPEC_FILE = "AccountsAPIDocumentation.json"
        POLICIES_FILE = "policies.json"

        API_INSTANCE_UPSTREAM_ENDPOINT_URI = "http://accounts-api-mock-inu0ht.5sc6y6-2.usa-e2.cloudhub.io/api"
        API_INSTANCE_LABEL = "cicd-created"
        API_INSTANCE_LISTEN_PORT = "80"

        FLEX_GATEWAY_TARGET_ID = credentials("${env.BRANCH_NAME}-flex-gw-target-id")
    }

    stages {
        stage('Load API Specification') {
            steps {
                script {
                    apiSpec = readJSON file: "${WORKSPACE}/${API_SPEC_FILE}"
                    assetName = apiSpec['info']['title'] + " (CI/CD)"
                    assetId = slugify(assetName)
                    assetDescription = apiSpec['info']['description']
                    assetVersion = apiSpec['info']['version']
                    assetMajorVersion = assetVersion.split("\\.")[0] as int
                    apiVersion = "v${assetMajorVersion}"
                    mainFile = "${WORKSPACE}/${API_SPEC_FILE}"
                    apiBasePath = "/${apiVersion}"
                    apiLabel = "${assetVersion}"

                    echo "Asset Name: ${assetName}"
                    echo "Asset Id: ${assetId}"
                    echo "Asset Version: ${assetVersion}"
                    echo "API Version: ${apiVersion}"
                    echo "Asset Description: ${assetDescription}"
                    echo "Main File: ${mainFile}"
                    echo "API Base Path: ${apiBasePath}"
                    echo "API Label: ${apiLabel}"
                }
            }
        }

        stage('Check API version') {
            steps {
                script {

                    // check if API version is in Exchange
                    status = sh(returnStatus: true, script: "anypoint-cli-v4 exchange:asset:describe ${assetId}/${assetVersion} -o json")
                    echo "Exchange asset describe result status: ${status}"
                    assetVersionNotInExchange = status != 0

                    scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 api-mgr api list --assetId ${assetId} -o json")
                    def instances = readJSON text: scriptOutput.trim()

                    // instances with same major version exists
                    apiInstanceExists = instances.any{ it.productVersion == "${apiVersion}" }

                    // collection of instances to update spec and label
                    instancesToUdate = instances.findAll{ it.productVersion == "${apiVersion}" && it.assetVersion != "${assetVersion}" }
                    echo "Instances to update: ${instancesToUdate}"

                    // collection of instances to deprecate
                    instancesToDeprecate = instances.findAll{ it.productVersion != "${apiVersion}" && !it.deprecated }
                    echo "Instances to deprecate: ${instancesToDeprecate}"
                }
            }
        }

        stage('Create API in Exchange') {
            when {
                expression { assetVersionNotInExchange }
            }
            steps {
                script {
                    scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 exchange:asset:upload --name \"${assetName}\" --description \"${assetDescription}\" --properties='{\"mainFile\": \"${API_SPEC_FILE}\", \"apiVersion\": \"${apiVersion}\"}' ${assetId}/${assetVersion} --type rest-api --files '{\"oas.json\": \"${mainFile}\"}\'")
                    echo "Exchange asset upload result: ${scriptOutput}"
                }                
            }
        }

        stage('Upgrade existing API Instance') {
            when {
                expression { !instancesToUdate.isEmpty() }
            }
            steps {
                script {
                    echo "Updating spec to: ${instancesToUdate}"
                    instancesToUdate.each {
                        instanceId = it.id
                        
                        // update instance specification 
                        scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 api-mgr api change-specification ${it.id} ${assetVersion}")
                        echo "api change-specification output: ${scriptOutput}"
                        
                        // update instance label 
                        scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 api-mgr api edit ${it.id} --apiInstanceLabel ${assetVersion} --port ${API_INSTANCE_LISTEN_PORT} --username ${ANYPOINT_DEPLOYER_USR} --password ${ANYPOINT_DEPLOYER_PSW}")
                        echo "api edit output: ${scriptOutput}"
                    }
                }
            }
        }

        stage('Create new API Instance') {
            when {
                expression { !apiInstanceExists }
            }
            steps {
                script {
                    // create api instance
                    scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 api-mgr:api:manage -f --deploymentType hybrid --uri ${API_INSTANCE_UPSTREAM_ENDPOINT_URI} --path ${apiBasePath} --port ${API_INSTANCE_LISTEN_PORT} --scheme http --type raml --withProxy --apiInstanceLabel ${apiLabel} ${assetId} ${assetVersion}")
                    echo "scriptOutput: ${scriptOutput}"

                    // Store the instanceID in a variable
                    env.APIINSTANCEID = scriptOutput.split(":")[1].trim()
                    echo "Instance created: ${env.APIINSTANCEID}"
                }
            }
        }

        stage('Apply policies to API Instance') {
            when {
                expression { env.APIINSTANCEID != null }
            }
            steps {
                script {

                    policies = readJSON file: "${WORKSPACE}/${POLICIES_FILE}"

                    policies.each {
                        def policyAssetId = it['assetId']
                        def policyGroupId =  it['groupId']
                        def policyAssetVersion = it['assetVersion']
                        def policyConfig = groovy.json.JsonOutput.toJson(it['configurationData'].toString())

                        echo "policyConfig: $policyConfig"

                        status = sh(returnStatus: true, script: "anypoint-cli-v4 api-mgr:policy:apply ${env.APIINSTANCEID} ${policyAssetId} --policyVersion ${policyAssetVersion} --groupId ${policyGroupId} -c $policyConfig")
                        echo "Policy apply result status: ${status}"
                    }
                }
            }
        }

        stage('Deploy API Instance') {
            when {
                expression { env.APIINSTANCEID != null }
            }
            steps {
                script {
                    scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 api-mgr:api:deploy ${apiInstanceId} --target ${FLEX_GATEWAY_TARGET_ID} --overwrite --username ${ANYPOINT_DEPLOYER_USR} --password ${ANYPOINT_DEPLOYER_PSW}")
                    echo "scriptOutput: ${scriptOutput}"
                }
            }
        }

        stage('Deprecate old API Instances') {
            when {
                expression { !instancesToDeprecate.isEmpty() }
            }
            steps {
                script {
                    instancesToDeprecate.each {
                        // deprecate instance 
                        scriptOutput = sh(returnStdout: true, script: "anypoint-cli-v4 api-mgr api deprecate ${it.id}")
                        echo "api ${it.id} deprecation output: ${scriptOutput}"
                    }
                }
            }
        }
    }
}

def String slugify(String input) {
    def slug = input.replaceAll("\\s+", "-")
    slug = slug.replaceAll("\\.", "-")
    slug = slug.replaceAll("[\\/\\(\\)]","")
    slug = slug.replaceAll("^\\w-", "")
    return slug.toLowerCase(Locale.ENGLISH)
}