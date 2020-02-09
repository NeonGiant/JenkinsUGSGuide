# UE4, Perforce , Jenkins and Unreal Game Sync: 
### A Neon Giant Guide for Continuous Integration in UE4

##### **Disclaimer:**
The present document is a compilation of learning experiences and research done by 2 programmers at Neon Giant AB. without previous experience as build engineers. The intention of this guide is to help other developers that are (or will be) in the same situation we were. This guide will save you a lot of time in research and forum consulting about UE4 and Continuous Integration.

That said, what this guide is not:
- this guide is not going to teach you how to set up a P4 server.
- this guide is not going to teach you how to work with p4 streams and users
- this guide is not going to teach you how to install Jenkins in your server.
- this guide doesn’t cover any source control software other than Perforce.

There’s plenty of documentation online for this topics. We still will give you advice about different things regarding P4 etc when relevant.

##### Terminology used in the guide:

- UE4: Unreal Engine 4.
- P4: perforce
- UGS: Unreal Game Sync
- C.I: Continuous Integration
- CL: Change List (p4)

##### Why P4:
- it’s great when it comes to handling large amounts of files
- it’s great when it comes to handling large files
- It’s fully supported (and recommended) by Epic
- It plays really well with UE4 binary assets when it comes to exclusive check-outs
- Everybody in the team is used to it because our experience in previous companies
- It’s what Epic uses
- Epic’s depot have all the fixes and updates (some of them are not available in git)
- If you work in a project for consoles, all the SDK’s are in their P4 server, so it’s much easier to integrate
- Engine Upgrades are easier to manage with streams

You can set up your own p4 server but, if like us, you don’t want to waste time and resources on it, we recommend you third party solutions like Assembla. We rely on them and it works very well for us.

##### Why Jenkins:
- It is open source, so free to use and very well documented
- There are lots of very useful plugins
- Easy to install an maintaince

##### Why UGS:
UGS is a deployment tool written by Epic to deploy code, assets and binaries to other team members (basically a way to distribute the engine to content creators). We chose this because, well, it’s your only option unless you want to write your own tool. We wrote our own and we used for months but UGS is better in every aspect, so we decided to swap.
Later in this guide we will show how to use it and config it.

### 1.How to Jenkins
#### 1.1.Settings in Jenkins

*A)Plugins*

- [Jenkins P4 plugin](https://plugins.jenkins.io/p4): to manage connections, get latest and submit CL’s
- [Lockable Resources Plugin](https://plugins.jenkins.io/lockable-resources): it’s used to say things like “this build needs to wait for this other build to finish” “if this build fails, don’t do A or B” 
- [Build Pipeline](https://plugins.jenkins.io/build-pipeline-plugin): it allows you to visualize build pipelines and manage

*B) Others*

- A p4 user to be used by Jenkins. By default, p4 users disconnect every 12h for security reasons. Your Jenkins user needs to be permanent, otherwise it will disconnect and builds will fail.

- Global variables in Jenkins. Please, keep in mind that the paths in the image are generic. Adjust them to match your configuration
![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/01.png)

- A variable for the Lockable resource Plugin. 
When a build is happening, you don’t want another one triggering. Specially if they do Get Latest from P4.
![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/02.png)

	Once it’s added, you can check on the state of the lockable resources by clicking the highlighted icon in the image (usually on the left column in the Jenkins Home Page)
![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/03.png)


#### 1.2.Compile the Editor and the project for distribution

In Jenkins you have to create **jobs** for your needs. 
A typical job in a UE4 pipeline is to compile the **editor + project** in development mode so you can distribute it to other content creators. 

This section will describe how that job configuration looks like in our Jenkins server. you will notice we skip sections. That means we don't change the default setup or because they are not relevant for the purpose of this guide.

##### 1.2.1.Discard old builds 
Builds can take a lot of space and you probably don’t need lots of them stored.

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/04.png)

##### 1.2.2.Mark this Job as lockable

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/05.png)

##### 1.2.3.Setup how Jenkins P4 user should behave with this job
We set **Clobber** to avoid warnings when getting assets that are open and **RMDIR** so it deletes empty folders if needed.

*NEON-TIP:* we strongly recommend to setup a virtual stream that filters out all the art source files in P4. This would be all the FBX, PSD and any other files artists use to create content they will later import to UE4. You don’t need that for builds and it can take insane amounts of space and time when getting latest, specially the first time. This virtual stream would be the one you have to setup in **Stream**

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/06.png)

