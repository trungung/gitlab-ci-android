# Docker Image for Build Android [![](https://images.microbadger.com/badges/image/jerbob92/gitlab-ci-android.svg)](https://microbadger.com/images/jerbob92/gitlab-ci-android "Get your own image badge on microbadger.com")

GitLab CI image for building and testing Android apps

This code on repo will automatically build on Docker Hub : 

https://hub.docker.com/r/jerbob92/gitlab-ci-android/

## Example `.gitlab-ci.yml` file
```yml
image: jerbob92/gitlab-ci-android:latest

before_script:
    - export GRADLE_USER_HOME=`pwd`/.gradle
    - mkdir -p $GRADLE_USER_HOME
    - chmod +x ./gradlew

cache:
  paths:
     - .gradle/wrapper
     - .gradle/caches

build:
  stage: build
  script:
     - ./gradlew assemble

test:
  stage: test
  script:
     - ./gradlew check

```

## Example `.gitlab-ci.yml` with tests in emulator and recording of the screen.

```
image: jerbob92/gitlab-ci-android:latest

stages:
  - build
  - test

before_script:
    - export GRADLE_USER_HOME=`pwd`/.gradle
    - mkdir -p $GRADLE_USER_HOME
    - chmod +x ./gradlew

cache:
  paths:
     - .gradle/wrapper
     - .gradle/caches

build:
  stage: build
  script:
      - ./gradlew assembleDebug
      - ./gradlew assembleDebugAndroidTest
  artifacts:
    paths:
      - app/build/outputs/

test:
  stage: test
  script:
      - echo "no" | /sdk/tools/android create avd -f -n test -t android-23 --abi "google_apis/x86" -s WXGA720
      - cp -rf resources/avd.ini ~/.android/avd/test.avd/config.ini
      - echo "no" | /sdk/tools/emulator64-x86 -avd test -wipe-data -noaudio -no-window -gpu off -verbose -qemu -usbdevice tablet -vnc :2 &
      - /helpers/wait-for-avd-boot.sh
      - /sdk/platform-tools/adb install -r app/build/outputs/apk/app-debug.apk
      - /sdk/platform-tools/adb install -r app/build/outputs/apk/app-debug-androidTest-unaligned.apk
      - /sdk/platform-tools/adb shell pm grant [package-name] android.permission.SET_ANIMATION_SCALE
      - flvrec.py -o test.flv localhost 5902 &
      - /sdk/platform-tools/adb shell am instrument -w -r -e debug false [package-name]/[test-class] | tee test.log
      - pkill -f flvrec
      - yamdi -i test.flv -o test_recording.flv
      - rm test.flv
      - if grep -q "FAILURES!!!" test.log; then exit 1; fi
  artifacts:
    paths:
      - test_recording.flv
      - test.log
    when: on_failure
    expire_in: 1 week

```

Change [package-name] to your package name and [test-class] to your testing class.
