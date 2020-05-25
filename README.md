# JMC Overrides
This repository contains build scripts and file overrides required to build
the OpenJDK Mission Control on AdoptOpenJDK. The overrides consist of the main
`Jenkinsfile` that controls the actual build process as basically described within
the [JMC README](https://github.com/openjdk/jmc/blob/master/README.md)

## Build procedure
The build runs thru the following steps:

- Checkout of the overrides project itself
- Checkout the JMC project for the correct release or branch
- Replace of the root `pom.xml` in order to re-enable the default maven deployment 
  behavior and possibility to define alternative deployment repositories for the core
  libraries
- Replacement of the RCP application `updatesites.properties` in order for the correct
  update sites
- Replacement of the IDE/RCP update site `index.html` files to point to the correct
  AdoptOpenJDK sites.
- Build of the core libraries first
- Start of a Jetty P2 site used for the main build part
- Build IDE and RCP parts
- Deploy all artifacts to the AdoptOpenJDK Artifactory instance
 
## Update of overridden files
The override files will be kept up to date by manually comparing those with the original
versions on the main JMC repository keeping the intended changes