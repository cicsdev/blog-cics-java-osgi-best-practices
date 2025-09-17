
OSGi demystified: don't bungle your bundles  
============================================

Author: Ivan Hargreaves, CICS Java and Liberty Architect. Designer and Senior Developer of JVM
server technologies.


*'OSGi Demystified' is a series of articles addressing common OSGi
issues in CICS. We offer insight into OSGi, discuss best practices, and
provide setup and configuration advice. This article is a round up of OSGI best practices.*


## 1. Using common OSGi bundles

Inevitably, at some point in larger applications, you'll realise that
some of your OSGi bundles provide common services to a number of
applications. Or that you use third-party libraries in multiple
applications. Either way, you may find the OSGI_BUNDLES option of the
JVM profile and concluded that it allows you to install a list of
common/library OSGi bundles. The OSGi bundles in that list are
automatically installed when the OSGi JVM server goes enabled. However,
you may also have realised that the same OSGi bundles can be installed
as CICS bundle parts in a CICS bundle. So what's the difference?

Ultimately both routes achieve the same result, so it is a matter of
preference and architecture. By adding the OSGi bundles to the
OSGI_BUNDLES option you have a simpler mechanism to install libraries
that does not require CICS bundle packaging. The library bundles are
guaranteed to be available by the time the first application runs, and
they are guaranteed to be unchanged for the duration of the JVM server's
enable status. Conversely, an OSGi bundle installed using a CICS bundle
provides more flexibility, an independent life-cycle (managed by the
CICS bundle) and the potential to be updated at runtime. However, the
install ordering of CICS bundles relies on *you* to ensure that
libraries are installed before applications. It also requires a more
complex deployment process to ensure the libraries are installed
regardless of the CICS start-up procedure -- whether that be CSD group,
warm restarts, SPI or initial start.

We often refer to the OSGI_BUNDLES installed libraries as middleware
bundles. If you are confident these OSGi bundles will not need to be
dynamically updated or to have independent life-cycling, then use of
this option can reduce the complexity of deployment and the burden of
ordering the install sequence.

