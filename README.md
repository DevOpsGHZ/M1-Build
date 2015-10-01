# M1-Build

##Introduction

 * Trigger a buld in response to a git commit
 * Execute a build job via a shell , which ensures a clean build each time
 * Determine failure or success of a build job, and as a result trigger an email notification
 * Have multiple jobs corresponding to multiple branches in a repository 
 * Track and display a history of past builds (a simple list works) via http

##Setting up
###Vagrant
Install [vagrant](https://www.vagrantup.com/downloads.html) and a virtual machine provider like [VirtualBox](https://www.virtualbox.org/wiki/Downloads).

Initiate a virtual machine:

	vagrant init ubuntu/trusty64
It will automatically create a config file `Vagrantfile`, open this file and uncomment the line:

	config.vm.network "private_network", ip: "192.168.33.10"

You can set the ip to another one, but here we just use the default one.

Then start up the virtual machine and connect to it:

	vagrant up
	vagrant ssh
	
And install some basic stuffs:

	sudo apt-get update
	sudo apt-get install git make vim python-dev python-pip
	sudo pip install virtualenv
###Jenkins
Install [Jenkins](https://jenkins-ci.org/) on the virtual machine we just created.
		
	wget -q -O - https://jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
	sudo sh -c 'echo deb http://pkg.jenkins-ci.org/debian binary/ > /etc/apt/sources.list.d/jenkins.list'
	sudo apt-get update
	sudo apt-get install jenkins
As the Jenkins may need to run command with `sudo`, so we need to add Jenkins to sudoers. Open the sudoers file:

	sudo vi /etc/sudoers

Add a line:

	%jenkins ALL=NOPASSWD: ALL


On host machine, open `192.168.33.10:8080` to Jenkins page. If it says webpage not available, try this command on virtual machine:

	sudo /etc/init.d/jenkins restart
	
Then we need to install some plugins on Jenkins, in Manage Jenkins -> Manage Plugins, install `Git Plugin` and `Multi-Branch Project Plugin`.

Now create a new item, choose `Freestyle multi-branch project`, in the setting page, under the `Source Code Management` section, choose `Git` as the source, and input the path to our local git repository. It will automatically include all the branches in this repo, you can manually exclude some branches in the `Advanced...` setting. 

![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/project-config-git.png)

And under the `Build Triggers` section, check the `Poll SCM`, but leave the schedule empty. 
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/project-config-trigger.png)
Then, in the `Build` section, add a build step, choose `Execute shell`. As it will set all the branches to execute the same shell commands, in order to run different build jobs based on different branches, besides the common commands like install packages, we can let Jenkins to execute a build script in the repository, and the build scripts are different from branch to branch.

	PATH=$WORKSPACE/env/bin:/usr/local/bin:$PATH
	if [ ! -d "env" ]; then
        virtualenv env
	fi
	. env/bin/activate
	sudo env/bin/pip install -r mysite/requirements.txt --download-cache=/tmp/$JOB_NAME
	python mysite/manage.py makemigrations
	python mysite/manage.py migrate
	chmod +x build.sh
	BUILD_ID=dontKillMe ./build.sh
	
Finally, set the post-build action to send an E-mail. In the project configure page, add a `post-build action` -- `Email notification`, input the recipients. Then in Manage Jenkins -> Configure System page, set the System Admin email address:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/general-config-address.png)
And under the Email notification section, set the SMTP server:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/general-config-email.png)

###Multi branch
To handle multiple jobs corresponding to multiple branches in a repository, we use a jenkins plugin [Multi-Branch Project Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Multi-Branch+Project+Plugin). For each branch, this plugin uses a shared configuration to create sub-projects.

![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/multi-branch-plugin.png)

###Build script
In our project, we use a Django project as an example, as the `dev` build job and `release` build job have some differences, so we use different `build.sh`:

dev build.sh:
	
	python mysite/manage.py runserver 0.0.0.0:8000 &

release build.sh:
	
	sed -i '/DEBUG = True/c\DEBUG = False' mysite/mysite/settings.py
	sed -i 's/.*ALLOWED_HOSTS.*/ALLOWED_HOSTS=["www.yourdomain.com"]/' mysite/mysite/settings.py
	python mysite/manage.py runserver 0.0.0.0:8001 &
###Git hooks
In the git repository, create a file `post-commit` in `.git/hooks/`, add these two lines to it:

	#!/bin/sh
	curl http://localhost:8080/git/notifyCommit?url=file:/path/to/repo
	
It means, when there is a commit, it will send a commit notification to `localhost:8080` that in `/path/to/repo` there is a new commit.

Then, run this command to make it executalbe:

	chomd +x .git/hooks/post-commit

##Results
After a `git commit`, it will send a notification to Jenkins:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/trigger-a-build.png)

Then, Jenkins will start the build process if there are changes in the corrsponding branch:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/building.png)

We can see the detailed `console output` in the build pages:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/console-output.png)

And the build history is here:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/build-history.png)

And when the build failed, Jenkins will send an E-mail to the pre-defined address:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/fail.png)

In the E-mail, it contains detailed console output:
![image](https://raw.githubusercontent.com/DevOpsGHZ/M1-Build/master/screenshots/fail-email.png)
