# Stage Specific Agents, Inline Pipeline Pod Temaplates and Cross Team Collaboration

In this exercise we will explore stage specific agents, a more advanced usage of the Kubernetes plugin by defining a Pod Template inline in our Pipeline Jenkinsfile and will demonstrate [CloudBees Core Cross Team Collaboration feature](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/).

## Stage Specific Agents and Agent None

Up to this point we have had only one global `agent` defined and it is being used by all `stages` of our `pipeline`. However, we don't need an `agent` for the ***Build and Push Image*** `stage`. We will update the Pipeline to have **no** global `agent` and use the current global `nodejs-app` `agent` just for the Test `stage`.

1. Open the GitHub editor for the `Jenkinsfile` file in the **master** branch of your forked **helloworld-nodejs** repository.
2. Replace the global `agent` section with the following:
```
  agent none
```

3. Next, in the **Test** `stage` add the following `agent` section right above the `steps` section:
```
    agent { label 'nodejs-app' }
```
4. You may be asking yourself how the `steps` are able to run in the `stages` where there is no `agent`. Every Pipeline script runs on the Jenkins Master using a **flyweight executor** (i.e. Java thread). However, certain Pipeline `steps` require a heavyweight executor - that is an executor on an `agent` ([more info on flyweight vs heavyweight executors](https://support.cloudbees.com/hc/en-us/articles/360012808951-Pipeline-Difference-between-flyweight-and-heavyweight-Executors)). One such step is the `sh` step. We will add such a step to the ***Build and Push Image*** `stage` to illustrate this. Add an `sh` step to the ***Build and Push Image*** `stage` after the `echo` step so the stage looks like the following:
```
    stage('Build and Push Image') {
      when {
        beforeAgent true
        branch 'master'
      }
      steps {
        echo "TODO - build and push image"
        sh 'java -version'
      }
    }
```
5. Commit the changes and navigate to the **helloworld-nodejs** job in Blue Ocean on your Team Master and the job for the **master** branch should be running or queued to run. The job will fail with the following error: <p><img src="img/cross-team/stage_agent_master_fail.png" width=800/>
6. Open the GitHub editor for the `Jenkinsfile` file in the **master** branch of your forked helloworld-nodejs repository
7. Remove the `sh 'java -version'` step from the ***Build and Push Image*** `stage` and commit the changes to the **master** branch.
8. The commit will trigger the **helloworld-nodejs** **master** branch job again and it will complete successfully.

## Kubernetes Pod Templates Defined in Pipeline Script

In this exercise we will add another Docker **container** for executing tests. We also want to use a different vesion of the **node** Docker image than the one provided by the Master managed Pod Template which is `node:8.12.0-alpine`. So far we have been using the **nodejs-app** [Kubernetes *Pod Template* defined for us at our Team Master level](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/agents/#_editing_pod_templates_per_team_using_masters). In order to be able to control what `containers` and what Docker `image` version we use in our Pipeline we will update the **Jenkinsfile** Pipeline script with an [*inline* Kubernetes Pod Template definition](https://github.com/jenkinsci/kubernetes-plugin#declarative-pipeline).

1. The [Jenkins Kubernetes plugin allows you to use standard Kubernetes Pod yaml configuration](https://github.com/jenkinsci/kubernetes-plugin#using-yaml-to-define-pod-templates) to define Pod Templates directly in your Pipeline script. We will do just that in a new `nodejs-pod.yaml` file. The `yamlFile` parameter value of the `kubernetes` agent definition is a repository relative path to a yaml file representing the [Pod spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#pod-v1-core) you want to use as an agent Pod Template. At the top-level of the **master** branch of your forked copy of the **helloworld-nodejs** repository click on the **Create new  file** button towards the top right of the screen. 
2. Name the file `nodejs-pod.yaml` and add the following content:
```
kind: Pod
metadata:
  name: nodejs-app
spec:
  containers:
  - name: nodejs
    image: node:10.10.1-alpine
    command:
    - cat
    tty: true
  - name: testcafe
    image: 946759952272.dkr.ecr.us-east-1.amazonaws.com/kypseli/testcafe:alpha-1
    command:
    - cat
    tty: true
```

3. At the bottom of the screen enter a commit message, leave ***Commit directly to the `master` branch** selected and click the **Commit new file** button
4. Now we need to update our Pipeline to use that file. Open the GitHub editor for the **Jenkinsfile** Pipeline script in the **master** branch of your forked **helloworld-nodejs** repository.
5. Replace the `agent` section of the **Test** `stage` with the following - note that the valule of the `yamlFile` parameter is the name of the file we created:

```
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yamlFile 'nodejs-pod.yaml'
        }
      }
```


6. Commit the changes and then navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master. The job will queue indefinitely, but why? 
7. The answer is provided by the [CloudBees Kube Agent Management plugin](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/agents/#monitoring-kubernetes-agents). Exit to the classic UI on your Team Master and navigate up to the **helloworld-nodejs** Multibranch folder. On the bottom left of of the screen there is a dedicated widget that provides information about the ongoing provisioning of Kubernetes agents. It also highlights failures, allowing you to determine the root cause of a provisioning failure. Click on the link for the failed or pending pod template. <p><img src="img/cross-team/pipeline_pod_template_failure.png" width=800/>
8. You will see that the **nodejs** container has an error - it looks like there is not a **node** container image available with that tag. If you go to [Docker Hub and look at the tags available for the **node** image](https://hub.docker.com/r/library/node/tags/) you will see there is a **10.10.0-alpine** but not a **10.10.1-alpine** tag for the **node** image: <p><img src="img/cross-team/pipeline_pod_template_containers_error.png" width=800/> 
9. Abort the current run (or it will keep trying to load that faulty pod template forever) <p><img src="img/cross-team/pipeline_pod_template_stop.png" width=800/> 
10. Next, open the GitHub editor for the **nodejs-pod.yaml** Pipeline script in the **master** branch of your forked **helloworld-nodejs** repository. Update the `image` for the **nodejs** `container` to be `node:10.10.0-alpine`.
11. Commit the changes and then navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master. The job will run successfully. Also, note the output of the `sh 'node --version'` step - it is `v10.10.0` instead of `v8.12.0`: <p><img src="img/cross-team/pipeline_pod_template_node_version.png" width=850/>

## Cross-Team Master Events

The [Cross Team Collaboration feature](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/) is designed to greatly improve team collaboration by connecting team Pipelines to deliver software faster. It essentially allows a Pipeline to create a notification event which will be consumed by other Pipelines waiting on it. It consists of a [**Publishing Event**](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/#cross-team-event-publishers) and a [**Trigger Condition**](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/#cross-team-event-triggers).

The Cross Team Collaboration feature has a configurable router for routing events and it needs to be configured on your Team Master before you will be able to receive the event published by another Team Master. Once again, CasC was used to pre-configure this for everyone, but you can still check it by going to the top-level of your Team Master in the classic UI, clicking on **Manage Jenkins** and then clicking on **Configure Notification** - you should see the following configured: <p><img src="img/cross-team/cross_team_config.png" width=800/>

1. Now our pipeline must must be updated to listen for a **hello-api-deploy-event** event. We will do that by adding a `trigger` to your **Jenkinsfile** Pipeline script.
2. Open the GitHub editor for the **Jenkinsfile** file in the **master** branch of your forked **helloworld-nodejs** repository.
3. Add the following `trigger` block just above the top-level `stages` block:

```groovy
  triggers {
    eventTrigger simpleMatch('hello-api-deploy-event')
  }
```

4. Commit the changes and then navigate to the **master** branch of your **helloworld-nodejs** job in Blue Ocean on your Team Master. 

>**NOTE:**After first adding a new `trigger` you must run the job at least once so that the `trigger` is saved to the Jenkins job configuration (similar to what was necessary for the `buildDiscarder` `option` earlier). <p><img src="img/cross-team/cross_team_trigger_configured.png" width=850/>

Now I will set up a Multinbranch Pipeline project for the https://github.com/cloudbees-days/helloworld-api repository. The **helloworld-api** repository contains `Jenksfile` that publishes the **hello-api-deploy-event** [simple event](https://go.cloudbees.com/docs/cloudbees-core/cloud-admin-guide/cross-team-collaboration/#cross-team-event-types). That event will be published **across all Team Masters in our Workshop cluster** via the CloudBees Operations Center event router causing everyones' **helloworld-nodejs** Pipelines to be triggered. 

Now I will run the **helloworld-api** job and everyone should see the **master** branch of their **helloworld-nodejs** job triggered. <p><img src="img/cross-team/cross_team_triggered_by_event.png" width=850/>

After you have completed the above exercises, you can make sure that your **Jenkinsfile** Pipeline script is correct by comparing to or copying from [below](https://github.com/cloudbees-days/cloudbees-core-workshop/blob/master/cross-team-collaboration.md#finished-jenkinsfile-for-pipeline-pod-temaplates-and-cross-team-collaboration).

Please check out the [CloudBees Core for K8s CD/Jenkins X Workshop](https://github.com/cloudbees-days/jenkins-x-workshop) and the [CloudBees DevOptics Workshop](https://github.com/cloudbees-days/devoptics-workshop).

## Next Lesson

Before moving on to the next lesson you can make sure that your **Jenkinsfile** Pipeline script on the **master** branch of your forked **helloworld-nodejs** repository matches the one from [below](#finished-jenkinsfile-for-pipeline-pod-templates-and-cross-team-collaboration).

You may proceed to the next set of exercises - **[Pipeline Catalog Templates](./catalog-templates.md)** - when your instructor tells you.

### Finished Jenkinsfile for *Pipeline Pod Templates and Cross Team Collaboration*
```
pipeline {
  agent none
  options { 
    buildDiscarder(logRotator(numToKeepStr: '2'))
    skipDefaultCheckout true
  }
  triggers {
    eventTrigger simpleMatch('hello-api-deploy-event')
  }
  stages {
    stage('Test') {
      agent {
        kubernetes {
          label 'nodejs-app-inline'
          yamlFile 'nodejs-pod.yaml'
        }
      }
      steps {
        checkout scm
        container('nodejs') {
          echo 'Hello World!'   
          sh 'node --version'
        }
      }
    }
    stage('Build and Push Image') {
      when {
        beforeAgent true
        branch 'master'
      }
      steps {
        echo "TODO - build and push image"
      }
    }
  }
}
```