For more information: [JVM server profile
options](https://www.ibm.com/support/knowledgecenter/SSGMCP_5.4.0/configuring/java/dfha2_jvmprofile_server_options.html)

## 2. Properties files and OSGi

A common approach to configuring applications is to use a properties
file. In the most basic case, a properties file resides in a hard-coded
location outside of your application. A refinement to that basic
approach is to pick up a system property that instructs your application
where to find the properties file. Despite offering a little more
flexibility, the configuration is still entirely separate from your
application -- and can lead to errors.

```
    File file = new File("/u/ibm/test.properties");
    FileInputStream fileInput = new FileInputStream(file);
    Properties properties = new Properties();
    properties.load(fileInput);
    fileInput.close();
```

A natural evolution for OSGi is to place the properties file inside the
OSGi bundle so that the application and configuration remain together.
However, by placing the properties file inside the bundle, you are no
longer in control of the actual file-system location of the file when
the bundle is installed. To combat this problem, you can use use the
bundle's classloader in conjunction with the getResourceAsStream()
method to find and load the properties, as shown below:

```
    InputStream inputStream = getClass().getResourceAsStream("/test.properties");
    Properties props = new Properties();
    props.load(inputStream);
```

Ultimately you might wish to look at using the OSGi Configuration Admin
service. The OSGi Configuration Admin service defines a mechanism for
managing, creating, and passing configuration settings to an OSGi
bundle. Put simply, configuration is a list of name-value pairs which
can be created and managed by a *management* application. The
*management* application, perhaps a GUI front-end for configuring
properties, creates or updates the configuration and passes it to the
Configuration Admin Service. The Configuration Admin Service acts like a
central hub and persists/distributes the configuration to interested
parties. Interested parties, such as your OSGi bundle, register as
ManagedService services and declare their interest in the configuration
(via a unique ID).

Further reading:

-   [Configuration Admin Service
    explained](https://osgilook.wordpress.com/2009/03/22/configuration-admin-service-explained-the-managedservice-interface/)
-   [Configuring OSGi declarative
    services](http://blog.vogella.com/2016/09/26/configuring-osgi-declarative-services/)

## 3. Convert to OSGi bundle -- or use Bundle-ClassPath?

Migrating to OSGi isn't always straightforward and there are *design/practicality* trade-offs. For example, should you place existing JARs on an OSGi bundle-classpath and wrap them inside the OSGi bundle? or should you convert those JARs to OSGi bundles in their own right? As explained earlier in this blog series, each OSGi bundle has it's own classloader. By using the `Bundle-ClassPath` header in the MANIFEST.MF of an OSGi bundle you can add other JARs and classes to that bundle's class path. The JARs and classes become part of the bundle. One potential drawback of this approach is that those JARs lose version independence and granularity and they are no longer eligible to be shared libraries. However, there are situations where this approach is the prudent choice. Ultimately it is a design choice. Our rule of thumb is this -- for lightweight libraries with few dependencies, where the library is unlikely to be used by other OSGi bundles in the system, Bundle-ClassPath is fine. You can even chose to export its packages through your own OSGi bundle should you desire. Where a library is large, has other dependencies, many consumers, or might be versioned within a system, installing it as an OSGi bundle in its own right is the way to go. This can reduce the overall footprint of the system, promote reuse, and reduce both complexity and duplication.

##  4. Avoid Require-Bundle

Using the OSGi bundle MANIFEST.MF header **Require-Bundle** is to be avoided where possible, it ties you to a specific bundle implementation. You also lose package granularity because you are forced to consume other packages from the same "required" bundle. If those packages are not your preferred versions of an implementation then very quickly your system can become constrained and inflexible. A more fine-grained approach with Import-Package is recommended because it allows you to pick and choose each package/version individually.    

```
        Require-Bundle: com.ibm.multiple.packages        
        Import-Package: com.ibm.specific.package;version="[1.0.0, 2.0.0)"        
```
## 5. Don't split your packages

A *split package* is the situation where classes from the same package are provided by more than one JAR. In the normal Java world, where you are dealing with a single hierarchical class loader there is no issue. Even when accessing package-private classes everything works quite merrily. However, if you bring this concept to the OSGi world, you are begging for trouble. In OSGi, each OSGi bundle has its own class loader. If you put two parts of a package in different OSGi bundles, those classes end up on different class loaders and are considered incompatible. This is because Java uses the *class loader/package/class* combination as the unique identifier. Split packages in OSGi often result in a `java.lang.IllegalAccessError` exception at runtime.

In OSGi, a package is considered an atomic unit and so it should be loaded by the same class loader to ensure consistency. If you really need split packages, and re-factoring isn't an option, a workaround is to make one of the OSGi bundles an OSGi bundle fragment instead. A fragment always lives within a host OSGi bundle so applying this fragment always lives within a host OSGi bundle so applying this technique ensures both are loaded by the same class loader.        

## 6. Know your execution environment

The OSGi bundle MANIFEST.MF header **Bundle-RequiredExecutionEnvironment** is often misunderstood. The value of this header is the *minimum* level of Java runtime in which the OSGi bundle can execute. It is often incorrectly set to an arbitrarily high level, for example JavaSE-1.8 when a value of JavaSE-1.5 might be the more correct. Setting it correctly gives your OSGi bundle greater flexibility to install into multiple environments.        
```
        Bundle-RequiredExecutionEnvironment: JavaSE-1.5
```
## 7. Avoid circular dependencies

Circular dependencies usually indicate sub-optimal packaging or application architecture problems. It is important to ensure these dependencies are designed out of the structure of your applications. A number of approaches claim to remove these cycles like *Inversion of control (Dependency Injection)*, *Interfaces*, and creation of
an *Inversion of control (Dependency Injection)*, *Interfaces*, and creation of an *intermediate object*. Although these techniques can "fix" the problem, it is often better to look at the design between the modules and question why the cyclic dependency exists. Can the application can be better designed to avoid them? Sometimes simply moving classes to a more appropriate package is enough. If you are developing in an IDE such as Eclipse this type of error is highlighted with an error message similar to the following:
`A cycle was detected in the build path of project 'com.company.package'.`

The following articles provide useful information on how to tackle those issues along with some best practices for OSGi development:
  -   [OSGi and cyclic dependencies](https://baptiste-wicht.com/posts/2010/03/osgi-and-cyclic-dependencies.html)
  -   [Best practices for developing and working with OSGi applications](https://www.ibm.com/developerworks/websphere/techjournal/1007_charters/1007_charters.html%20)        
  -   [How to solve circular package dependencies](https://softwareengineering.stackexchange.com/questions/186921/how-to-solve-circular-package-dependencies)        

## 8. Separate API from implementation

By putting your Java API in a separate OSGi bundle to the implementation of that API, you gain stability and flexibility. An implementation will generally require regular refresh due to bug fixes and improvements, while the API tends to remain stable. Keeping the two apart means that consumers of the API will not be required to refresh their dependencies each time a minor change comes along. A clear separation between API and implementation can also help avoid circular dependency problems.

## Conclusion

In our roundup of best-practices we've covered many different aspects of OSGi development from guidance with common bundles, avoiding split packages, preventing cyclic dependencies, configuration property files, whether Bundle-Classpath or full OSGi conversion is best for you. Our whether Bundle-Classpath or full OSGi conversion is best for you. Our hope is that our experience can help you avoid difficulties and embrace the benefits.
  
