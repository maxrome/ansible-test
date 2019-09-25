# How to define a deployment order

By default, after the creation of a Release, the DevOps chain deploy all the components concurrently.

The user can override this behaviour creating an overall pipeline that declares the order of the deployments, defining which deployment must be executed in sequence, and which one can be executed in parallel.

**This behaviour is triggered only when jira project contains the following property in metadata ('devops' group):**

```
deploy_release_job = DeployRelease
```

## 'DeployRelease' pipeline creation, step by step guide

1. In the Bitbucket project, create a project overall git repository. (In charge to DevOps Team)
2. Create the pipeline file in `pipelines/DeployRelease.groovy` (In charge to Dev Team)

   You have to modify only the stages order, adding eventually parallel stages. The pipeline, must have the same parameters and must call the same methods as the pipeline below:

   ```
   #!groovy
   
   @Library('devopsSharedLibs@feature/deployment-order') _
   
   pipeline {
       agent { label 'master' }

       parameters {
           choice(         name: 'LOG_LEVEL', choices: ['DEBUG','INFO','ERROR', 'WARN'],    description: 'The log level of the pipeline')
           booleanParam(   name: 'DRY_RUN_TEST', defaultValue: true, description: 'When true, the    deployment of the components is not actually triggered')
           string(         name: 'DEVOPS_BRANCH', description: 'DevOps branch', trim: true,    defaultValue: "master")
           string(         name: 'PROJECT_KEY', description: 'The Jira project key', trim: true)
           string(         name: 'RELEASE_VERSION', description: 'Release version to deploy',    trim: true)
           string(         name: 'JIRA_RELEASE_ISSUE_KEY', description: 'The Jira Dev-SubTask Key',    trim: true)
           choice(         name: 'TARGET_ENVIRONMENT', choices: ['DEV','TEST','IAT', 'PROD'],    description: 'Target enviroment')
           booleanParam(   name: 'IS_BUSINESS_SIMULATION', description: 'Business simulation',    defaultValue: false)
           text(           name: 'METADATA', description: 'The project and component metadata')
           string(         name: 'BASE_REPOSITORY_GIT_URL', description: 'The base git url used to    compose each component git repository url. (eg: "https://dummybb.devops.vodafone.it/scm/   beapp/")', trim: true)
           text(           name: 'COMPONENTS_LIST', description: 'List of components to deploy    (format COMPONENT_NAME:VERSION). One component for each line.')
       }
   
       stages {
           stage('Parse parameters'){
               steps{
                   parseDeployReleaseParameters()
               }
           }
   
   
           stage('Deploy service-A') {
               when{ expression {isInDeploy('service-A') } }
               steps {
                   triggerDeploy('service-A')
               }
           }


           stage('Deploy Services service-B and service-C') {
   
               parallel {
   
                   stage('Deploy service-B') {
                       when{ expression {isInDeploy('service-B') } }
                       steps {
                           triggerDeploy('service-B')
                       }
                   }

                   stage('Deploy service-C') {
                       when{ expression {isInDeploy('service-C') } }
                       steps {
                           triggerDeploy('service-C')
                       }
                   }
   
               }
           }

           stage('Deploy service-D') {
               when{ expression {isInDeploy('service-D') } }
               steps {
                   triggerDeploy('service-D')
               }
           }
       }

       post {
           failure {
               setJiraReleaseInError()
           }
       }
   }
   ```

   The previous pipelines trigger the deployment of each component, following this flow:

   @startuml

   (*) --> "Service A"
   --> ===B1=== 
   --> "Service B"
   --> ===B2===

   ===B1=== --> "Service C"
   --> ===B2===

   --> "Service D"
   --> (*)

   @enduml

   > When you write this pipeline, **check for each stage, that the `when` statement and the `triggerDeploy` invocation reference the same component. It is responsability of the Dev team if there are these kind of errors.**
   
   > **Be careful, pipelines that do not comply with this format, or that use different parameters, or invoke other methods, are not supported. In these cases, it is responsability of the Dev Team of what the pipelines does.**

3. Creating the `DeployRelease` job in each environment Jenkins (In charge to DevOps Team)
   The job must be placed in PROJECT_NAME/PROJECT_NAME-DeployRelease

   For example, if the project is named DODEVMR, the job must have the path:
   ```
   dodevmr/DODEVMR-DeployRelease
   ```
   The job must references the pipeline located to the project overall git repository

4. Edit the devops project config file (In charge to DevOps Team)
   
   Inside the metadata project section add these properties:
   
   - `deploy_release_job`
   - `deploy_release_job_repository`
   - `deploy_release_job_branch`
   - `deploy_release_job_pipeline`

   For example, if you have created the overall repository `DODEVMR/dodevmr-root` and you have versioned the file `pipelines/DeployRelease.groovy` in the branch `master`, you have to define these metadata:

   ```
   metadata:
     project:
       devops:
         deploy_release_job: DeployRelease
         deploy_release_job_repository: http://it1900yr.it.sedc.internal.vodafone.com:8443/scm/dodevmr/dodevmr-root.git
         deploy_release_job_branch: master
         deploy_release_job_pipeline: pipelines/DeployRelease.groovy
   ```





