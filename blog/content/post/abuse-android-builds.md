---
title: "Abusing Android build system"
date: 2019-09-30T09:03:00+01:00
draft: false
summary: A way to manage multiple Android flavours and releases storing the secrets securely in Git
---

In this second part (read [part 1]({{<relref "secrets-git">}}) if you haven't), we will see how we did to manage different flavors of the apps, and having both staging and production versions of the same. One requirement is to keep all the important secrets [secured in git]({{<relref "secrets-git">}}). We will abuse Gradle a little and create a couple of scripts to help us out.

We are going to build an app that will have two different flavors. Let's say that one will be `skin1` and the other will be `skin2` (very original names). Inside the assets folders you can change some colors, images etc. (something we will not do in this example). Besides having two different versions of the app being built form the same source base, we want also to be able to build them targeting different environments: `testing` and `production`. We will have different Google Play entries for them so we need to sign them with different keys.

If you are not using [managed signed certificates](https://developer.android.com/studio/publish/app-signing) from Google, to sign Android Apps you need to have a keystore. The keystore has a password to open it. Inside the keystore there is the key, which has a name (alias is called) and the key is protected with a password.

To be able to archive those goals, in Gradle we would have something like this:

```gradle
productFlavors {
        skin1 {
            applicationId "com.mydomain.app.skin1"
            signingConfig signingConfigs.skin1
        }
        skin2 {
            applicationId "com.mydomain.app.skin2"
            signingConfig signingConfigs.skin2
        }
}
```

That is to create the flavours. But we will also add the signing config as follows:

```gradle
signingConfigs {
        skin1 {
            storeFile file(SKIN_1_STORE_FILE)
            keyAlias SKIN_1_KEY_ALIAS
            storePassword SKIN_1_STORE_PASSWORD
            keyPassword SKIN_1_KEY_PASSWORD
        }

        skin2 {
            storeFile file(SKIN_2_STORE_FILE)
            keyAlias SKIN_2_KEY_ALIAS
            storePassword SKIN_2_STORE_PASSWORD
            keyPassword SKIN_2_KEY_PASSWORD
        }
}
```

As you can see from that last snippet, we have defined the signing information with some variables. `SKIN1_STORE_FILE` and so on. And now is where we are going to abuse Gradle. Probably there could be other options using Groovy (or now also Kotlin). But we found out that the following was the easiest way. 

Gradle has a file called `gradle.properties`. It is a file to define some settings. You define variables and their value, they are globals in your Gradle. And that is exactly what we need to create those variables that we used in the signing configuration. We need a `gradle.properties` as follows:

```ini
SKIN_1_STORE_FILE=../keys/skin1.jks
SKIN_1_STORE_PASSWORD=F3frT3a1
SKIN_1_KEY_ALIAS=skin1
SKIN_1_KEY_PASSWORD=lpG5ygA

SKIN_2_STORE_FILE=../keys/skin2.jks
SKIN_2_STORE_PASSWORD=F3frT3a2
SKIN_2_KEY_ALIAS=skin2
SKIN_2_KEY_PASSWORD=gpgeFg87
```

But one of the requirements was to have different signing configurations for a testing/staging environment and for production. Hence what we are going to create directory structure like this:

```bash
project/
        conf_staging/
                   gradle.properties
        conf_production/
                    gradle.properties
```

We would have two (or more if needed) `gradle.properties` files. Each one having the required variables to point to the correct keystore and how to access it. But Gradle expects to have the `gradle.properties` file in the root of the project, not somewhere in a subdirectory. All we need to do is to create a symlink.

```bash
$ ln -sfn conf_staging/gradle.properties gradle.properties
```

What we have is two scripts in the root folder, one is `production.sh`  and another one is `staging.sh`. They do the symlink and a few more things out of the scope of this explanation.

Remember that the `gradle.properties` contain the passwords to open the key stores. What we did is [protecting the production ones]({{<relref "secrets-git">}}) and leaving the staging ones unprotected.

One last important bit, to make the project work directly from Android Studio after a `git clone` is to have the symlink to the staging version checked in.

The keystore does not need to be encrypted in git as it is already protected by the password. Also depending in how you want to do it, they keystore could be the same for all the flavours/releases or each combination can have a different one. That is up to how do you prefer it. One keystore can have multiple keys.

Our case is probably not the most common scenario, but this setup has worked perfectly for us for a few years. Maybe it can help you also.
