
On mac pull the master branch and do the following:
cd to the working folder i.e gradle-example

I did cd ~/myCode/github-sv/gradle-example

rm -rf .gradle

Remove the dependencies already cached:
```
rm -rf ~/.gradle/caches/*
```
Remove all .jar and .class files in a directory and its subdirectories
```
find . -type f \( -name "*.jar" -o -name "*.class" \) -delete

```
Configure the jfrog cli to use your artifactory instance.

jfrog rt c psazuse --url=https://psazuse.jfrog.io/artifactory --user=admin --password=your-password

jf rt use psazuse

Create the repos need for the lab:
```
jf rt curl  -X PUT api/repositories/sdx-app-ext-libs-local -H "Content-Type: application/json" -T sdx-app-ext-libs-local.json --server-id=psazuse
jf rt curl  -X PUT api/repositories/sdx-app-gradle-dev-local -H "Content-Type: application/json" -T sdx-app-gradle-dev-local.json --server-id=psazuse
jf rt curl  -X PUT api/repositories/sdx-app-gradle-rc-local -H "Content-Type: application/json" -T sdx-app-gradle-rc-local.json --server-id=psazuse
jf rt curl  -X PUT api/repositories/sdx-app-gradle-release-local -H "Content-Type: application/json" -T sdx-app-gradle-release-local.json --server-id=psazuse

jf rt curl  -X PUT api/repositories/sdx-app-gradle-remote -H "Content-Type: application/json" -T sdx-app-gradle-remote.json --server-id=psazuse

jf rt curl  -X PUT api/repositories/sdx-app-gradle-virtual -H "Content-Type: application/json" -T sdx-app-gradle-virtual.json --server-id=psazuse
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
Create the Xry watches and policies
```
jf xr curl -XPOST -H "Content-Type: application/json" api/v2/policies -T sdx-security-policy.json --server-id psazuse
jf xr curl -XPOST -H "Content-Type: application/json" api/v2/watches -T sdx-watch.json --server-id psazuse 
```

Gradle build
jf gradle clean artifactoryPublish -b ./build.gradle --build-name=sdx-app-gradle-build --build-number=4

Collecting environments variables : 
jf rt bce sdx-app-gradle-build 4

Gradle Build publish
jf rt bp sdx-app-gradle-build 4

Gradle build scan
jf rt bs sdx-app-gradle-build 7

Gradle build promotion
with dependencies
jf rt bpr sdx-app-gradle-build 5 sdx-app-gradle-rc-local --status="Security Scan Passed" --comment="No vulnerabilities" --include-dependencies="true"

without dependencies
jf rt bpr sdx-app-gradle-build 7 sdx-app-gradle-rc-local --status="Security Scan Passed" --comment="No vulnerabilities" 
Update security policy

Run a new build
Run a scan
promote to quarantine (without dependencies)
jf rt bpr sdx-app-gradle-build 4 sdx-app-quarantine-local --status="Security Quarantine" --comment="Vulnerabilities found at build time" 

Release promotion : 
With dependencies
jf rt bpr sdx-app-gradle-build 5 sdx-app-gradle-release-local --status="Released" --comment="prod-ready" --include-dependencies="true"

Without dependencies : 
jf rt bpr sdx-app-gradle-build 7 sdx-app-gradle-release-local --status="Released" --comment="prod-ready"

If promoting with dependencies, then promoting separately via a copy + filespec
jf rt cp --spec="./filespec-aql.json" --spec-vars="sdx-build-name=sdx-app-gradle-build;sdx-build-number=7;sdx-target-repo=sdx-app-ext-libs-local"