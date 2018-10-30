# sample-jenkins-pipeline

## Outline

This is a sample project to define a Jenkins Pipeline. It includes stages as follows:

- Build Images
- Deploy Pods with Postgres, Fuse, DS and AMQ.
- Promete images to Dev

## Environment

Requirments

- a project which is deployed Jenkins Server automatically.
- Source codes which is deployed on EAP and Decision Server.  

Versions

- Tested on OpenShift 3.9

## How to deploy

1. Clone this repository

~~~
$ git clone https://github.com/kanekoh/sample-jenkins-pipeline
$ cd sample-jenkins-pipeline
~~~

2. Create new project or use an existing project.
~~~
$ oc new-project <Project Name>
~~~

3. Create BuildConfig for pipeline.

~~~
oc process -f pipeline.yaml | oc create -f -
~~~

~~~
oc adm policy add-cluster-role-to-user cluster-admin system:serviceaccount:project5:jenkins
~~~

TODO: Will add some paramters be defhined in pipeline.