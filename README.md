# <a name="intro"></a>java-stix-validator

A microservice and Single Page Application for validating Structured Threat Information eXpression (STIX™) 
documents.

java-stix-validator is being developed under the official [STIXProject][stix project].

[![Build Status](https://api.travis-ci.org/STIXProject/java-stix-validator.svg)](https://travis-ci.org/STIXProject/java-stix-validator) [![Heroku](http://heroku-badge.herokuapp.com/?app=agile-journey-9583&style=flat)](http://agile-journey-9583.herokuapp.com)

## <a name="overview"></a>Overview

The validator uses java-stix to validate STIX documents. As java-stix is not a one-for-one replacement 
for [python-stix][python-stix] neither is [python-based stix-validator][stix-validator].  Meaning: this 
validator does not validate for best practices, and may pass documents [python-based stix-validator][stix-validator] 
may fail. The validator supports v1.1.1 and v1.2.0 releases of the STIX schema, and can 
be modified to support follow-on releases.

A [Heroku hosted instance][heroku instance] of the validator is available for
you to use, or you can [clone, build, and run an instance your own box](#building).
Optionally, you can also create and run a Docker container using the project's Dockerfile.

## <a name="versioning"></a>Versioning

Releases of java-stix-validator will comply with the Semantic Versioning specification 
at [http://semver.org/][semver]. Java-stix-validator is currently under active development; 
see TODO.txt (if there is one) for a tentative roadmap.

## <a name="question"></a> Got a Question or Problem?
If you have questions about how to use java-stix-validator, please direct these to  [stix@mitre.org](mailto:stix@mitre.org).

## <a name="contribute"></a>Contributing to java-stix-validator?

We'd welcome you contributing to java-stix-validator. Read the [CONTRIBUTING.MD](CONTRIBUTING.MD) for guidlines.

## <a name="how-this-works"></a>How this works

The microservice backend in written in the [Spark Framework ][sparkjava], a Sinatra-like 
lightweight Java web framework, and the front-end Single Page Application (SPA) is written in 
[AngularJS][AngularJS], a JavaScript MVW framework.  The SPA's 3rd party runtime JavaScipt and 
stylesheet dependencies are gathered and staged by at build time by [Bower][bower] and gradle. 
Gradle at build time will install Bower for you using [Node and npm][node and npm].  Both of 
these installs are handled within the confines of the project. Also since Spark includes an 
embedded instance of [Jetty][Jetty], a Web server and javax.servlet container, you will have 
everything you need to run the validator locally.

The validator's microservice back-end uses Java Reflection to load multiple releases of 
java-stix as resource files at the initialization of the ValidationService into 
individual, isolated classloaders:

```
stixSchemas = new HashMap<Version, Object>();

for (Version version : versions) {
  try {

    ResourcePatternResolver patternResolver = new PathMatchingResourcePatternResolver();

    Resource[] resources = patternResolver
        .getResources("classpath:reflection-libs/v" + version
            + "/**/*.jar");

    ArrayList<URL> urls = new ArrayList<URL>();

    // work around for URLClassLoader's inability to retrieve
    // classes from Jars contained within Jars as resources
    for (Resource resource : resources) {
      ReadableByteChannel readableByteChannel = Channels
          .newChannel(resource.getURL().openStream());
      File tempFile = File.createTempFile("validator-", null);
      FileOutputStream fileOutputStream = new FileOutputStream(
          tempFile);
      fileOutputStream.getChannel().transferFrom(
          readableByteChannel, 0, Long.MAX_VALUE);
      fileOutputStream.close();
      urls.add(tempFile.toURI().toURL());
    }

    @SuppressWarnings({ "resource" })
    ClassLoader classLoader = new URLClassLoader(
        urls.toArray(new URL[urls.size()]));

    Class<?> cls = classLoader
        .loadClass("org.mitre.stix.STIXSchema");

    @SuppressWarnings("rawtypes")
    Constructor[] constructors = cls.getDeclaredConstructors();
    constructors[0].setAccessible(true);
    Object instance = constructors[0].newInstance();

    Method setValidationErrorHandlerMethod = instance.getClass()
        .getMethod("setValidationErrorHandler",
            ErrorHandler.class);

    setValidationErrorHandlerMethod.invoke(instance,
        new Object[] { null });

    System.out.println("Created STIXSchema for v " + version
        + " instance");

    stixSchemas.put(version, instance);

  } catch (ClassNotFoundException | SecurityException
      | IllegalAccessException | IllegalArgumentException
      | InvocationTargetException | IOException
      | InstantiationException | NoSuchMethodException e) {

    throw new RuntimeException(e);
  }
```

to permit the the validator's microservice to validate documents across the most 
recent release of the STIX schema version:

```
for (Version knownVersion : stixSchemas.keySet()) {
  if (lookForVersion.equals(knownVersion)) {
    Object obj = stixSchemas.get(knownVersion);

    Method validateMethod;
    try {
      validateMethod = obj.getClass().getMethod("validate",
        String.class);
      validates = (boolean) validateMethod.invoke(obj,
        xmlText);
      break;

    } catch (InvocationTargetException e) {

``` 

#<a name="building"></a>Building

java-stix-validator builds under Java8, and will not build under a 
prior Java release.

## <a name="cloning"></a>Clone the repository

Clone the java-stix-validator project via:

	git clone https://github.com/nemonik/java-stix-project.git

### <a name="gradle_wrapper"></a>Using the Gradle Wrapper to build

I'd advise using the Gradle command-line as the project comes with a Gradle 
Wrapper, a batch script on Windows, and a shell script for UNIX platforms 
including OS X thereby removing the need to have Gradle installed. 

For a UNIX platform, you can run Gradle from the project root via:

	./gradlew stage

This documentation centers on UNIX-centric Gradle command-line execution for 
brevity, but if you are on Windows you can run Gradle via:

	.\gradlew.bat stage

<a name="Running"></a>Running

Once built the validator can be run via:

	./build/install/java-stix-validator/bin/java-stix-validator

The shell output will look like the following:

	➜  java-stix-validator git:(master) ✗ ./build/install/java-stix-validator/bin/java-stix-validator
	Created STIXSchema for v 1.1.1 instance
	Created STIXSchema for v 1.2.0 instance
	[Thread-1] INFO spark.webserver.SparkServer - == Spark has ignited ...
	[Thread-1] INFO spark.webserver.SparkServer - >> Listening on 0.0.0.0:8080
	[Thread-1] INFO org.eclipse.jetty.server.Server - jetty-9.0.2.v20130417
	[Thread-1] INFO org.eclipse.jetty.server.ServerConnector - Started ServerConnector@4ab75171{HTTP/1.1}{0.0.0.0:8080}

Once, started open [http://localhost:8080][localhost] in your Web browser of choice.

Optionally, you can run the validator at a port other than the default of `8080`
by setting an environmental variable like so:

	export PORT=9090
	./build/install/java-stix-validator/bin/java-stix-validator

### <a name="building_via_docker"></a>Building via the Dockerfile

Alternatively, if you have Docker installed, from the root of the project run 
the following to create a container image for this Dockerfile via:

	docker build -rm -t stix/validator .

Then create a container using the image you just created via:

	docker run -p 8080:8080 -it --rm --name validator stix/validator

If you're using boot2docker, you will need to access via VM’s host only interface IP 
address:

	boot2docker ip

And then point your browser to http://$(boot2docker ip):8080 vice https://localhost:8080

## <a name="using"></a>Using

Use the tab to select the means for submitting your STIX document(s) for validation.

The first tab accepts URLs to retrieve and validate STIX documents.  With the second 
you can cut-and-paste the whole text of STIX document and submit for validation.  In 
the third tab, you can drag and drop one or more STIX documents at a time into the 
dashed-line box for validation.

[heroku instance]:http://agile-journey-9583.herokuapp.com
[bower]: http://bower.io/
[node and npm]: https://nodejs.org/
[Jetty]: http://www.eclipse.org/jetty/
[AngularJS]: https://angularjs.org/
[sparkjava]: http://sparkjava.com/
[localhost]: http://localhost:8080
[python-stix]: https://github.com/STIXProject/python-stix
[stix-validator]: https://github.com/STIXProject/stix-validator
[list]: https://stix.mitre.org/community/registration.html
[stix project]: http://stixproject.github.io/
[stix]: http://stix.mitre.org/
[semver]: http://semver.org/
