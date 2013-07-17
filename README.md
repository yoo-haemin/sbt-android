# Android SDK Plugin for SBT #

Current version is 0.7.4

*WARNING* `0.7.2` has _broken_ scala projects. Update or use `0.7.1` instead.

Note: 0.7.0 and later is incompatible with build files for previous versions
of the plugin.

## New features in 0.7.x ##

* Projects can now follow an ant-style or gradle-style layout. The location
  of `AndroidManifest.xml` will auto-select which layout to use, if it is
  at the top-level, ant-style will be selected and under `src/main` will
  choose gradle-style. (more testing needs to be performed against the
  gradle-style layouts)
  * Gradle-style project layouts need to be created by other means, either
    using the gradle plugin, IDEA, maven, sbt-android or by hand.
  * When using Gradle-style project layouts without properties files
    platformTarget in Android should be set manually to the string name
    of the platform as listed in `android list targets`. Any settings
    normally loaded from a `.properties` file should also be configured
    in the build settings as necessary.
* Consuming apklib and aar artifacts from maven or ivy
* Producing and publishing apklib and aar artifacts to maven or ivy
* Switch to using `com.android.build.AndroidBuilder` for many operations
  to maintain parity with Google's own android build process. Like that used
  in the new Gradle build
* All plugin classes have been moved into the `android` package namespace.
* Simplify configuration with the new `androidBuild`, `androidBuildAar`,
  `androidBuildApklib`, `buildApklib`, and `buildAar` shortcuts located in
  `android.Plugin`

## Description ##

This is a simple plugin for existing and newly created android projects.
It is tested and developed against sbt 0.11.2 and 0.12.3; I have not
verified whether other versions work.

The plugin supports normal android projects and projects that reference
library projects. 3rd party libraries can be included by placing them in
`libs` as in regular projects, or they can be added by using sbt's
`libraryDependencies` feature.

Features not support from the regular android build yet are compiling `NDK`
code. Although `NDK` libraries will be picked up from `libs` as in typical
ant builds.

### Differences from jberkel/android-plugin ###

Why create a new plugin for building android applications?  Because
`jberkel/android-plugin` seems to be pretty difficult to use, and enforces
an sbt-style project layout. Ease of configuration is a primary objective
of this plugin; configuring a project to use this plugin is ~4 lines of
configuration plus a standard android project layout, jberkel's requires
the installation of `conscript`, `g8` and cloning a template project which
has lots of autogenerated configuration. This is incompatible with the
built-in SDK configuration and doesn't load up into Eclipse easily either.

* android-sdk-plugin includes rudimentary device management support,
  jberkel's implementation may be more fully featured.
  * The commands `devices` and `device` are implemented. The former lists
    all connected devices. The latter command is for selecting a target
    device if there is more than one device. If there is more than one
    device, and no target is selected, all commands will execute against the
    first device in the list.
  * `android:install` and `android:run` are tasks that can be used to install
    and run the built apk on device respectively.
* This plugin uses the standard Android project layout as created by
  Eclipse and `android create project`. Additionally, it reads all the
  existing configuration out of the project's `.properties` files.
* `TR` for typed resources improves upon `TR` in android-plugin. It should be
  compatible with existing applications that use `TR` while also adding a
  type to `TypedLayout[A]`. An implicit conversion on `LayoutInflater` to
  `TypedLayoutInflater` allows calling
  `inflater.inflate(TR.layout.foo, container, optionalBoolean)` and receiving
  a properly typed resulting view object.
  * Import `TypedResource._` to get the implicit conversions
* No apklib or aar creation support yet.
* All plugin classes are not namespaced under the `android` package

## Usage ##

