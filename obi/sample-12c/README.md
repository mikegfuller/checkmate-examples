# Checkmate for OBI Quickstart
This Quickstart demonstrates the basic functionality of Checkmate for OBI 8.0.2, using version 12.2.1.2 of OBIEE. The project folder includes sample OBIEE content from [SampleAppLite](http://docs.oracle.com/middleware/12212/biee/BIESG/GUID-E439E473-DD4D-48FE-9BF1-7AED4ADD73B6.htm#BIESG9340) already checked into the [`src/main`](src/main) directory. All you should need to use this Quickstart is an OBIEE 12.2.1.2 environment. We are assuming you are using Linux, so adjust commands slightly if using Windows.

Checkmate is built using [Gradle](www.gradle.org): a declarative, DSL-based build tool most commonly associated with building JVM-based software. Specifically, Checkmate is a series of [Gradle Plugins](https://guides.gradle.org/designing-gradle-plugins/) with the OBI functionality existing in the [com.redpillanalytics.checkmate.obi](https://plugins.gradle.org/plugin/com.redpillanalytics.checkmate.obi) plugin that introduces the following features for Oracle Business Intelligence: source control integration, content versioning and publishing, automated regression and integration testing, and automated deployments.

# Vanilla Configuration
We only need a few parameters to get a vanilla configuration of Checkmate for OBI. Most Gradle configurations exist in the `build.gradle` file, which can exist anywhere in the build filesystem, but we usually put it in the plugin directory or the project directory. In this quickstart, it exists in the project directory. This repository already contains a vanilla [`build.gradle`](build.gradle) file, with several of the advanced features that you will apply later commented out.

For this quickstart, the `build.gradle` file is in the project directory. The very first thing is

```gradle
plugins {
  id 'com.redpillanalytics.checkmate.obi' version '8.0.2'
  id 'maven-publish'
}
```

The `plugins` block applies any desired plugins from the [Gradle Plugin Portal](https://plugins.gradle.org) using the unique ID associated with that plugin.

* `com.redpillanalytics.checkmate.obi`: This is the Checkmate for OBI plugin.
* `maven-publish`: a Core Gradle plugin that enables publishing to Maven repositories. Checkmate for OBI uses Maven Publish to publish distributions of OBI content.

This is the only configuration required to get a basic Checkmate for OBI skeleton working: a Gradle [project with several callable tasks](https://docs.gradle.org/3.5/userguide/tutorial_using_tasks.html#sec:projects_and_tasks). We can use the [Gradle Wrapper](https://docs.gradle.org/3.5/userguide/gradle_wrapper.html) checked in to this repository to see all the tasks associated with the project directory, specified with the `-p` option, and the `tasks` command:

```gradle
./gradlew -p obi/sample-12c tasks
```

The first time any command is executed, Gradle will pull down any library dependencies used by Checkmate for OBI from the central Maven repository called [Bintray jCenter](https://bintray.com/bintray/jcenter), including the Gradle distribution itself.

The Checkmate for OBI enables the following Task groups:
* **Analytics**: Tasks associated with loading data generated from Checkmate builds to downstream data platforms. Will be configured later.
* **Distribution**: Managing the creation and deletion of different types of OBIEE distribution types.
* **Export**: Tasks that facilitate exporting content from an OBIEE instance into the build location or into source control.
* **Import**: Tasks that facilitate importing content into an OBIEE instance, usually using artifacts built in the build location, or downloaded from Maven.
* **SCM**: Tasks for integrating with Source Control Management, specifically [Git](https://git-scm.com).
* **Services**: Just the one `metadataReload` task, which executes *Reload Files and Metadata* in OBIEE.
* **Testing**: Tasks for Regression Testing OBIEE. We haven't configured any Test Groups yet, so the existing tasks are shell tasks for when we configure this later on.

The Maven Publish plugin enables the following Task groups:
* **Publishing**: Publishing content to Maven repositories.

Gradle enables certain default Task groups as well:
* **Build Setup**: For generating new projects, the Gradle Wrapper, etc.
* **Help**: Basic help tasks.
* **Verification**: Runs any configured checks enabled in the project.

# Checkmate Environment Setup
We need to setup the basics about our OBIEE 12.2.1.2 environment so Checkmate knows how and where to execute some of these tasks. We use Checkmate for OBI **build parameters** to enable this. Build parameters can be enabled one of four ways, in reverse-prioritized order... meaning the last item in the list overrides the second-to-the-last item, and so forth:
* Specified in the `build.gradle` file
* Specified in a `gradle.properties` file, which is a standard Java properties file existing in the project directory
* Specified with environment variables
* Specified using Gradle project properties, which are passed to with the Gradle Wrapper command-line using `-P<property>=<value>`

Checkmate provides this degree of flexibility because many build properties are environment specific, and would need to change from one environment to the next. Additionally, some parameters--such as passwords--are sensitive, and need to be treated as such. For instance, [Jenkins Credentials](https://jenkins.io/doc/book/pipeline/syntax/#environment), which are typically used to store passwords in Continuous Delivery environments, are exposed as environment variables, so this is a very handy way to pass sensitive build parameters to Checkmate for OBI.

For the sake of simplictiy and clarity, we'll declare all the build parameters in the `build.gradle` file... even the sensitive ones. Just remember... you would want to use another approach in a real delivery pipeline.

```gradle
obi.middlewareHome = '/home/oracle/fmw/product/12.2.1.2/obi1'
obi.domainHome = '/home/oracle/fmw/config/domains/bi'
obi.compatibility = '12.2.1.2'
obi.adminUser = 'weblogic'
obi.adminPassword = 'Admin123'
obi.repositoryPassword = 'Admin123'
obi.publishBar = false
```

Notes on a few of the parameters below:
* **obi.domainHome:** Defaults to `<obi.middlewareHome>/user_projects/domains/bi`
* **obi.compatibility:** Options are 12.2.1.2, 12.2.1.1, 12.2.1.0, 11.1.1.9 and 11.1.1.7
* **obi.publishBar:** When we publish content (we'll see this in practice shortly), if `obi.publishBar = true`, then Checkmate for OBI will generate the 12c BAR file using the environment specified with the details above. We'll enable this in a bit, but not yet.


Additionally... we need to make all of the command-line tools that OBIEE 12c uses available to Checkmate for OBI. There are numerous ways to configure this, but the easiest is just to add the `$DOMAIN/bitools/bin` directory available to the `$PATH` environment variable:

```bash
export PATH=$PATH:/home/oracle/bin:/home/oracle/fmw/config/domains/bi/bitools/bin
```

# Building and Publishing
The workflow for building OBIEE content usually occurs in the following steps:
* **Build:** The building of OBIEE deployment artifacts from source control. In source control we have the following checked in: the metadata repository as MDS-XML, and the presentation catalog in filesystem structure. The **build** steps involves building a binary RPD, as well as a catalog archive file of the Shared Folders of the catalog.
* **Bundle:** Building **distribution files** of all OBIEE content generated in the **Build** phase. In effect, these are zip files containing repository and catalog content.
* **Publish:** Publishing distribution files to one or more [Maven repositories](https://maven.apache.org/pom.html#Repositories).

Currently, we have the following publications configured in our `build.gradle`:

```gradle
publishing {
  repositories {
    // publish to Maven local repository, which is usually ~/.m2
    // This is only for testing purposes
    mavenLocal()
  }
}
```

This will only publish our OBI distribution files to the [Maven Local](https://docs.gradle.org/3.5/userguide/publishing_maven.html#publishing_maven:install) repository, which defaults to `$HOME/.m2`. Obviously, this is for testing purposes only. In a real continuous delivery pipeline, content should be published to a real Maven-compatible repository. At the very least, publish to an S3 bucket, which Gradle supports, using the following syntax:

```gradle
publishing {
  repositories {
    maven {
      url 's3://bucket-name/path/to/directory'
      credentials(AwsCredentials) {
        accessKey 'access key'
        secretKey 'secret key'
      }
    }
  }
}
```

Why do we bother publishing our OBIEE distributions to Maven repositories? So we can pull that distribution later by simply referring to the version number. This allows us to automate deployments to downstream environments, as Checkmate for OBI can simply pull the distribution file from Maven prior to deployment. It also allows us to automate regression testing. As you will see later in the Quickstart, we can declare published distribution versions as baselines for testing new builds generated by Checkmate for OBI. This gives us a convenient way to pull down any prior distribution for regression testing or integration testing purposes. We can also generate incremental patch files this way: we simply pull down from Maven whatever distribution was last published to our **Production** server, and use that in the patch-creation process.

To build our OBI project, we can simply execute the following:

```gradle
./gradlew -p obi/sample-12c build
```

You'll notice that the `build` task doesn't really do anything on it's own: it's really just a container for two other tasks that do all the work: `metadataBuild` and `catalogBuild`. This introduces Gradle's powerful dependencies and ordering features, which uses a [DAG](https://docs.gradle.org/3.5/userguide/build_lifecycle.html) implementation.

Furthermore... we can run the entire Build, Bundle and Publish workflow by simply running the `publish` task, which has dependencies on building and bundling all the content.

```gradle
./gradlew -p obi/sample-12c publish
```

In the output, you'll notice the *up-to-date* checks that Checkmate for OBI is doing when running the dependent tasks. This can be seen in the following output:

```
:catalogBuild UP-TO-DATE
:metadataBuild UP-TO-DATE
:build UP-TO-DATE
```

Checkmate for OBI is written to take advantage of the [Gradle Incremental Build](https://docs.gradle.org/3.5/userguide/more_about_tasks.html#sec:up_to_date_checks) feature. The catalog and metadata build tasks are not executed again, because none of the task input and output files have changed. This keeps Checkmate from re-running tasks that it doesn't have to.

Let's take a look at what our Build, Bundle and Publish process generated. If we look at the [`build`](build) directory in the project directory, we can see all the things that Checkmate for OBI built, including some of the following:

```shell
ls -l obi/sample-12c/build/*

catalog:
total 1820
drwxr-xr-x. 1 501 games     136 Jul 25 23:58 current
-rw-r-----. 1 501 games 1863215 Jul 25 23:58 current.catalog

repository:
total 28
-rwxr-----. 1 501 games 28456 Jul 25 23:58 current.rpd
drwxr-xr-x. 1 501 games    68 Jul 25 23:58 xml-variables

distributions:
total 8480
-rw-r--r--. 1 501 games 4339941 Jul 26 00:07 sample-12c-build-0.0.9.zip
-rw-r--r--. 1 501 games 4339941 Jul 26 00:07 sample-12c-deploy-0.0.9.zip
```

Additionally, we can look at our Maven Local repository, and see the versioned distribution files published there. You can explore the directory structure further to get a flavor for the Maven filesystem:

```
ls -l ~/.m2/repository/obiee
total 8
drwxrwxr-x. 4 oracle oracle 4096 Jul 19 10:18 sample-12c-build
drwxrwxr-x. 4 oracle oracle 4096 Jul 19 10:18 sample-12c-deploy
```

Later on, we'll expore the differences between the **build** and the **deploy** distributions. For now... just know that **build** is a subset of **deploy**.

# Build Groups
Checkmate for OBI uses the concept of a **build group**: a collection of tasks associated with a particular dependency, which in our case, is a dependency on a published distribution file of OBI content. Up until now, all the tasks demonstrated exist without being tied to a dependency: they are the core Checkmate for OBI tasks that revolve around working with content checked into a Git repository.

A build group allows us to declare a dependency on a prior release of a distribution file, and then get a bunch of new, dynamically generated tasks that belong to that build group. Checkmate contains three build groups by default:
* **feature:** used primarily to regression test new feature branches prior to their being merged into a mainline of code, usually the **develop** or **master** branch.
* **release:** used to regression test new releases prior to being deployed to downstream environments, or prior to being merged into release branches such as **master** or **release.x.x.x**.
* **promote:** used for promoting content to downstream environments.

Because these build groups are already built-in, we don't have to do much to enable them; all we have to do is declare a dependency on a prior distribution version, and the tasks in this build group will magically appear. Since we now have the 0.0.9 distribution published to our Maven Local repository, we can use that distribution as our dependency for these build groups. We can also define custom build groups using the `obi.buildGroups {}` DSL structure.

The first thing we have to tell Checkmate for OBI is where to go looking for our prior distribution files. We're still using Maven Local for this:

```gradle
repositories {
  // resolve local maven repository for published distributions, which is usually ~/.m2
  // this is really only for testing purposes
  mavenLocal()
}
```

We'll make the following changes to our [`build.gradle`](build.gradle) file, which should be possible by simply commenting out a few lines:

```gradle
dependencies {
  // Using the Checkmate Testing library which is recommended.
  obiee group: 'com.redpillanalytics', name: 'checkmate', version: '+'
  // You can also use Baseline Validation Tool
  // The installation needs to be available in one of your Maven repositories
  // If it exists, Checkmate will unzip and install it for you
  //obiee group: 'com.oracle', name: 'oracle-bvt', version: '12.2.1.0.0'

  // Dependencies on previous OBIEE builds
  // Used for building incremental patches, regression testing, and deployments
  feature group: 'obiee', name: 'sample-12c-build', version: '+'
  release group: 'obiee', name: 'sample-12c-build', version: '0.0.9'
  //promote group: 'obiee', name: 'sample-12c-deploy', version: '0.0.9'
  //promote group: 'obiee', name: 'sample-12c-bar', version: '0.0.9'
}
```

There's a lot more in the dependencies closure that we'll discuss later. For now, we'll just focus on the two build group dependencies that we want to uncomment out:

```gradle
feature group: 'obiee', name: 'sample-12c-build', version: '+'
release group: 'obiee', name: 'sample-12c-build', version: '0.0.9'
```

The DSL is a bit confusing, because we are using Gradle's built-in dependency resolution functionality to resolve our OBI distribution files. Basically, we are using a Gradle configuration called **obiee** to declare dependencies on distribution files that we want Checkmate for OBI to pull down and unzip whenever we use one of the tasks in that build group. We are declaring a particular distribution file... in this case, the **build** distribution, with a particular version. Notice for the **feature** build group, we simply have a plus: this signifies to Checkmate for OBI that we simply want to pull down the most recent distribution file. After you uncomment these two dependencies, pay attention to the new tasks that are enabled:

```gradle
./gradlew -p obi/sample-12c tasks
```

You should see a bunch of new tasks enabled that begin with *feature* and *release*. These tasks will perform whatever Checkmate for OBI requires, but will use the content inside the distribution file to faciliate the tasks. In some cases... the build group tasks will use both the content in the distribution file as well as content checked into the Git repository. An example is `releaseCompare`, which will generate incremental patch files for both the repository and the catalog by comparing the content in the distribution file with whatever is in source control. Expect to see some *up-to-date* checks as Checkmate for OBI skips tasks that don't need to be rerun:

```gradle
./gradlew -p obi/sample-12c releaseCompare
```

Now, we can take a look at the enhanced content in our build directory:

```shell
ls -l obi/sample-12c/build/*

obi/sample-12c/build/catalog:
total 3648
drwxr-xr-x. 1 501 games     136 Jul 26 11:42 current
-rw-r-----. 1 501 games 1863173 Jul 26 11:42 current.catalog
drwx------. 1 501 games     102 Jul 26 11:42 init
drwxr-xr-x. 1 501 games     136 Jul 26 11:42 release
-rw-r-----. 1 501 games 1863215 Jul 26 11:42 release.catalog
-rw-r-----. 1 501 games    1739 Jul 26 11:42 release-diff.txt
-rw-r-----. 1 501 games    1739 Jul 26 11:42 release-undiff.txt

obi/sample-12c/build/repository:
total 64
-rwxr-----. 1 501 games 28456 Jul 26 11:42 current.rpd
-rw-------. 1 501 games     0 Jul 26 11:42 release-compare.csv
-rwxr-----. 1 501 games   126 Jul 26 11:42 release-patch.xml
-rwxr-----. 1 501 games 28456 Jul 26 11:42 release.rpd
-rwxr-----. 1 501 games   126 Jul 26 11:42 release-unpatch.xml
drwxr-xr-x. 1 501 games    68 Jul 26 11:42 xml-variables

```

We generated all the incremental patch files, including the rollback patches, but the content of those patch files is empty, because there is currently no difference in what was published to version 0.0.9 and what is currently in source control. But you get the idea. Let's publish again, so we can create a *deploy* distribution, which contains all the patch files, as well as the original repository and catalog artifacts.

```gradle
./gradlew -p obi/sample-12c publish
```

Now, let's enable the **promote** build group, which is what we use for deploying distribution files to downstream environments.

```gradle
dependencies {
  // Using the Checkmate Testing library which is recommended.
  obiee group: 'com.redpillanalytics', name: 'checkmate', version: '+'
  // You can also use Baseline Validation Tool
  // The installation needs to be available in one of your Maven repositories
  // If it exists, Checkmate will unzip and install it for you
  //obiee group: 'com.oracle', name: 'oracle-bvt', version: '12.2.1.0.0'

  // Dependencies on previous OBIEE builds
  // Used for building incremental patches, regression testing, and deployments
  feature group: 'obiee', name: 'sample-12c-build', version: '+'
  release group: 'obiee', name: 'sample-12c-build', version: '0.0.9'
  promote group: 'obiee', name: 'sample-12c-deploy', version: '0.0.9'
  //promote group: 'obiee', name: 'sample-12c-bar', version: '0.0.9'
}
```

Notice that we've uncommented `promote group: 'obiee', name: 'sample-12c-deploy', version: '0.0.9'`, which gives us a series of new Checkmate for OBI tasks to work with the **promote** build group. We'll use the `promotePatch` task, which uses the metadata and catalog incremental patches from the **deploy** distribution file, and applies them to the online OBIEE instance:

```gradle
./gradlew -p obi/sample-12c promotePatch
```

You'll notice that `promotePatch` is a container task for executing `promoteMetadataPatch` and `promoteCatalogPatch`, either of which can be run individually.

# Regression Testing
The high-level process Checkmate for OBI uses to regression test pull requests, source commits, or branch merges, is described below:
* Pull down a distribution file of OBI content published previously.
* Build deployment artifacts from the current content in the Git repository.
* Upload the previous content into an OBIEE environment.
* Run a series of analyses, saving the logical SQL and query results. This is called the *baseline* phase.
* Deploy the current version of content into the OBIEE environment, either using incremental patches, or by deploying whole catalogs and repositories, automatically managing the connection pool information.
* Run the same series of analyses that were run during the *baseline*, saving the output. This is called the *revision* phase.
* Compare the output generated from the baseline phase with the output generated from the revision phase. Naturally, we call this the *compare* phase.


For regression testing, Checkmate for OBI has the concept of a **test group**, which is configured using the `obi.testGroups {}` DSL structure. The default test group called **regression** is already built in, but we can use the same DSL structure to customize that build group:

```gradle
obi.testGroups {
  regression {
    // the catalog folder to test. Recursively tests all analyses in that folder
    // accepts a colon (:) separeted list of directories
    libraryFolder = '/shared'
    // By default, hash values are used for comparision of results
    // You can force the full comparision of logical SQL and results (takes longer)
    compareText = false
    // Provide more output
    showOutput = false
  }
}
```

Let's take a look at some of the parameters configured here, as well as a few others not shown worth mentioning:
* **libraryFolder:** The folder(s) in the presentation catalog that we want to regression test. This can be a colon (:) separated list of catalog folders, and Checkmate recursively includes every analysis in those folders.
* **compareText:** By default, Checkmate for OBI uses hash values of logical SQL and query results to facilitate faster comparisons, which improves performance when logical SQL is very complicated, or the analysis returns a lot of records in the result set. Setting this to `true` enables the full text search of the results.
* **showOutput:** Output more information about the execution of each regression test. Feel free to set this to `true` to see the logical SQL being executed, any error stack generated, etc.
* **impersonateUser:** Specify a user that you want to impersonate for this purposes of this test group. With this functionality, we are able to configure multiple test groups that run the same regression tests but with different security profiles. This requires that the `adminUser` specified above be granted the `impersonate` application role, which is not done by default.

Regression Testing involves some reasonably complex workflows, but we've tried to make that easier with several container tasks that provide the dependencies and ordering to put all of this together. From our `tasks` command, we've highlighted the output of the relevant tasks, listing the granular tasks first, followed by the complex workflow tasks:

```shell
Testing tasks
-------------
baselineTest - Execute all Baseline regression tests for the entire project.
compareTest - Execute all Compare regression tests for the entire project.
extractTestSuites - Extract the compiled test suites and copy them to the 'build/classes' directory.
regressionBaselineLibrary - Create the Baseline regression test library CSV file for Test Group 'regression'.
regressionBaselineTest - Execute the Baseline regression test library file for Test Group 'regression'.
regressionCompareTest - Compare the differences between the Baseline and Revision regression test results for Test Group 'regression'.
regressionRevisionLibrary - Create the Revision regression test library CSV file for Test Group 'regression'.
regressionRevisionTest - Execute the Revision regression test library file for Test Group 'regression'.
revisionTest - Execute all Revision regression tests for the entire project.

Workflow tasks
--------------
releaseBaselineWorkflow - Import 'release' metadata and catalog artifacts, manage connection pools, and execute the Baseline Test Library.
releaseRevisionPatchWorkflow - Apply 'release' metadata and catalog patches, manage connection pools, and execute the Revision Test Library.
releaseRevisionWorkflow - Import 'release' metadata and catalog artifacts, manage connection pools, and execute the Revision Test Library.
```

Executing a regression testing workflow, managing all three phases of the process (**baseline**, **revision**, **compare**), is as easy as executing the following two workflow tasks:

```gradle
./gradlew -p obi/sample-12c releaseBaselineWorkflow
:connPoolsExport
:releaseExtractBuild
:releaseMetadataImport
:releaseCatalogImport
:releaseImport
:connPoolsImport
:regressionBaselineLibrary
:extractTestSuites
:regressionBaselineTest
Results: SUCCESS (42 tests, 42 successes, 0 failures, 0 skipped)
:baselineTest
:releaseBaselineWorkflow

BUILD SUCCESSFUL

Total time: 3 mins 19.836 secs

./gradlew -p obi/sample-12c releaseRevisionWorkflow
:releaseExtractBuild
:releaseMetadataImport
:releaseCatalogImport
:releaseImport
:connPoolsImport
:regressionRevisionLibrary
:extractTestSuites UP-TO-DATE
:regressionRevisionTest
Results: SUCCESS (42 tests, 42 successes, 0 failures, 0 skipped)
:revisionTest
:regressionCompareTest
Results: SUCCESS (85 tests, 84 successes, 0 failures, 1 skipped)
:compareTest
:releaseRevisionWorkflow

BUILD SUCCESSFUL

Total time: 3 mins 21.775 secs
```

Notice that the `releaseRevisionWorkflow` task goes ahead and executes the **Compare** phase of the regression testing, though this is the default, and is of course configurable. It's also worth noting that Checkmate for OBI generates JUnit XML files during this process, which is the industry-standard way of expressing a testing result: all Continuous Delivery and DevOps platforms recognize this standard, and can be configured to ask on the results of those files.

# OBIEE 12c BAR Files
The first releases of Checkmate for OBI pre-dated OBIEE 12c, and therefore, pre-dated the new BAR file functionality in 12c. In a way, the Checkmate for OBI distribution file was our way of building a BAR file... we were just slightly ahead of the game. There's an interesting decision to be made when configuring a Checkmate workflow... to use the BAR file or the distribution file.

At this point in the OBIEE Roadmap, we don't see a lot of value that the BAR file provides over the distribution file... especially since the distribution file also contains incremental patch files, which the BAR file simply doesn't have. Additionally, distribution files can be generated without requiring a running instance of OBIEE; Checkmate for OBI generates the distribution files simply from building content that exist in a Git repository.

However, we recognize that the BAR file is the future for OBI, so we certainly support it. We can generate a BAR file using the `exportBAR` task, and can import a BAR file into OBIEE using the `importBAR` file. Additionally, but simply setting `publishBAR = true`, Checkmate for OBI will automatically generate and publish a BAR file to our Maven repository along with the distribution file.

```gradle
obi.publishBar = true
```

```gradle
./gradlew -p obi/sample-12c publish
:barExport
:generatePomFileForBarPublication
:publishBarPublicationToMavenLocalRepository
:assemble UP-TO-DATE
:catalogBuild UP-TO-DATE
:check UP-TO-DATE
:metadataBuild UP-TO-DATE
:build UP-TO-DATE
:buildZip UP-TO-DATE
:generatePomFileForBuildPublication
:publishBuildPublicationToMavenLocalRepository
:deployZip
:generatePomFileForDeployPublication
:publishDeployPublicationToMavenLocalRepository
:publish

BUILD SUCCESSFUL

Total time: 4 mins 31.606 secs
```

Additionally, it's just as easy to import the BAR file we just created. We have to configure it as a dependency, just as we did with the distribution file. This is because Checkmate for OBI pulls down the desired BAR file from Maven in the same fashion it pulls down the distribution file, and we use the **promote** build group to deploy it:

```gradle
promote group: 'obiee', name: 'sample-12c-bar', version: '0.0.9'
```

Now we are able to run the `releaseBarImport` task:

```gradle
./gradlew -p obi/sample-12c promoteBarImport
:promoteBarImport

BUILD SUCCESSFUL

Total time: 46.59 secs
```

We can also easily import the predefined SampleAppLite BAR file if you ever need to:

```gradle
./gradlew -p obi/sample-12c barImportSAL
:barImportSAL

BUILD SUCCESSFUL

Total time: 1 mins 47.64 secs
```
Or, use the "empty BAR file" that also ships with OBIEE 12c:

```gradle
./gradlew -p obi/sample-12c barReset
:barReset

BUILD SUCCESSFUL

Total time: 39.818 secs
```