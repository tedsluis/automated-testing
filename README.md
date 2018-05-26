# Setting-up a Jenkins pipeline on OpenShift
  
Tested on OpenShift 3.3 - 3.7  
  
note: Deploy the Jenkins with persistent storage from the Red Hat template. This comes with single sign-one and doesn't need to run with root priviledges.    
  
## Documentation
* https://github.com/openshift/jenkins-plugin
* https://docs.openshift.org/latest/using_images/other_images/jenkins.html
* https://jenkins.io/doc/book/pipeline/syntax/
* https://jenkins.io/doc/book/pipeline/jenkinsfile/
* https://jenkins.io/doc/pipeline/steps/workflow-basic-steps/
* https://jenkins.io/doc/pipeline/steps/openshift-pipeline/
* https://docs.openshift.org/latest/dev_guide/builds/build_strategies.html#jenkinsfile
* https://developers.redhat.com/blog/2017/01/09/using-pipelines-in-openshift-3-3-for-cicd/
* https://wilsonmar.github.io/jenkins2-pipeline/
  
## Notes  
   
#### Add secret with .gitconfig (with sslVerify = false) for the OpenShift builder 
Apply in every namespace whenever the error encountered: 'error: fatal: unable to access 'https://github.com/......./': Peer's Certificate issuer is not recognized.'  
* Create secret with .gitconfig (sslVerify = false).
* Link the secret to the builder serviceaccount. 
* Insert annotate into the secret.  
  
For this you need to be admin in the requiered namespace, for example:  
~~~
$ cd jenkins
$ oc secrets new mycert .gitconfig=.gitconfig
secret/mycert
$ oc secrets link builder mycert
$ oc annotate secret mycert 'build.openshift.io/source-secret-match-uri-1=https://github.com/*'
secret "mycert" annotated
~~~
   
#### Add edit permissions to the 'jenkins' serviceaccount for the requiered namespaces 
For this you need to be admin in the requiered namespaces, for example:   
~~~
$ oc policy add-role-to-user edit system:serviceaccount:automated-testing:jenkins -n 
~~~
    
#### Add permission to for serviceaccounts pull images from the requiered namespaces   
For this you need to be admin in the requiered namespaces, for example:  
~~~
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:testing --namespace=automated-testing
$ oc policy add-role-to-group system:image-puller system:serviceaccounts:testing --namespace=openshift
~~~
   
#### Export Jenkins plugins and import it into an other Jenkins  
Manage plugins and plugin versions via a plugin file. 
  
To export a list of plugins names with versions and save them in al file (format = plugin-name:version).  
Go to the script console (for example: http://jenkins-automated-testing.wildcard/script) and run this script:    
~~~
Jenkins.instance.pluginManager.plugins.each{
  plugin -> 
    println ("${plugin.getDisplayName()} (${plugin.getShortName()}): ${plugin.getVersion()}")
}
~~~
Copy the output to a file with the extention .hpi and save it in git.  
  
To import a list of plugin names with versions go to 'plugin management' 'advanced tab', for example:   
http://jenkins-automated-testing.wildcard/pluginManager/advanced   
Upload the [plugins.hpi](https://github.com/tedsluis/automated-testing/blob/master/plugins.hpi) plugin file. 
  
### Create build configs for Jenkins pipelines
This part is only usefull if you want to intergrate your pipelines within OpenShift. 
To being able to trigger Jenkins build from within OpenShift you can use a buildconfig with the method 'jenkinsfile'.  
Such a buildconfig contains the git repository name with a the path to jenkinsfile (groovy code) in that git repo.  
Down here you find two examples of two pipelines. Via the command ir will create a buildconfig. Before running it, be sure you configure your own git repo and jenkinsfile with the template:  
~~~
$ oc process -f ocp-pipelines/basetest/basetest-template.yaml | oc create -f -
$ oc process -f ocp-pipelines/cleanup/cleanup-template.yaml | oc create -f -
~~~
  
Ones you have created a buildconfig with a Jenkinsfile, you start it via the openShift GUI (build, pipelines), via the commandline ('oc start-build buildconfigname') or from within Jenkins itself.   
   
~~~
$ oc get bc
NAME          TYPE              FROM         LATEST
cleanup-ota   JenkinsPipeline   Git@master   5
ota           JenkinsPipeline   Git@master   41
~~~
  
#### Example groovy code
* https://github.com/tedsluis/automated-testing/blob/master/ocp-pipelines/basetest/jenkinsfile  
* https://github.com/tedsluis/automated-testing/blob/master/ocp-pipelines/node-test/jenkinsfile  
* https://github.com/tedsluis/automated-testing/blob/master/ocp-pipelines/cleanup/jenkinsfile  
     
#### Add .gitconfig to Jenkins home 
You may need to add '/var/lib/jenkins/.gitconfig' within the Jenkins pod: 
~~~
[http]
     sslVerify = false
~~~
   
#### Backup Jenkins pipeline
To back-up files with the Jenkins pod use:  
~~~
$ oc get pods jenkins -o name
namespace/jenkins-2-63b6r
$ oc rsync --include=true  jenkins-2-63b6r:/var/lib/jenkins/jobs/OpenShift-deployment-test/config.xml jenkins/jobs/OpenShift-deployment-test
~~~

