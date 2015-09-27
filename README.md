# M1-Build

##Introduction

 * Trigger a buld in response to a git commit
 * Execute a build job via a shell , which ensures a clean build each time
 * Determine failure or success of a build job, and as a result trigger an email notification
 * Have multiple jobs corresponding to multiple branches in a repository 
 * Track and display a history of past builds (a simple list works) via http

##Steps

1. Install and configure vagrant   
   * Initiate vagrant vm using `vagrant init ubuntu/trusty64`+`vagrant up`     
   *  Edit Vagrantfile: uncomment `config.vm.network "private_network", ip: "192.168.33.10"` 
   * run `vagrant reload` +`vagrant ssh`
2. Install Jenkins     
   * `wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -`
   * `sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'`
   * `sudo apt-get update`
   * `sudo apt-get install jenkins`
3. Configure Jenkins in dashboard
   * open 192.168.33.10:8080
   * Manage Jenkins -> Manage Plugins then install GIT plugin and 	Multi-Branch Project Plugin.
4. Configure a new item in Jenkins
   * Create a freestyle new item
   * Choose Git as Source Code Management then input git repository url or local directory
   * Choose Poll SCM under Build Triggers
   * **Execute shell**
5. Create Git hook
   * In vagrant, create a file 'post-comment' under .git/hooks/ directory, inside the file write: `#!/bin/sh
curl` and `http://192.168.33.10 :8080/git/notifyCommit?url=file:/`

##Results

