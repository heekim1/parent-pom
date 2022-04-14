require wget, mvn 3.5+, java, docker 1.12.6+, paste, awk, curl

# Creating Debian package for a 3rd party tool

## Workflow

![Flow Chart](BfxDockertoBelmontShell.png)

## Create a new maven project

1. Set ```ctdna-parent-pom``` as parent POM and ```deb``` as packaging.
2. Add plugin to download source code to ```target/checkout```, or binary to ```target/bin```.
    1. Use ```maven-scm-plugin``` to clone/checkout source code from git/svn.  Repo URL is set in the ```scm``` section and you can specify a branch or tag to download.  Reference ```twoBitToFa``` for sample code.
    2. Use ```maven-antrun-plugin``` to download source code or binaries using ftp.  This plugin allows you to hardcode the username/password in the POM file directly (most likely it's anonymous).  Reference ```blast``` for sample code.
    3. Use ```download-maven-plugin``` to download using http/https.  Reference ```perl``` for sample code.
    4. Use ```maven-dependency-plugin``` to download binaries published in Nexus (e.g. a jar file).  Be sure to add the dependency to ```dependencies``` section of ```pom.xml```.  Reference ```htsjdk``` for sample code.
3. Use ```exec-maven-plugin``` to compile the code.  Code should be compiled in the ```dev-base``` container (which includes all necessary libraries).  The container should also bind mount to ```target/checkout``` for source code location and ```target/bin``` for output.  Create a ```build.sh``` to execute the actual compilation and copy the final output file if necessary.  Reference ```twoBitForFA```, ```blast``` or ```perl``` for sample code.
    4. make sure the final binaries are placed under ```/target/bin```.  ```maven-antrun-plugin``` can copy files and change permission.
4. Create ```src/deb/DEBIAN/control.gsp```.  This is the [control file](https://www.debian.org/doc/manuals/maint-guide/dreq.en.html#control) to build a Debian package.
    4. Make sure ```Package``` is set to artifact ID dash version, e.g. ```twoBitToFa-3.4.5```.
    4. Also, make sure the last line in ```Description``` has an extra new line.
    4. ```Depends``` should also be set, so the necessary dependency will be pulled in.  Reference ```ctdna-fusion``` or ```twoBitForFa``` for sample code.
    5. The ```ctdna-parent-pom``` will detect if the control file exists and build the Debian package.
    6. The binaries in ```target/bin``` will be automatically packaged in a Debian package under ```/bfx/bin/<artifactId>/<version>```.

## Build Debian package

1. Run ```mvn package``` or ```mvn install``` for java based project to build the jar file and run junit test.  If ```test``` directory has bats files, it'll run those tests too.  specify ```maven.test.skip``` to skip test.
1. Run ```mvn test``` to run java tests and/or bats tests.
1. Run ```mvn package -Ppack``` or ```mvn install -Ppack``` to also package the Debian package if ```build/debian/control.gsp``` exists and Docker image if ```build/docker/Dockerfile.gsp``` exists.
1. Run ```mvn deploy -Dprod``` for production build, which will also turn on packaging souce debian package on 3rd party modules.

# Create Debian package and Docker image for ctDNA workflow

1. Set ```ctdna-parent-pom``` as parent POM and ```deb``` as packaging.
1. Use ```maven-scm-plugin``` to clone source code from git.
1. Create ```src/deb/DEBIAN/control.gsp``` to package the workflow itself into a Debian package.
1. Add all required 3rd party tools to the ```dependencies``` section in the POM, make sure they have type ```deb```.
1. Create ```test/docker/Dockerfile.gsp```
    2. The ```ctdna-parent-pom``` will detect this file and copy all dependencies defined in the ```dependencies``` section and add the deb file of the current project to ```target/dependency```.  Then it will generate the ```Packages.gz``` file required for Debian to add the directory as a local apt repo.
    2. Add ```/tmp/deb``` as apt repo, execute ```apt-get update``` and installed each modules required with ```--allow-unauthenticated``` flag.
    2. Set ```USER``` to ```runner```.  (the user is created in the ```ctdna/ubuntu:18.04``` image.)
    2. Setup ```PATH```, ```LD_LIBRARY_PATH``` and ```PERL5LIB```. Reference ```ctdna-fusion``` for sample code.
1. Create ```.maven-dockerignore```, so the ```docker-maven-plugin``` will not try to add itself to the docker build env.  Just copy the one in ```ctdna-fusion``` will do.

## Build Debian package and Docker Image

1. Add a ```mirror``` section to ```.m2/settings.xml```

   ```xml
<profiles>
    <profile>
      <repositories>
        <repository>
          <snapshots>
            <enabled>false</enabled>
          </snapshots>
          <id>central</id>
          <name>default-maven-virtual</name>
          <url>https://kimh89.jfrog.io/artifactory/default-maven-virtual</url>
        </repository>
        <repository>
          <snapshots />
          <id>snapshots</id>
          <name>default-maven-virtual</name>
          <url>https://kimh89.jfrog.io/artifactory/default-maven-virtual</url>
        </repository>
      </repositories>
      <pluginRepositories>
        <pluginRepository>
          <snapshots>
            <enabled>false</enabled>
          </snapshots>
          <id>central</id>
          <name>default-maven-virtual</name>
          <url>https://kimh89.jfrog.io/artifactory/default-maven-virtual</url>
        </pluginRepository>
        <pluginRepository>
          <snapshots />
          <id>snapshots</id>
          <name>default-maven-virtual</name>
          <url>https://kimh89.jfrog.io/artifactory/default-maven-virtual</url>
        </pluginRepository>
      </pluginRepositories>
      <id>artifactory</id>
    </profile>
  </profiles>
  <activeProfiles>
    <activeProfile>artifactory</activeProfile>
  </activeProfiles>
   ```
2. Run ```mvn install```.

## Publish to Docker Registry and Nexus

1. Add a ```server``` section to ```.m2/settings.xml```

   ```xml
   <servers>
     <server>
       <id>bfx-releases</id>
       <username>bfxdeploy</username>
       <password>---</password>
     </server>
   </servers>
   ```
1. Similarly, a ```server``` section for Docker and Debian
   
   ```xml
   <servers>
     <server>
       <id>kimh89.jfrog.io</id>
       <username>bfxonco</username>
       <password>----</password>
     </server>
     <server>
       <id>kimh89.jfrog.io</id>
       <username>bfxonco</username>
       <password>----</password>
     </server>
   </servers>
   ```
2. Execute ```mvn deploy```


# Flow Chart

![Flow Chart](bfx-maven.png)

# Other Useful Commands

1. ```mvn install -Dpack -Dskip.deb.src=false``` to package source code in a Debian package.

2. ```mvn deploy -Dskip.docker.push=true``` to skip pushing to docker registry

3. ```mvn install -Dskip.packaging``` to skip creating deb package

3. ```mvn deploy -Dprod``` to package debian and debian source packages, docker image and publish to Jfrog, artifactory (deb repo) and artifactory (docker repo).

# Debian Control Files

## Template

1. It's a groovy template.
1. The ```control``` file for the binary debian package is ```src/src-deb/DEBIAN/control.gsp``` and source debian package is ```src/deb/DEBIAN/control.gsp```.
2. Both can reference to the project name (artifactId) and version as ```$name``` and ```$version``` respectively.  ```Package: $name-$version``` on project picard will rendered as ```Package: picard-2.9.0```.
3. A more complex example, ```<% out << name.toUpperCase().replace('-','_') %>_PATH=/bfx/bin/$name/$version``` in onco-ngs-utils will rendered as ```ONCO_NGS_UTILS_PATH=/bfx/bin/onco-gns-utils/1.0.0```.
4. note that any ```$``` has to be escaped (prefixed with ```\```)

## Environment Variables

Since the Debian packages are not installed in common locations like ```/usr/bin``` or ```/usr/lib```, to use the Debian packages, correct environment variables, like ```PATH``` or ```LD_LIBRARY_PATH```, have to be set.

1. Each Debian package's control file should have a ```Environment``` field.  It's not a standard Debian control file field, but Debian seems not to bother by it and will simply display it with all standard tools (```dpkg-deb```, ```apt-cache``` and ```/var/lib/dpkg/status```)

   ```
   Package: $name-$version
   Version: 1.0.0
   Section: base
   Priority: optional
   Architecture: all
   Depends: perl-5.22.2, twoBitToFa-3.4.5, blast-2.2.30, samtools-1.2
   Maintainer: Your Name <you@email.com>
   Environment:
    PATH=/bfx/bin/$name/$version:\$PATH
   Description: ctdna-fusion
   ```
2. ```ctdna/ubuntu:18.04``` has a global ```.bashrc``` as well as ```profile``` which will parse the ```/var/lib/dpkg/status``` file and populate the environment with all the settings under ```Environment```.  Thus, there's no need to set any environment variables (```ENV```) in the ```src/docker/Dockerfile.gsp```. Note that the ```Environment``` settings are defined in the ```src/debian/control.gsp``` file as shown in the example above.
3. ```ctdna-parent-pom``` will also generate an ```environment file``` it's the project has a ```src/docker/Dockerfile.gsp```

   ```sh
   cat target/ctdna-fusion-1.0.0.env
   export  PATH=/bfx/bin/ctdna-fusion/1.0.0:$PATH;
   export  PATH=/bfx/bin/perl/5.22.2/bin:$PATH;
   export  LD_LIBRARY_PATH=/bfx/bin/perl/5.22.2/lib/5.22.2/x86_64-linux/CORE/:$LD_LIBRARY_PATH;
   export  PERL5LIB=/bfx/bin/perl/5.22.2/lib/5.22.2;
   ```

   it can be pulled down in maven

   ```xml
   <dependency>
     <groupId>com.kimh89.dev</groupId>
     <artifactId>ctdna-fusion</artifactId>
     <version>1.0.0</version>
     <type>text</type>
     <classifier>env</classifier>
   </dependency>
   ```

4. The ```environment file``` is generated from the maven dependency, it doesn't check the ```Depends``` field in the ```control.gsp``` file.  Since the goal is to have multiple versions of the same tool coexists, it is possible that a tool depends on R-3.2.0 while another tool depends on R-3.3.0 to be pulled in together.  The ```environment file``` generated will have one preceding another, ended up only one version of R is used.  To avoid this problem,

  1. branch out one of the tools so it depends on the same version of R (ref Handling multiple builds of the same version); OR
  2. only put the language dependency (like R, java) on the top level ctdna-* projects.



# Handling multiple builds of the same version

When multiple build options for the same tools is required, branch out and name the ```pom.xml``` and ```control.gsp``` with a suffix to indicate it's a variation of the same tool/version.
