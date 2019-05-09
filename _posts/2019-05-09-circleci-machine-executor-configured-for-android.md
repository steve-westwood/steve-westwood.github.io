---
layout: post
title: "CircleCi's Machine Executor Configured for Android"
date: 2019-05-09
---

If you ever tried to configure CircleCi to build your Android App you would probably go straight to the [CircleCi's official language guide for Android](https://circleci.com/docs/2.0/language-android/). Which is a reasonable course of action. 

CircleCi have a [docker executor](https://circleci.com/docs/2.0/executor-types/#using-docker) with _your name_ on it. If your name is `circleci/android:api-25-alpha` or similar. A bit of tinkering and it works! Your project builds and, in general, acts like a CI pipeline should do.

Life is hunky-dory, all is calm, all is well, until... 

You start adding new features, or (god forbid) you develop your App using React-Native, and gradually your build gets a bit more _involved_.

**Enter the Java Out of Memory issue!**

## Java Out Of Memory (OOM) Errors in CircleCi's Docker Executor

> ▸ FAILURE: Build failed with an exception.
>
> ▸ * What went wrong:
>
> ▸ Execution failed for task ':app:bundleReleaseJsAndAssets'.
>
> ▸ > Process 'command 'node'' finished with non-zero exit value 137

This is a Java OOM error, although it's not entirly clear what's happening from the build output. A `137 error` and a vague error message isn't overly descriptive. A bit of googling and [it becomes clearer](https://success.docker.com/article/what-causes-a-container-to-exit-with-code-137). The particular error above was provoked in a previously building React-Native Android project, after upgrade to use Android's v28 SDK. It bums out when it tries to execute a node command to package up the JavaScript in a deployable bundle, but the _source_ of the error isn't important. There's nothing _wrong_ with the command, it has just pushed the **docker executor** over its memory limit.

## Breaking Docker with Java

Turns out this isn't that hard to do. A combination of Java having an overly greedy build process, less than ideal default configuration for Docker, and CircleCi assigning only 4GB of memory to their docker executors makes it easier than it should be. CircleCi have recognised this as an issue an have a [helpful blog post](https://circleci.com/blog/how-to-handle-java-oom-errors/) which may help you. I tried the suggestions mentioned in that blog and it didn't help. My memory hungry React-Native Android build was just eating ALL THE MEMORY.

## The (CircleCi) Solution

If all else fails, CircleCi says upgrade so that your project can use the [resource_class](https://circleci.com/docs/2.0/configuration-reference/#resource_class) feature (and a whole bunch of other cool stuff btw). Throw more memory at the problem! (At a small cost!) Trouble with this upgrade is that you can't just upgrade the problem Project. You have to pay _n_ dollars per project, and if you have a few dozen projects that cost will stack up, especially as at this point you probably only need the additional features to solve you single problem project.

But don't despair, there is another way...

## The other (cheaper) way to fix the problem

Luckily CircleCi suite of executors contains a more powerful agent, and we can use it (with a bit of configuration) in the standard plan!

![CircleCi's Machine Executor](https://steve-westwood.github.io/images/machine_executor.png)

I present to you the [machine executor](https://circleci.com/docs/2.0/executor-types/#using-machine) from CircleCi which has 8GB of RAM! :tada:

...but doesn't have the Android SDK software loaded :-1:

## So lets configure it so it has ALL the stuff

> This configuration will be for Android and React-Native

First of all lets select the machine executor, and environment variables we'll use later.

```yaml
version: 2
jobs:
    build_android:
        environment:
            BASH_ENV: envrc
        machine:
            enabled: true
```

The main software we need is the Android Studio SDK to build the App. At current time of writing the SDK version is v28 (Android 9 API), and the download of the SDK tools can be found (on the Android Studio downloads page)[https://developer.android.com/studio/#downloads].

> As an aside I can see that you used to be able to download these tools using the **apt** utility, so I'm expecting the method of getting these tools to chnage in the future as Google changes it mind over the best delivery mechanism.

```yaml
            - run:
                  name: Download android sdk
                  command: |
                      cd .. 
                      mkdir android-sdk
                      cd android-sdk
                      wget https://dl.google.com/android/repository/sdk-tools-linux-4333796.zip
                      unzip sdk-tools-linux-4333796.zip
```

Now we can add these tools to `$PATH` environment variable so they will be usable in later bash commands.

```yaml

            - run:
                  name: Set env vars for android sdk tools
                  command: |
                      echo '
                      export ANDROID_HOME=$HOME/android-sdk
                      export PATH=$PATH:$HOME/android-sdk/tools/bin
                      export PATH=$PATH:$HOME/android-sdk/platform-tools' >> envrc
```

Now we have the SDK software available on the machine executor, but there is _one_ last hurdle before they can be used successfully to build our project. Google wants you to accept their terms and conditions before issuing a license to use these command line tools, so we have to **accept all license** via the command line before going forward. This was a bit tricky but luckily I found [this conversation on CircleCi's forum](https://discuss.circleci.com/t/android-platform-28-sdk-license-not-accepted/27768/11) which had the solution (to pipe yes to the accept licenses command and return a non-exit response).

```yaml
            - run:
                  name: Accept sdk licenses
                  command: |
                      yes | sdkmanager --licenses || exit 0
                      yes | sdkmanager --update || exit 0
```







