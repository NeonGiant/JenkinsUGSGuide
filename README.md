# UE4, Perforce , Jenkins and Unreal Game Sync:
### A Neon Giant Guide for Continuous Integration in UE4

### Table Of Contents
1. [Introduction: disclaimer, terminology and reasons why](#introduction)
2. [How to Jenkins](#jenkins)
	1. [Settings in Jenkins](#jenkins_settings)
	2. [Compile the Editor and the project for distribution](#jenkins_compile_job)
		1. [Discard old builds](#jenkins_discard_builds)
		2. [Mark this Job as lockable](#jenkins_lock_jobs)
		3. [Setup how Jenkins P4 user should behave with this job](#jenkins_p4_setup)
		4. [Polling](#jenkins_polling)
		5. [Build, Build Command and Post Build](#jenkins_build)
		6. [Slack Notifications](#jenkins_slack)
	3. [Build Pipelines and packaged builds](#jenkins_pipelines)
3. [How to UGS](#ugs)
4. [Extra: The Derive Data Cache](#DDC)

### 1.Introduction <a name="introduction"></a>
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
- Easy to install an maintain

##### Why UGS:
UGS is a deployment tool written by Epic to deploy code, assets and binaries to other team members (basically a way to distribute the engine to content creators). We chose this because, well, it’s your only option unless you want to write your own tool. We wrote our own and we used for months but UGS is better in every aspect, so we decided to swap.

### 2.How to Jenkins <a name="jenkins"></a>
#### 2.1.Settings in Jenkins <a name="jenkins_settings"></a>

*A)Plugins*

- [Jenkins P4 plugin](https://plugins.jenkins.io/p4): to manage connections, get latest and submit CL’s
- [Lockable Resources Plugin](https://plugins.jenkins.io/lockable-resources): it’s used to say things like “this job needs to wait for this other build to finish” “if this build fails, don’t do A neither B” 
- [Build Pipeline](https://plugins.jenkins.io/build-pipeline-plugin): it allows you to visualize build pipelines and manage them

*B) Other Settings*

- A p4 user to be used by Jenkins. By default, p4 users disconnect every 12h for security reasons. Your Jenkins user needs to be permanent, otherwise it will disconnect and builds will fail.

- Global variables in Jenkins. Please, keep in mind that the paths in the image are generic. Adjust them to match your configuration
![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/01.png)

- A variable for the Lockable resource Plugin. 
When a build is happening, you don’t want another one triggering. Specially if they do Get Latest from P4.
![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/02.png)

	Once it’s added, you can check on the state of the lockable resources by clicking the highlighted icon in the image (usually on the left column in the Jenkins Home Page)
![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/03.png)


#### 2.2.Compile the Editor and the project for distribution <a name="jenkins_compile_job"></a>

In Jenkins you have to create **jobs** for your needs. 
A typical job in a UE4 pipeline is to compile the **editor + project** in development mode so you can distribute it to other content creators. 

This section will show how that job configuration looks like in our Jenkins server. You will notice we skip sections. That means we don't change the default setup or that they are not relevant for the purpose of this guide.

##### 2.2.1.Discard old builds <a name="jenkins_discard_builds"></a>
Builds can take a lot of space and you probably don’t need many old builds.

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/04.png)

##### 2.2.2.Mark this Job as lockable <a name="jenkins_lock_jobs"></a>

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/05.png)

##### 2.2.3.Setup how Jenkins P4 user should behave with this job <a name="jenkins_p4_setup"></a>
We set **Clobber** to avoid warnings when getting assets that are open and **RMDIR** so it deletes empty folders if needed.

*NEON-TIP:* we strongly recommend to setup a virtual stream that filters out all the art source files in P4. This would be all the FBX, PSD and any other files artists use to create content they will later import to UE4. You don’t need that for builds and it can take insane amounts of space and time when getting latest, specially the first time. This virtual stream would be the one you have to setup in **Stream**

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/06.png)

##### 2.2.3.1.It’s important that you mark this as sync only and that you populate the have list of P4
Otherwise you will encounter problems where files are missing or getting twice when already in the latest revision. The quiet P4 messages is to keep the log clean. 

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/07.png)

##### 2.2.3.2.Enable parallel sync in your P4 server/clients. 
**Pro:** it makes syncing much faster

**Con:** you lose the ability of seeing which file is syncing. It will only say “syncing with N threads” in the log instead. 

 ![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/08.png)

##### 2.2.4.Polling <a name="jenkins_polling"></a>
This is the part that tells Jenkins “hey, every N minutes, check if any of this file types have been submitted to P4 and, if yes, start this job and create a new build”. In our case we check every 30min. This is fundamental to create new builds when there are source code changes. This information will be used by UGS (more on that later).

It's very important to exclude the Jenkins' P4 user. This user will upload binaries to your P4 server for distribution. If you don't ignore it, it will trigger a new build every time a build is done (infinite loop of build creation). 

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/09.png)

##### 2.2.5.Build <a name="jenkins_build"></a>
The most important part. Here we run 2 commands: one is for UGS (explained in the UGS chapter), the other one to actually do the build. The second command is the bare minimum you will need.

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/10.png)
    
    P:\workspace\Job\Engine\Build\BatchFiles\RunUAT.bat BuildGraph -Script=P:/workspace/Job/Engine/Build/Graph/Examples/BuildEditor_Tools_Game.xml -Target="Submit To Perforce for UGS" -set:EditorTarget=UE4Editor -set:GameTargets=MyProjectEditor -set:ProjectFile=p:\workspace\Job\MyProject\MyProject.uproject -set:ArchiveStream=//depot/main -set:Licensee=true -buildmachine -p4 -submit

The build script we use is a modified copy of `Engine\Build\Graph\Examples\BuildEditorAndTools.xml`.
 
- **RunUAT.bat:** this is the Unreal Automated Tool. What actually makes the build.
- **BuildGraph:** it tells the UAT that we’re going to use a build graph script. More info [here](https://docs.unrealengine.com/en-US/Programming/BuildTools/AutomationTool/BuildGraph/Usage/index.html)
- **Script:** path to the UAT BuildGraph.
- **Target:** indentifier for the node that will be executed. Open our file and look for `Submit To Perforce for UGS`
- **Set:**
	- *EditorTarget:* Editor to compile
	- *GameTargets:* Game Targets to build
	- *ProjectFile:* path to the project file
	- *ArchiveStream:* Whether to embed changelist number into binaries
	- *Licensee:* Whether to treat the changelist number as being from a licensee Perforce server
	- *buildmachine:* To mark this CL as a build machine CL, so UGS will not consider it as a engine/game change
	- *p4:* Enables Perforce functionality (default if run on a build machine)
	- *submit:* Allows UAT command to submit changes

If you have doubts about which flags can you pass, check in `Engine\Source\Programs\AutomationTool\AutomationUtils\Automation.cs`.

Now, the changes we needed to do in the file:


1.`<Option Name="ProjectFile" Restrict="[^ ]*" DefaultValue="" Description="Path to game target project file"/>`

This is because we changed the folder structure of our project. We don't have the project within the engine folder, but on on a separate folder at the same level. UGS assumes that's how you are going to work with it. 
There's no good reason to change it and we don't recommend it. There's no gain on it. 

2.`<Option Name="OutputDir" DefaultValue="$(RootDir)\..\..\LocalBuilds\$(EditorTarget)Binaries" Description ="Output directory for compiled binaries"/>`

Changed for the exact same reason. If you stick to UE4 default folder structure, you will be fine.

3.`<Option Name="ForceSubmit" Restrict="true|false" DefaultValue="false" Description="Forces the submit to happen even if another change has been submitted (resolves in favor of local files)"/>`

We removed this line because we will not force any submits. Up to you. 

4.`<SetVersion Change="$(Change)" CompatibleChange="$(CodeChange)" Branch="$(EscapedBranch)" Licensee="$(Licensee)" If="$(Versioned)"/>`

5.`<Compile Target="CrashReportClient" Platform="Win64" Configuration="Shipping" Tag="#ToolBinaries"/>`

We always want to build the crash report so every developer in the team can report back crashes with callstacks we can trace back

6.`<Compile Target="UnrealHeaderTool" Platform="Win64" Configuration="Development" Arguments="-precompile" Tag="#ToolBinaries"/>`

We want to make static libraries for all engine modules as intermediates for this target
(to check available Arguments look into `Engine\Source\Programs\UnrealBuildTool\Configuration\TargetRules.cs`)

7.`<Compile Target="UnrealFrontend" Configuration="Development" Platform="Win64" Tag="#ToolBinaries"/>`

We always build the Unreal Frontend since our QA team uses it to make local builds. Useful too for VLog reports.

8.
 
	<Node Name="Compile $(EditorTarget) Win64" Requires="Compile Tools Win64" Produces="#EditorBinaries">
    	<Compile Target="$(EditorTarget)" Platform="Win64" Configuration="Development" Tag="#EditorBinaries"/>
    </Node>

9.`<Compile Target="$(GameTarget) $(ProjectFile)" Platform="$(TargetPlatform)" Configuration="Development" Tag="#GameBinaries_$(GameTarget)_$(TargetPlatform)"/>`

Added `$(ProjectFile)` due the change in the folder structure.

10.`<Compile Target="$(GameTarget)" Platform="$(TargetPlatform)" Configuration="Shipping" Tag="#GameBinaries_$(GameTarget)_$(TargetPlatform)"/>`

We removed this. We only want this build script for development. For shipping we have a different one.

11.

    <Node Name="Tag Output Files" Requires="#ToolBinaries;$(GameBinaries)" Produces="#OutputFiles">
    	<Tag Files="#ToolBinaries;$(GameBinaries)" Except=".../Intermediate/..." With="#OutputFiles"/>
    </Node>

12.`<Copy Files="#OutputFiles" From="$(RootDir)\..\..\" To="$(OutputDir)"/>` and any other change you can see where "\..\..\" has been added.

Problems with the folder structure.

13.`<Submit Description="[CL $(CodeChange)] Updated binaries" Files="$(ArchiveFile)" FileType="binary+FS32" Workspace="$(COMPUTERNAME)_ArchiveForUGS" Stream="$(ArchiveStream)" Host="neongiant/neongiant.myProject_ue4" RootDir="$(ArchivePerforceDir)" If="'$(ArchiveStream)' != ''"/>`

We removed "force submit".
We need to pass a different Host because of our P4 server is hosted by a 3rd party (Assembla).

If you need to pass a host too, you will need to edit the file `Engine\Source\Programs\AutomationTool\BuildGraph\Tasks\SubmitTask.cs` in two places:

- within `public class SubmitTaskParameters` scope, add this

        /// <summary>
    	/// The host for the workspace specified; defaults to the machine name.
    	/// </summary>
    	[TaskParameter(Optional = true)]
    	public string Host;

- then, within the function `public override void Execute(JobContext Job, HashSet<FileReference> BuildProducts, Dictionary<string, HashSet<FileReference>> TagNameToFileSet)` look for `if (Parameters.Workspace != null)`. At that point, Epic grabs the parameters for P4. Add this line after the client is cached. 

`Client.Host = Parameters.Host ?? Environment.MachineName;`

##### 2.2.6.Post-Build Actions
What to do after the build is done and what to do if it fails. In this case, both actions refer to UGS. We will come back to this in the UGS section.

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/11.png)

##### 2.2.7.Slack Notifications [Optional] <a name="jenkins_slack"></a>
Jenkins has support for Slack and it’s very convenient to inform about problems with the builds. More info [here](https://medium.com/appgambit/integrating-jenkins-with-slack-notifications-4f14d1ce9c7a)

This is how it looks in our Slack channel for Jenkins.

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/13.png)

In the example, it says “still Failing” because it was the 2nd time in a row that failed. The “Open” link takes you directly to the summary in Jenkins for said Job. You can add as much info in the message as you want (CL that failed, user etc).

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/12.png)


#### 2.3.Build Pipelines and packaged builds <a name="jenkins_pipelines"></a>

A build Pipeline is a succession of jobs that will trigger one after another. This is very useful for us because we can do this:

1. Re-build a fresh development PC build (this takes time because of the re-build part)
2. Build a Packaged PC_Development Build
3. Build a Packaged PC_Shipping Build
4. Build a Packaged PC_Test Build

We run this pipeline every night automatically so, when we get to the office in the morning, one of this 2 will happen:

- Either we will have fresh clean builds in different configuration for QA to test 
- Or will have messages in the Slack channel about errors in builds that we can take a look right away.

This is how it looks when set up and running:

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/14.png)

When you click the **Run** button (top left in the icons line), the 1st build will trigger, once it’s finished, the second one and so on. 
Green if they succeed, red if they don’t. 

- **Rebuild_PC_Development:** the configuration is more or less the same as the one you saw before, but with some exceptions:

	- There’s no polling. We schedule this job on a fixed time
	- Build triggers periodically (the time syntax in Jenkins is not user friendly, you probably want to check the online documentation about it)
	
	![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/15.png)

	- The command used is almost the same as the previous one, but this time we use a re-build BuildGraph
	`%ROOT%\Engine\Build\BatchFiles\RunUAT.bat BuildGraph -Script=P:/workspace/Job/Engine/Build/Graph/Examples/BuildEditor_Tools_Game_Rebuild.xml -set:EditorTarget=UE4Editor -set:GameTargets=MyProject -set:ProjectFile=p:\workspace\Job\MyProject\MyProject.uproject -target="Default Agent" -set:Licensee=true -buildmachine -p4 -submit`

	- **Post-Build Actions:** here’s where you specify the next step in the pipeline
	
	![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/16.png)

	- Now, you need to actually configure the Pipeline so you can view it. 
In Jenkins Home, click “**+**” in the tabs area to ad a new tab we will use for the **Pipeline View**

	![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/17.png)

	- Then add a new **Pipeline View**, set a name and hit **Ok**
	
	![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/18.png)

	- In the next window, all you are required to do is to select an initial job

	
	![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/19.png)

	
	- Your **Pipeline View** will automatically look for the **Post Build Actions** in that initial job and add all the “chained” jobs it finds, showing the state of the last run builds for each

	![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/20.png)

	
### 3. How to UGS <a name="ugs"></a>

The best thing you can do is read the official documentation: [UGS](https://docs.unrealengine.com/en-US/Programming/Deployment/index.html). It gives a good explanation on how to get things up and running. 

Remember steps 1.2.7 and 1.2.8 where we skip the UGS Command? this is where we talk about it. 

`%ROOT%\Scripts\post_badge_starting.bat %P4_CHANGELIST% "%JOB_NAME%`

There's one problem we had with Epic's code and UGS: it assumes that the hostname for P4 is the machine name and, as we mentioned when explaining the build command, we have our p4 hosted by a 3rd party, so we use a completely different name.
We fixed this problem by editing the file `Engine\Source\Programs\AutomationTool\AutomationUtils\P4Environment.cs`. 

Somewhere in the file (line 179 in UE4 4.22), you will find this

    P4ClientInfo ThisClient;
    if(String.IsNullOrEmpty(Client))

within that `if`, add something like this with your build machine name and the url/name of your P4 host. 

    
    string HostName = System.Net.Dns.GetHostName();
    if (HostName == "YOUR_BUILD_MACHINE_NAME")
    {
    	HostName = "url/url.myproject_ue4";
    }

### 4. Extra: The Derive Data Cache <a name="DDC"></a>

One thing we do and that we strongly recommend, is to have a job in Jenkins to build the Derive Data Cache. This will serve 3 purposes:

- It will clean it from old assets
- It will spot errors in assets
- It will compile the shaders for everyone with access to the DDC.

The last part is the most important for use. We have seen cases where someone opens a level and bam! 15 thousand un-compiled shaders are there waiting for you!

Here you can find a good explanation on what's the [DDC](https://docs.unrealengine.com/en-us/Engine/Basics/DerivedDataCache)

This is how we set it up.

- First step is, as the documentation states, to set the DDC in your network and your editor.

- We lock it so it doesn't mess up with other jobs

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/21.png)

- Set it as a parameterized project and add a parameter. The parameters are used to tell UE4 for which platforms it needs to compile the shaders/textures etc. If your project is to be released for pc and consoles, you could do something like "WindowsNoEditor+XboxOne+PS4+Switch".

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/22.png)

- Set it to build every night

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/23.png)

- Last step is to tell the job to execute a batch command when building the ddc. 

![](https://github.com/BrunoRebaque/NeonGiant/blob/master/Images/JenkinsGuide/24.png)

    %ENGINE_ROOT%\Engine\Binaries\Win64\UE4Editor.exe %MY_PROJECT%\MyProject.uproject -run=DerivedDataCache -fill -TargetPlatform=%Platforms% -ddc=BuildMachineDerivedDataBackendGraph

The last parameter in the command refers to a new data cache setting you will have to add to the `DefaultEngine.ini` file within your project.
 
This prevents the build machine to create its own copy of the DDC. The DDC can use a lot of space. The bigger the project, the more you need, so this step is very much recommended.

In the `Shared=` section you will need to add your network path to the shared DDC.

    [BuildMachineDerivedDataBackendGraph]
    MinimumDaysToKeepFile=30
    Root=(Type=KeyLength, Length=120, Inner=AsyncPut)
    AsyncPut=(Type=AsyncPut, Inner=Hierarchy)
    Hierarchy=(Type=Hierarchical, Inner=Boot, Inner=Pak, Inner=EnginePak, Inner=Shared)
    Boot=(Type=Boot, Filename=%GAMEDIR%DerivedDataCache/Boot.ddc, MaxCacheSize=65536)
    Local=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, PurgeTransient=true, DeleteUnused=true, UnusedFileAge=90, FoldersToClean=-1, Path=../../../Engine/DerivedDataCache)
    Shared=(Type=FileSystem, ReadOnly=false, Clean=false, Flush=false, DeleteUnused=true, UnusedFileAge=90, FoldersToClean=-1, Path=Path_To_your_shared_DDC, EnvPathOverride=UE-SharedDataCachePath)
    Pak=(Type=ReadPak, Filename=%GAMEDIR%DerivedDataCache/DDC.ddp)
    EnginePak=(Type=ReadPak, Filename=../../../Engine/DerivedDataCache/DDC.ddp)
    