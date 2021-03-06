How To Run various testsuite configurations
===========================================

## Base steps

It's recomended to build the workspace including distribution.

    
    cd $KEYCLOAK_SOURCES
    mvn clean install -DskipTests=true
    cd distribution
    mvn clean install
    

## Debugging - tips & tricks

### Arquillian debugging

Adding this system property when running any test:

    
    -Darquillian.debug=true
    
will add lots of info to the log. Especially about:
* The test method names, which will be executed for each test class, will be written at the proper running order to the log at the beginning of each test (done by KcArquillian class). 
* All the triggered arquillian lifecycle events and executed observers listening to those events will be written to the log
* The bootstrap of WebDriver will be unlimited. By default there is just 1 minute timeout and test is cancelled when WebDriver is not bootstrapped within it.

### WebDriver timeout

By default, WebDriver has 10 seconds timeout to load every page and it timeouts with error after that. Use this to increase timeout to 1 hour instead:

    
    -Dpageload.timeout=3600000
    
    
### Surefire debugging

For debugging, the best is to run the test from IDE and debug it directly. When you use embedded Undertow (which is by default), then JUnit test, Keycloak server 
and adapter are all in the same JVM and you can debug them easily. If it is not an option and you are forced to test with Maven and Wildfly (or EAP), you can use this:
 
   
    -Dmaven.surefire.debug=true
   
   
and you will be able to attach remote debugger to the test. Unfortunately server and adapter are running in different JVMs, so this won't help to debug those. 

TODO: Improve and add more info about Wildfly debugging...

## Testsuite logging

It is configured in `testsuite/integration-arquillian/tests/base/src/test/resources/log4j.properties` . You can see that logging of testsuite itself (category `org.keycloak.testsuite`) is debug by default.

When you run tests with undertow (which is by default), there is logging for Keycloak server and adapter (category `org.keycloak` ) in `info` when you run tests from IDE, but `off` when 
you run tests with maven. The reason is that, we don't want huge logs when running mvn build. However using system property `keycloak.logging.level` will override it. This can be used for both IDE or maven.
So for example using `-Dkeycloak.logging.level=debug` will enable debug logging for keycloak server and adapter. 

For more fine-tuning of individual categories, you can look at log4j.properties file and temporarily enable/disable them here.

TODO: Add info about Wildfly logging

## Run adapter tests

### Wildfly

    
    # Prepare servers
    mvn -f testsuite/integration-arquillian/servers/pom.xml clean install \
       -Pauth-server-wildfly \
       -Papp-server-wildfly

    # Run tests
    mvn -f testsuite/integration-arquillian/tests/other/adapters/jboss/wildfly/pom.xml \
       clean install \
       -Pauth-server-wildfly \
       -Papp-server-wildfly
    

### JBoss Fuse 6.3

1) Download JBoss Fuse 6.3 to your filesystem. It can be downloaded from http://origin-repository.jboss.org/nexus/content/groups/m2-proxy/org/jboss/fuse/jboss-fuse-karaf 
Assumed you downloaded `jboss-fuse-karaf-6.3.0.redhat-229.zip`

2) Install to your local maven repository and change the properties according to your env (This step can be likely avoided if you somehow configure your local maven settings to point directly to Fuse repo):


    mvn install:install-file \
      -DgroupId=org.jboss.fuse \
      -DartifactId=jboss-fuse-karaf \
      -Dversion=6.3.0.redhat-229 \
      -Dpackaging=zip \
      -Dfile=/mydownloads/jboss-fuse-karaf-6.3.0.redhat-229.zip


3) Prepare Fuse and run the tests (change props according to your environment, versions etc):


    # Prepare Fuse server
    mvn -f testsuite/integration-arquillian/servers \
      clean install \
      -Pauth-server-wildfly \
      -Papp-server-fuse63 \
      -Dfuse63.version=6.3.0.redhat-229 \
      -Dapp.server.karaf.update.config=true \
      -Dmaven.local.settings=$HOME/.m2/settings.xml \
      -Drepositories=,http://download.eng.bos.redhat.com/brewroot/repos/sso-7.1-build/latest/maven/ \
      -Dmaven.repo.local=$HOME/.m2/repository
 
    # Run the Fuse adapter tests
    mvn -f testsuite/integration-arquillian/tests/other/adapters/karaf/fuse63/pom.xml \
      clean install \
      -Pauth-server-wildfly \
      -Papp-server-fuse63 \
      -Dfuse63.version=6.3.0.redhat-229


### EAP6 with Hawtio

1) Download JBoss EAP 6.4.0.GA zip

2) Install to your local maven repository and change the properties according to your env (This step can be likely avoided if you somehow configure your local maven settings to point directly to EAP repo):


    mvn install:install-file \
      -DgroupId=org.jboss.as \
      -DartifactId=jboss-as-dist \
      -Dversion=7.5.0.Final-redhat-21 \
      -Dpackaging=zip \
      -Dfile=/mydownloads/jboss-eap-6.4.0.zip


