# react-yarn-azure-pipeline
This is a simple example of azure devops pipeline (in yml) to deploy a react project in azure App service (Windows machine). 
Let's say we have a react project - 

  - is maintained with yarn package management 
  - has .env files in the root (usually .env files are not part of a git repo)


# step1
- Prepare .env files (.env .env.development .env.production)
- Prepare a web.config file with the following rewrite rules (given we have a widows machine to host the azure app service)

```
<?xml version="1.0"?>
<configuration>
	<system.webServer>
		<rewrite>
			<rules>
				<rule name="React Routes" stopProcessing="true">
					<match url=".*" />
					<conditions logicalGrouping="MatchAll">
						<add input="{REQUEST_FILENAME}" matchType="IsFile" negate="true" />
						<add input="{REQUEST_FILENAME}" matchType="IsDirectory" negate="true" />
						<add input="{REQUEST_URI}" pattern="^/(api)" negate="true" />
					</conditions>
					<action type="Rewrite" url="/" />
				</rule>
			</rules>
		</rewrite>
	</system.webServer>
</configuration>
```

Add all those files (.env .env.development .env.production web.config) to azure devops library as secure files. We can download these secure files in the build machine using a DownloadSecureFile@1 pipeline task (yml). This way we are making sure the correct .env file is provided in the build machine before the task yarn build --mode development in the pipeline.

<img src="azure-library.png" />

# step2
Let's write the pipeline
```
# React App Deployment Pipeline

trigger:
  - main

pool:
  vmImage: 'windows-latest'

# define variables to use during the build
variables:
  projectFolder: '.'
  buildOutputFolder: 'dist'

# installing node.js
steps:
  - task: NodeTool@0
    inputs:
      versionSpec: '16.x'
    displayName: 'Install Node.js'

  # Run the yarn install
  - script: |
      yarn install

  # Download secure file from azure library
  - task: DownloadSecureFile@1
    inputs:
      secureFile: '.env.development'

  # Copy the .env file
  - task: CopyFiles@2
    inputs:
      sourceFolder: '$(Agent.TempDirectory)'
      contents: '**/*.env.development'
      targetFolder: '$(projectFolder)'
      cleanTargetFolder: false

  # Run the yarn build
  - script: |
      yarn build --mode development

  # Download secure file from azure library
  - task: DownloadSecureFile@1
    inputs:
      secureFile: 'web.config'

  # Copy the web.config file
  - task: CopyFiles@2
    inputs:
      sourceFolder: '$(Agent.TempDirectory)'
      contents: '**/*web.config'
      targetFolder: '$(buildOutputFolder)'
      cleanTargetFolder: false

  # Copy all the files to the staging directory
  - task: CopyFiles@2
    inputs:
      sourceFolder: '$(buildOutputFolder)'
      contents: '**/*'
      targetFolder: '$(Build.ArtifactStagingDirectory)'
      cleanTargetFolder: true

  # Archive all the files into a zip file for publishing
  - task: ArchiveFiles@2
    inputs:
      rootFolderOrFile: '$(Build.ArtifactStagingDirectory)'
      archiveType: 'zip'
      archiveFile: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'
      includeRootFolder: false

  # Publish the zip file
  - task: PublishBuildArtifacts@1
    inputs:
      pathtoPublish: '$(Build.ArtifactStagingDirectory)/$(Build.BuildId).zip'

```

Now the build pipeline is completed which will create a artifact in a .zip file. Go ahead and create a release pipeline and deploy the artifact to an App service. ENJOY!