##### 1.2.4.It’s important that you mark this as sync only and that you populate the have list of P4
Otherwise you will encounter problems where files are missing or getting twice when already in the latest revision. The quiet P4 messages is to keep the log clean (more on that later). 

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/07.png)

##### 1.2.5.Enable parallel sync in your P4 server/clients. 
**Pro:** it’s much much faster to sync

**Con:** it can be tricky to know if a file failed since the log will not say anything but “syncing with 10 threads” instead of printing the currently sync file

 ![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/08.png)

##### 1.2.6.Polling
This is the part that tells Jenkins “hey, every N minutes, check if any of this file types have been submitted to P4 and, if yes, start this job and create a new build”. This is fundamental to create new builds when there are source code changes. This information will be used by UGS (more on that later).

It's very important to exclude the Jenkins' P4 user. This user will upload binaries to your P4 server for distribution. If you don't ignore it, it will trigger a new build every time a build is done. 

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/09.png)

##### 1.2.7.Build
The most important part. Here we run 2 commands: one is for UGS (explained in the UGS chapter), the other one to actually do the build. The second command is the bare minimum yo need.

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/10.png)
    
    P:\workspace\Job\Engine\Build\BatchFiles\RunUAT.bat BuildGraph -Script=P:/workspace/Job/Engine/Build/Graph/Examples/BuildEditor_Tools_Game.xml -Target="Submit To Perforce for UGS" -set:EditorTarget=UE4Editor -set:GameTargets=MyProjectEditor -set:ProjectFile=p:\workspace\Job\MyProject\MyProject.uproject -set:ArchiveStream=//depot/main -set:Licensee=true -buildmachine -p4 -submit

Let's explain all this params:

- **RunUAT.bat:** this is the Unreal Automated Tool. What actually makes the build.
- **BuildGraph:** it tells the UAT that we’re going to use a build graph script. More info here
- **Script:** path to the UAT BuildGraph. Recommended by Epic, we created a copy of one they had and modify some lines
- **Target:** 
- **Set:**
- **EditorTarget:**
	- *GameTargets:*
	- *ProjectFile:*
	- *ArchiveStream:*
	- *Licensee:*
	- *buildmachine:*
	- *p4:*
	- *submit:*


##### 1.2.8.Post-Build Actions
What to do after the build is done and what to do if it fails. In this case, both actions refer to UGS. We will come back to this in the UGS section.

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/11.png)

##### 1.2.9.Slack Notifications [Optional]
Jenkins has support for Slack and it’s very convenient to inform about problems with the builds. More info [here](https://medium.com/appgambit/integrating-jenkins-with-slack-notifications-4f14d1ce9c7a)

This is how it looks in our Slack channel for Jenkins.

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/13.png)

In the example, it says “still Failing” because it was the 2nd time in a row that failed. The “Open” link takes you directly to the summary in Jenkins for said Job. You can add as much info in the message as you want (CL that failed, user etc).

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/12.png)


#### 1.3.Build Pipelines and packaged builds

A build Pipeline is a succession of jobs that will trigger one after another. This is very useful for us because we can do this:

1. Re-build a fresh development PC build. (this takes time because of the re-build part)
2. Build a Packaged PC_Development Build
3. Build a Packaged PC_Shipping Build
4. Build a Packaged PC_Test Build

We run this pipeline every night automatically so, when we get to the office in the morning, one of the 2 will happen:

- We will have fresh clean builds in different configuration for QA to test 
- We will have messages in the Slack channel about errors in builds that we can take a look right away.

This is how it looks when set up and running:

![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/14.png)

When you click the **Run** button (top left in the icons line), the 1st build will trigger, once it’s finished, the second one and so on. 
Green if they succeed, red if they don’t. 

- **Rebuild_PC_Development:** the configuration is more or less the same as the one you saw before, but with some exceptions:

	- There’s no polling. We schedule this job on a fixed time
	- Build triggers periodically (the time syntax in Jenkins is not user friendly, you probably want to check their online documentation about it)
	
	![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/15.png)

	- The command used is almost the same as the previous one, but this time we use a re-build BuildGraph
	`%ROOT%\Engine\Build\BatchFiles\RunUAT.bat BuildGraph -Script=P:/workspace/Job/Engine/Build/Graph/Examples/BuildEditor_Tools_Game_Rebuild.xml -set:EditorTarget=UE4Editor -set:GameTargets=MyProject -set:ProjectFile=p:\workspace\Job\MyProject\MyProject.uproject -target="Default Agent" -set:Licensee=true -buildmachine -p4 -submit`

	- **Post-Build Actions:** here’s where we tell it what to do next in regards of the pipeline
	
	![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/16.png)

	- Now, you need to actually configure the Pipeline so you can view it. 
