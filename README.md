# The most trivial (and hacky) code generation with Gradle for Android

For example given an interface in root package of an application (let's assume it's `com.sample`)
```java
package com.sample;

interface Configuration {
    boolean hasFeature(int flag);
}
```

There is a need to have a specific implementation depending on `buildType` or `productFlavor` or (which is more applicable in this example) a specifc property during a build time. For example, if build is triggered with property: `supportedFeatures=1,3,5,10` we want all builds to reflect that. Here is what we can do:

`gradlew assemble -PsupportedFeatures=1,3,5,10`

```groovy
android {

    compileSdkVersion 25
    buildToolsVersion '25.0.2'
    
    defaultConfig {
        applicationId 'com.sample'
        minSdkVersion 16
        targetSdkVersion 25
        versionCode 1
        versionName '1.0.0'
        
        final def conf = project.hasProperty('supportedFeatures') \
                ? 'flags == ' + project.get('supportedFeatures').split(',').join(' || flags == ') \
                : 'false'

        final def configurationImpl = """
                public static class ConfigurationImpl implements Configuration {
                    @Override
                    public boolean hasFeature(int flags) {
                        return $conf;
                    }
                }
        """
        
        buildConfigField 'byte', '$$__$$', '0;\n' +
                configurationImpl
    }
}
```

So, first we check if our project has `supportedFeatures` property, if it does we construct the return statement with applied logic, if not we just return `false`.

Then we write a basic `Configuration` implementation and insert into it our return statement.

Then, we declare a mock `buildConfigField` with byte type (the smallest possible type in Java), give it unreadable name (to avoid its usage by random caller), place a semicolon `;` and a new line `\n` and concat with our configuration implementation source code. When the project will be build we will see this in `com.sample.BuildConfig`:

```java
package com.sample;

public final class BuildConfig {
    public static final boolean DEBUG = Boolean.parseBoolean("true");
    public static final String APPLICATION_ID = "com.sample";
    public static final String BUILD_TYPE = "debug";
    public static final String FLAVOR = "";
    public static final int VERSION_CODE = 1;
    public static final String VERSION_NAME = "1.0.0";
    
    // Fields from default config.
    public static final byte $$__$$ = 0;

    public static class ConfigurationImpl implements Configuration {
        @Override
        public boolean hasFeature(int flags) {
            return flags == 1 || flags == 3 || flags == 5 || flags == 10;
        }
    }
    ;
}
```

Then it can be used as usual:
```java
final Configuration configuration = new BuildConfig.ConfigurationImpl();
```

Please note that I'm not advocating to abuse Android Gradle plugin like this. **Most likely** there is a better way to achieve the same result, but if you need a quick solution to test something or you are in a *hacky* mood this can help a bit.