3) Download Fuse EAP installer (for example from http://origin-repository.jboss.org/nexus/content/groups/m2-proxy/com/redhat/fuse/eap/fuse-eap-installer/6.3.0.redhat-220/ )

4) Install previously downloaded file manually


    mvn install:install-file \
      -DgroupId=com.redhat.fuse.eap \
      -DartifactId=fuse-eap-installer \
      -Dversion=6.3.0.redhat-220 \
      -Dpackaging=jar \
      -Dfile=/fuse-eap-installer-6.3.0.redhat-220.jar


5) Prepare EAP6 with Hawtio and run the test


    # Prepare EAP6 and deploy hawtio
    mvn -f testsuite/integration-arquillian/servers \
      clean install \
      -Pauth-server-wildfly \
      -Papp-server-eap6-fuse \
      -Dapp.server.jboss.version=7.5.0.Final-redhat-21 \
      -Dfuse.installer.version=6.3.0.redhat-220
 
    # Run the test
    mvn -f testsuite/integration-arquillian/tests/other/adapters/jboss/eap6-fuse/pom.xml \
      clean install \
      -Pauth-server-wildfly \
      -Papp-server-eap6-fuse  
 

## Migration test

### DB migration test

This test will:
 - start Keycloak 1.9.8
 - import realm and some data to MySQL DB
 - stop Keycloak 1.9.8
 - start latest KEycloak, which automatically updates DB from 1.9.8
 - Do some test that data are correct
 

1) Prepare MySQL DB and ensure that MySQL DB is empty. See [../../misc/DatabaseTesting.md](../../misc/DatabaseTesting.md) for some hints for locally prepare Docker MySQL image.

2) Run the test (Update according to your DB connection, versions etc):


    export DB_HOST=localhost

    mvn -f testsuite/integration-arquillian/pom.xml \
      clean install \
      -Pauth-server-wildfly,jpa,clean-jpa,auth-server-migration \
      -Dtest=MigrationTest \
      -Dmigration.mode=auto \
      -Dmigrated.auth.server.version=1.9.8.Final \
      -Djdbc.mvn.groupId=mysql \
      -Djdbc.mvn.version=5.1.29 \
      -Djdbc.mvn.artifactId=mysql-connector-java \
      -Dkeycloak.connectionsJpa.url=jdbc:mysql://$DB_HOST/keycloak \
      -Dkeycloak.connectionsJpa.user=keycloak \
      -Dkeycloak.connectionsJpa.password=keycloak


### JSON export/import migration test
This will start latest Keycloak and import the realm JSON file, which was previously exported from Keycloak 1.9.8.Final
  

    mvn -f testsuite/integration-arquillian/pom.xml \
      clean install \
      -Pauth-server-wildfly,migration-import \
      -Dtest=MigrationTest \
      -Dmigration.mode=import \
      -Dmigrated.auth.server.version=1.9.8.Final



## Social Login
The social login tests require setup of all social networks including an example social user. These details can't be 
shared as it would result in the clients and users eventually being blocked. By default these tests are skipped.
   
To run the full test you need to configure clients in Google, Facebook, GitHub, Twitter, LinkedIn, Microsoft and 
StackOverflow. See the server administration guide for details on how to do that. Further, you also need to create a 
sample user that can login to the social network.
 
The details should be added to a standard properties file. For some properties you can use shared common properties and
override when needed. Or you can specify these for all providers. All providers require at least clientId and 
clientSecret (StackOverflow also requires clientKey).
 
An example social.properties file looks like:

    common.username=sampleuser@example.org
    common.password=commonpassword
    common.profile.firstName=Foo
    common.profile.lastName=Bar
    common.profile.email=sampleuser@example.org

    google.clientId=asdfasdfasdfasdfsadf
    google.clientSecret=zxcvzxcvzxcvzxcv

    facebook.clientId=asdfasdfasdfasdfsadf
    facebook.clientSecret=zxcvzxcvzxcvzxcv
    facebook.profile.lastName=Test

In the example above the common username, password and profile are shared for all providers, but Facebook has a 
different last name.

Some providers actively block bots so you need to use a proper browser to test. Either Firefox or Chrome should work.

To run the tests run:

    mvn -f testsuite/integration-arquillian/pom.xml \
          clean install \
          -Pauth-server-wildfly \
          -Dtest=SocialLoginTest \
          -Dbrowser=chrome \
          -Dsocial.config=/path/to/social.properties


## Different Browsers
 
To run with Chrome add `-Dbrowser=chrome`. Depending on the Chrome version you have you may need to download the latest
chromedriver from https://sites.google.com/a/chromium.org/chromedriver/downloads and point to it with 
`-Dwebdriver.chrome.driver=/path/to/chromedriver`.
 
    