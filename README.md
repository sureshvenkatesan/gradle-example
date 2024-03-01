## Gradle build , Xray Scan,  promote , Save dependencies for future builds , quarantine builds

This Lab explains how to :
a) publish a gradle build to DEV repo sdx-app-gradle-dev-local by resolving dependencies from remote repo sdx-app-gradle-remote
b) then promote the build to release candidate repo sdx-app-gradle-rc-local
c) then promote the build to release repo sdx-app-gradle-release-local

Note: All these repos are of package-type gradle.

Lock the dependencies 2 ways:
a) When you do a release  lock the dependencies  in a seperate local repo  sdx-app-ext-libs-local
or
b) During build promotion promote the dependencies alongside the actual application i.e in the release repo sdx-app-gradle-release-local

Encapsulate all these repos in a virtual repo:
- first this virtual has only the sdx-app-gradle-dev-local , sdx-app-gradle-rc-local , sdx-app-gradle-release-local and
  sdx-app-gradle-remote
- Then demonstrate by locking the depndencies in the sdx-app-ext-libs-local you can even remove the remote and still have the possibility to reproduce the build. For example , 2 years from now you want tio fix a bug and rebuild the application with the same set of dependencies ( and if the remote repo doesnot have these old depndencies ) , you should still have the ability to build it . So the build can then resolve the needed dependencies from sdx-app-ext-libs-local .

We demonstrate it by removing the `sdx-app-gradle-remote` from the `sdx-app-gradle-virtual` and instead use the `sdx-app-ext-libs-local` in the `sdx-app-gradle-virtual` and be able to build successfully.

---

## Steps to build using the JFrog cli for build integration.

On mac pull the master branch and do the following:
cd to the working folder i.e gradle-example

In my test I did
```
cd ~/myCode/github-sv/gradle-example
```

Remove the dependencies already cached:
```
rm -rf .gradle
rm -rf ~/.gradle/caches/*
```
Remove all .jar and .class files in a directory and its subdirectories
```
find . -type f \( -name "*.jar" -o -name "*.class" \) -delete

```
Configure the jfrog cli to use your artifactory instance.

jfrog rt c psazuse --url=https://psazuse.jfrog.io/artifactory --user=admin --password=your-password


Create the repos need for the lab  :
```
jf rt curl  -X PUT api/repositories/sdx-app-ext-libs-local -H "Content-Type: application/json" -T create_repositories/sdx-app-ext-libs-local.json --server-id=psazuse
jf rt curl  -X PUT api/repositories/sdx-app-gradle-dev-local -H "Content-Type: application/json" -T create_repositories/sdx-app-gradle-dev-local.json --server-id=psazuse
jf rt curl  -X PUT api/repositories/sdx-app-gradle-rc-local -H "Content-Type: application/json" -T create_repositories/sdx-app-gradle-rc-local.json --server-id=psazuse
jf rt curl  -X PUT api/repositories/sdx-app-gradle-release-local -H "Content-Type: application/json" -T create_repositories/sdx-app-gradle-release-local.json --server-id=psazuse

jf rt curl  -X PUT api/repositories/sdx-app-gradle-remote -H "Content-Type: application/json" -T create_repositories/sdx-app-gradle-remote.json --server-id=psazuse

jf rt curl  -X PUT api/repositories/sdx-app-gradle-virtual -H "Content-Type: application/json" -T create_repositories/sdx-app-gradle-virtual.json --server-id=psazuse
```

Do the Gradle config:
```
jf  gradlec --server-id-resolve=psazuse --server-id-deploy=psazuse --repo-resolve=sdx-app-gradle-virtual --repo-deploy=sdx-app-gradle-virtual
```

Note: This creates the config file in ./.jfrog/projects/gradle.yaml with below content:
```
version: 1
type: gradle
resolver:
    repo: sdx-app-gradle-virtual
    serverId: psazuse
deployer:
    repo: sdx-app-gradle-virtual
    serverId: psazuse
    deployMavenDescriptors: true
    deployIvyDescriptors: true
    ivyPattern: '[organization]/[module]/ivy-[revision].xml'
    artifactPattern: '[organization]/[module]/[revision]/[artifact]-[revision](-[classifier]).[ext]'

```

Create the build to be indexed by xray:
```
jf xr curl -XPOST api/v1/binMgr/builds -H 'content-type:application/json' -d '{"names": ["sdx-app-gradle-build"]}' --server-id psazuse
```
Create the Xray policies ( for only critical vulnerabilities)
```
jf xr curl -XPOST -H "Content-Type: application/json" api/v2/policies -T create_xray_policies_watches/sdx-security-policy.json --server-id psazuse
```