1. Install sbt (http://www.scala-sbt.org) or macports, etc.
2. Create a new android project using `android create project` or Eclipse
   * Instead of creating a new project, one can also do
     `android update project` to make sure everything is properly setup
     in an existing project.
   * Instead of keeping local.properties up-to-date, you may set the
     environment variable `ANDROID_HOME` pointing to the path where the
     Android SDK is unpacked. This will bypass the requirement of having
     to run `android update project` on existing projects.
3. Create a directory named `project` within your project and add a file
   `plugins.sbt`, in it, add the following lines:

    ```
    // this directive is not required for sbt 0.11.3 and above
    resolvers += Resolver.url("scala-sbt releases", new URL(
      "http://scalasbt.artifactoryonline.com/scalasbt/sbt-plugin-releases/"))(
      Resolver.ivyStylePatterns)

    addSbtPlugin("com.hanhuy.sbt" % "android-sdk-plugin" % "0.7.4")
    ```

4. Create a file named `build.sbt` in the root of your project and add the
   following lines with a blank line between each:
   * `android.Plugin.androidBuild`
   * `name := YOUR-PROJECT-NAME` (optional, but you'll get a stupid default
     if you don't set it)
   * An example of what build.sbt should look like can be found at
     https://gist.github.com/pfn/5872691

5. Now you will be able to run SBT, some available commands in sbt are:
   * `compile`
     * Compiles all the sources in the project, java and scala
     * Compile output is automatically processed through proguard if there
       are any Scala sources, otherwise; it can be enabled manually.
   * `android:package-release`
      * Builds a release APK and signs it with a release key if configured
   * `android:package-debug`
      * Builds a debug APK and signs it using the debug key
   * `android:package`
     * Builds an APK for the project of the last type selected, by default
       `debug`
   * Any task can be repeated continuously whenever any source code changes
     by prefixing the command with a `~`. `~ android:package-debug`
     will continuously build a debug build any time one of the project's
     source files is modified.
6. If you want android-sdk-plugin to automatically sign release packages
   add the following lines to `local.properties` (or any file.properties of
   your choice that you will not check in to source control):
   * `key.alias: YOUR-KEY-ALIAS`
   * `key.store: /path/to/your/.keystore`
   * `key.store.password: YOUR-KEY-PASSWORD`
   * `key.store.type: pkcs12` (optional, defaults to `jks`)

## Advanced Usage ##

* Consuming apklib and aar artifacts from other projects
  * `import android.Dependencies.{apklib,aar}` to use apklib() and aar()
  * `libraryDependencies += apklib("groupId" % "artifactId" % "version", "optionalArtifactFilename")`
    * Basically, wrap the typical dependency specification with either
      apklib() or aar() to consume the library
    * For library projects in a multi-project build that transitively include
      either aar or apklibs, you will need to add a dependency statement
      into your main-project's settings:
    * `collectResources in Android <<= collectResources in Android dependsOn (compile in Compile in otherLibraryProject)`
    * Alternatively, the `androidBuild()` overload may be used to specify
      all dependency library-projects which should relieve this problem.
* Using the google gms play-services aar:

    ```
    resolvers <+= (sdkPath in Android) { p =>
      "gms" at (file(p) / "extras" / "google" / "m2repository").toURI.toString
    },
    libraryDependencies +=
      aar("com.google.android.gms" % "play-services" % "3.1.36")
    ```

* Generating apklib and/or aar artifacts
  * To specify that your project will generate and publish either an `aar`
    or `apklib` artifact simply change the `android.Plugin.androidBuild`
    line to one of the variants that will build the desired output type.
    * For `apklib` use `android.Plugin.androidBuildApklib`
    * For `aar` use `android.Plugin.androidBuildAar`
  * Alternatively, use `android.Plugin.buildAar` and/or
    `android.Plugin.buildApklib` in addition to any of the variants above
    * In build.sbt, add `android.Plugin.buildAar` and/or
      `android.Plugin.buildApklib` on a new line.
    * It could also be specified, for example, like so:
      `android.Plugin.androidBuild ++ android.Plugin.buildAar`
* Multi-project builds
  * Several documented examples can be found at the following links, they
    cover a variety of situations, from multiple java projects, to mixed
    java and android projects and fully scala projects.
    * https://gist.github.com/pfn/5872427
    * https://gist.github.com/pfn/5872679
    * https://gist.github.com/pfn/5872770
  * See [Working with Android library projects](https://github.com/pfn/android-sdk-plugin/wiki/Working-with-Android-library-projects) 
    in the Wiki for detailed instructions on configuring Android library
    projects
* Configuring `android-sdk-plugin` by editing build.sbt
  * `import android.Keys._` at the top to make sure you can use the plugin's
    configuration options
  * Add configuration options according to the sbt style:
    * `useProguard in Android := true` to enable proguard. Note that if you don't use proguard, you *must* specify uses-library on a pre-installed scala lib on-device. Dexing the scala libs is not supported at this time.
  * Configurable keys can be discovered by typing `android:<tab>` at the
    sbt shell
* Configuring proguard, some options are available
  * `proguardOptions in Android += Seq("-dontobfuscate", "-dontoptimize")` -
    will tell proguard not to obfuscute nor optimize code (any valid proguard
    option is usable here)
 * `proguardConfig in Android ...` can be used to replace the entire
   proguard config included with android-sdk-plugin
* I have found that Scala applications on android build faster if they're
  using scala 2.8.2. Set the scala version in `build.sbt` by entering
  `scalaVersion := "2.8.2"`
*  Unit testing with robolectric, see my build.scala for this configuration:
  * https://gist.github.com/pfn/5872909
  * This example is somewhat old and may include settings that are no longer
    necessary, this project hasn't been touched in nearly a year.
  * To get rid of robolectric's warnings about not finding certain classes
    to shadow, change the project target to include google APIs
  * jberkel has written a Suite trait to be able to use robolectric with
    scalatest rather than junit, see https://gist.github.com/2662806

### TODO / Known Issues ###

* Implement the NDK build process
* Changes to `AndroidManifest.xml` will require the plugin to be reloaded.
  The manifest data is stored internally as read-only data and does not
  reload automatically when it is changed. The current workaround is to
  type `reload` manually everytime `AndroidManifest.xml` is updated.
* Adding new resource files without updating existing resource files will
  result in `resources.ap_` not being rebuilt. Symptom: weird crashes
  around resource inflation. Workaround: clean or touch an existing resource
  file.
* sbt `0.12` and `0.13` currently have a bug where jars specified in javac's
  -bootclasspath option forces a full rebuild of all classes everytime. sbt
  `0.12.3` and later has a hack that should workaround this problem. The
  plugin sets the system property `xsbt.skip.cp.lookup` to `true` to bypass
  this issue; this disables certain incremental compilation checks, but should
  not be an issue for the majority of use-cases.

#### Thanks to ####

* pfurla, jberkel, mharrah, retronym and the other folks from #sbt and #scala
  for bearing with my questions and helping me learn sbt.
