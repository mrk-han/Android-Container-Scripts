# Container scripts for external releases
From: https://android.googlesource.com/platform/external/adt-infra/+/958180293308f4be67f6369acb075503f84a08b1/emu-image/external/

Notes after first trial: It seems this is meant to be run on a linux host machine. It also grabs username and appends hardcoded email for Maintainer of Dockerfile. 

This is a set of minimal scripts to run the emulator in a container for various
systems such as Docker. A cloud is not used; only the
emulator and system image zip files are needed.

It utilizes `adb connect` and gRPC to communicate with the Emulator when inside the Docker Container

This was adapted from Google's published Android Container Scripts folder, in which they seek to provide better support for Emulators on CI. They have an internal script for setting up Emulators in Containers, and this was loosely adapted to be "external" facing. 

After the container images are built, they can be separately pushed to
some location such as docker hub or even dl.google.com.

# Docker

There are two scripts that work together to provide emulator docker images:

    emu_docker.py
    emu_download_menu.py

`emu_docker.py` sets up:
- A Docker image source directory with a Dockerfile that is buildable and runnable as a Docker image when given a:
    - A Linux emulator zip file
    - A system image zip file
    - A docker repo name (currently unused; any name will do).

`emu_download_menu.py` contains: 
- A set of publically available Android Emulator system images and emulators along with their URLs
    - This makes it easier to download zip files with `emu_docker.py`.

## Obtaining URLs for emulator/system image zip files

Issuing:

    python emu_download_menu.py

will query the currently published Android SDK and output URLs for the zip files of:

- Available and currently Docker-compatible system images
- Currently published and advertised emulator binaries

For each system image (SYSIMG):
- variant
- API level
- ABI
- URL 

For each emulator (EMU):
- the update channel (stable vs canary)
- version
- host os
- URL

Example output:

    SYSIMG android 21 L x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-21_r05.zip
    SYSIMG android 22 L x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-22_r06.zip
    SYSIMG android 23 M x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-23_r10.zip
    SYSIMG android 24 N x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-24_r08.zip
    SYSIMG android 25 N x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-25_r01.zip
    SYSIMG android 26 O x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-26_r01.zip
    SYSIMG android 27 O x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-27_r01.zip
    SYSIMG android 28 P x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-28_r04.zip
    SYSIMG android 28 Q x86_64 https://dl.google.com/android/repository/sys-img/android/x86_64-Q_r04.zip
    SYSIMG google_apis 21 L x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-21_r30.zip
    SYSIMG google_apis 22 L x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-22_r24.zip
    SYSIMG google_apis 23 M x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-23_r31.zip
    SYSIMG google_apis 24 N x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-24_r25.zip
    SYSIMG google_apis 25 N x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-25_r16.zip
    SYSIMG google_apis 26 O x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-26_r13.zip
    SYSIMG google_apis 28 P x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-28_r09.zip
    SYSIMG google_apis 28 Q x86_64 https://dl.google.com/android/repository/sys-img/google_apis/x86_64-Q_r04.zip
    SYSIMG google_apis_playstore 28 P x86_64 https://dl.google.com/android/repository/sys-img/google_apis_playstore/x86_64-28_r08.zip
    SYSIMG google_apis_playstore 28 Q x86_64 https://dl.google.com/android/repository/sys-img/google_apis_playstore/x86_64-Q_r04.zip
    EMU stable 29.0.11 windows https://dl.google.com/android/repository/emulator-windows-5598178.zip
    EMU stable 29.0.11 macosx https://dl.google.com/android/repository/emulator-darwin-5598178.zip
    EMU stable 29.0.11 linux https://dl.google.com/android/repository/emulator-linux-5598178.zip
    EMU stable 28.0.25 windows https://dl.google.com/android/repository/emulator-windows-5395263.zip
    EMU canary 29.0.12 windows https://dl.google.com/android/repository/emulator-windows-5613046.zip
    EMU canary 29.0.12 macosx https://dl.google.com/android/repository/emulator-darwin-5613046.zip
    EMU canary 29.0.12 linux https://dl.google.com/android/repository/emulator-linux-5613046.zip

One can then use tools like `wget` or a browser to download a desired emulator(EMU)
and system image(SYSIMG).  After the two are obtained, we can build a Docker image.

```
wget {SYSIMG URL}
wget {EMU URL}
```

^ This will install the EMU and SYSIMG in the current directory, to later be used in the emu_docker.py file.

## Building the Docker image: Setting up the source dir

Given an emulator zip file and a system image zip file, we can build a
directory that can be sent to `docker build` via the following invocation of
`emu_docker.py`:

    python emu_docker.py <emulator-zip> <system-image-zip> <docker-repo-name(unused currently)> [docker-src-dir (getcwd()/src by default)]

This places all the right elements to run a docker image, but does not build,
run or publish yet. A Linux emulator zip file must be used.

To build the Docker image corresponding to these emulators and system images:

    docker build <docker-src-dir, either ./src or specified argument to emu_docker.py>

A Docker image ID will output; save this image ID.

## Running the Docker image

We currently assume that KVM will be used with docker in order to provide CPU
virtualization capabilties to the resulting Docker image.

We provide the following run script:

    ./run.sh <docker-image-id>

It does the following:

    docker run -e "ADBKEY=$(cat ~/.android/adbkey)" --privileged  --publish 5556:5556/tcp --publish 5555:5555/tcp <docker-image-id>

The command line parameters mean the following:

-e "ADBKEY=$(cat ~/.android/adbkey)" Set the environment variable ADBKEY to contain the private key used by your current adb install. Usually this private key resides in ~/.android/adbkey
--privileged The emulator needs access to kvm, hence you will need to pass the --privileged flag. Without docker will not have access to KVM and you will not be able run the emulator.
--publish 5556:5556/tcp make the internal port 5556 in the docker container visible to the outside world at port 5556. This is the gRPC port, that can be used to interact with the emulator. The gRPC service is used to communicate with the running emulator inside the container.
--publish 5555:5555/tcp make the internal port 5555 in the docker container visible to the outside world at port 5555. This is the ADB port that can be used to interact with the emulator.

## Communicating with the emulator in the container

## adb

We forward the port 5555 for adb access to the emulator running inside the
container (TODO: make this configurable per container).

To enable ADB access, run the following adb command, assuming no other emulators/devices connected:

    adb connect localhost:5555

## gRPC/webrtc

We use a gRPC/webrtc service to show what is on the emulator and to interact
with it.  This assumes that you have npm/node plus protoc 3.6+ installed and
available, and that no other node servers are running on your machine.

Checkout the emulator repo dir:

    https://android.googlesource.com/platform/external/qemu/+/refs/heads/emu-master-dev/android/android-grpc/docs/grpc-samples/js/


In the `/js` dir, issue:

    make deps # Uses npm and protoc 3.6+ libraries to build the client
    make develop # Runs the service

and point your web browser to localhost:3000.

To stop, hit Ctrl-C in the terminal where `make develop` was issued, then issue:

    make stop

TODO: We are also working on a more isolated solution via envoy, nginx, and `docker-compose`. See https://android.googlesource.com/platform/external/qemu/+/refs/heads/emu-master-dev/android/android-grpc/docs/grpc-samples/js/docker/