Create the Xray watch  , to watch the 4 repos in the virtual and the build for policy violations.
Since it is monitoring the build as well , you can create the build before hand and not want to publish the build-run ( that we do in later steps)
```
jf xr curl -XPOST -H "Content-Type: application/json" api/v2/watches -T create_xray_policies_watches/sdx-watch.json --server-id psazuse 
```

Do the Gradle build ( run number 4) , which also deploys the binaries built to the sdx-app-gradle-virtual repo. 
```
jf gradle clean artifactoryPublish -b ./build.gradle --build-name=sdx-app-gradle-build --build-number=4
```

Collect environments variables : 
```
jf rt bce sdx-app-gradle-build 4
```

Gradle Build publish
```
jf rt bp sdx-app-gradle-build 4
```

If you have already published previous builds you can show the build diff between 2 runs.

Gradle Xray build scan
```
jf rt bs sdx-app-gradle-build 4
```
It will output a json which shows the build scan detected violations (CVEs).

Echo the error code from last command i.e the Gradle build scan
```
echo $?
```
Output: 3 indicates there were errors detected in the scan.

Now show the violations for this build  in the Xray tab in UI.

---
Assume there were no violations in the scan , then you can promote the build. 

### Gradle build promotion .
In the demo do the promotion without dependencies ( this is the default i.e --include-dependencies="false")
```
jf rt bpr sdx-app-gradle-build 4 sdx-app-gradle-rc-local --status="Security Scan Passed" --comment="No vulnerabilities" --server-id psazuse 
```
Note: Build promotion will move the deployed artifacts from `sdx-app-gradle-dev-local`  to the target `sdx-app-gradle-rc-local`

Now show the "Release History" of this build to show that the "Security Scan Passed".

Note: Another option is with dependencies ( not demoing this)
```
jf rt bpr sdx-app-gradle-build 4 sdx-app-gradle-rc-local --status="Security Scan Passed" --comment="No vulnerabilities" --include-dependencies="true" --server-id psazuse 
```

Now run some QA , regression tests etc and promote to release without the dependencies.

Release promotion : Without dependencies : 
```
jf rt bpr sdx-app-gradle-build 4 sdx-app-gradle-release-local --status="Released" --comment="prod-ready" --server-id psazuse 
```
Show the "Release History" of this build.

---
Now to ensure the build reproducibility , save the dependencies using the ./filespec-aql.json AQL.
Here copy only the dependencies not already ( from a previous build) to the target repo.
Show the filespec-aql.json .

If promoting with dependencies, then promoting separately via a copy + filespec
```
jf rt cp --spec="./filespec-aql.json" --spec-vars="sdx-build-name=sdx-app-gradle-build;sdx-build-number=4;sdx-target-repo=sdx-app-ext-libs-local" --server-id psazuse 
```
It searches the depndencies and copies them  ( not move) from the remote repo cache to the sdx-app-ext-libs-local repo.

After the command completes Show the `sdx-app-ext-libs-local` repo.

Now remove the `sdx-app-gradle-remote` from the `sdx-app-gradle-virtual` and add the `sdx-app-ext-libs-local` instead.

---
Now run a new build to show that it can resolve the dependencies now from `sdx-app-ext-libs-local`

```
jf gradle clean artifactoryPublish -b ./build.gradle --build-name=sdx-app-gradle-build --build-number=5
```
The build is successful.
Note: Dont save the dependencies to `sdx-app-ext-libs-local` for snapshot builds. Save them only for `release` builds. 

---
Another way to save the dependencies is by build promote of  a `release` build .
Note: I skip one step i.e the release candidate promotion , just for demo purpose.
So publish the build number 5.

```
jf rt bp sdx-app-gradle-build 5
```

Now promote With dependencies:
```
jf rt bpr sdx-app-gradle-build 5 sdx-app-gradle-release-local --status="Released" --comment="prod-ready" --include-dependencies="true" --server-id psazuse 
```

Show the build 5 is  "Released" ( as seen in the status) and the dependencies are now in the `sdx-app-gradle-release-local` repo alongside my  actual app i.e org.jfrog. ...

Now as long as you dont do the cleanup , you can safely even remove the `sdx-app-ext-libs-local` repo from the `sdx-app-gradle-virtual` and stillnbe able to run a new build successfully.

---
If you want to quarntine builds which failed xray scan , to can even 
promote to a quarantine repo  `sdx-app-quarantine-local` (without dependencies) :
```
jf rt bpr sdx-app-gradle-build 4 sdx-app-quarantine-local --status="Security Quarantine" --comment="Vulnerabilities found at build time" 
```