In Jenkins Home, this “**+**” in the tabs area to ad a new tab we will use for the **Pipeline View**

	![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/17.png)

	- Then add a new **Pipeline View**, set a name and hit **Ok**
	
	![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/18.png)

	- In the next window, all you are required to do is to select an initial job
	
	
	![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/19.png)

	
	- Your **Pipeline View** will automatically look for the **Post Build Actions** in that initial job and add all the “chained” jobs it finds, showing the state of the last run builds for each

	![](https://github.com/NeonGiant/JenkinsUGSGuide/tree/master/Images/JenkinsGuide/20.png)

	
### 3.How to UGS

Note: this guide assumes you use Jenkins, but it might as well be used for other solutions like TeamCity

#### 3.1.What is UGS

**Epic's explanation** and official documentation: [UGS](https://docs.unrealengine.com/en-US/Programming/Deployment/index.html). Please read it.


**Our explanation based on our experience: **

Imagine you have a build machine making builds for you. It compiles the editor, the project and it generates artifacts (this is Jenkins slang for binary files, pdb's, libraries... and any other file product of your build job). 
Now you have to deploy said artifacts to the team members that can't compile the editor. 
You have 2 options here:

**Option 1:** Every time there's a new build you, somehow, inform the rest of the team, the access the build machine and download everything new. Then they will need to know on which P4 CL was the build based, so they can sync to that specific CL too.

**Option 2:** you build some short of sync command/launcher and upload everything to P4. The problems you will face (we used our own launcher for months, written in Python).

- The artifacts after building the engine and the project consume a lot of space. You will have to go through the files and, using the p4Ignore file, block files you don't need in your server. In case of PDB's, you will to know which ones you need or not when someone has a bug. We didn't, so everything went up to the server.
 
- You will have to store in a file the CL used for said build. 
- You will have to code your own tool and keep an eye on bugs

The biggest problem is yet to come: syncing and ensuring the right version. 
Imagine this:

- The build was done based on CL #3
- Someone has its local editor build based on CL #2. Said person also has been working on a asset, a blueprint for example. That blueprint is checked out locally. We will call this blueprint BP_Test 
- A new build is up. in CL #3, a programmer has changed the source code for the base class of BP_Test. Changed parameter names, functions and what not.
- The team member gets latest with the launcher. When that person opens the editor, discovers that BP_Test is outdated due changes in source. It's an asset, so you can't merge or diff. Work is lost. 

Unless you add more code to your launcher: a shelve + revert safety. When getting the last build, look for files checked out locally and shelve + revert them.

You might want to also add a button to revert to the previous version. 

Also a button to launch the project and ensure that it's launched with the engine you compile and get from the build machine.

Now, imagine this with engine source changes...

See where are we going? yeah, nightmare land. 


But there's a glorious Option 3: UGS. It deals with this and many other problems we let out to not make this super long. 



##### 3.2. How do I get UGS?

 UGS is available in the UE4 source under "Engine\Source\Programs\UnrealGameSync". Simply compile the project. There are 2 versions of it:

- Release:
- Un-estable release

To learn how to use it, just follow the official documentation (link above), they explain the initial setup better than we will. 

Now, there are missing parts in the documentation:

- How do you deploy the Meta-Data Service? What is it for? Do I need it?
- How do I tell my build machine that I'm using UGS to deploy my builds? How does Jenkins know about it?
- Epic assumptions on setup: the project folder location and the P4 user data

We found this problems and solved them. Some required talking to Epic, some source code modification and others just try and fail. 

We will go over them.

#### 3.3. What's the Meta-Data Service and how do I use it?

#### 3.4. How To UGS and Jenkins

Remember step 1.2.7 where we skip the UGS Command? this is where we talk about it. 

    %ROOT%\Scripts\post_badge_starting.bat %P4_CHANGELIST% "%JOB_NAME%"


#### 3.5 My Project structure is not the default one in Unreal, UGS Can's work with my project

#### 3.6 My P4 user is not the same as my Windows user, UGS can't connect.