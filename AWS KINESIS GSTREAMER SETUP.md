# AWS KINESIS

### _G-STREAMER PRODUCER SK_

Amazon Kinesis Video Streams provides the C++ Producer Library, which you can use to write application code to send media data from a device to a Kinesis video stream.

To be able to use the C++ Producer Library, We list the necessary steps to install the Producer SDK for Kinesis are : 

#### Install Git: _`sudo apt-get install git`_
```sh
$ git --version
git version 2.14.1
```

#### Install [Cmake](https://www.kitware.com/platforms/#cmake): _`sudo apt-get install cmake`_
```sh
$ cmake --version
cmake version 3.9.1
```

#### Install Libtool: _`sudo apt-get install libtool`_
```sh
2.4.6-2
```

#### Install libtool-bin: _`sudo apt-get install libtool-bin`_
```sh
$ libtool --version
libtool (GNU libtool) 2.4.6
Written by Gordon Matzigkeit, 1996
```

#### Install GNU Automake: _`sudo apt-get install automake`_
```sh
$ automake --version
automake (GNU automake) 1.15
```

#### Install GNU Bison: _sudo apt-get install bison_
```sh
$ bison -V
bison (GNU Bison) 3.0.4
```

#### Install G++: _sudo apt-get install g++_
```sh
g++ --version
g++ (Ubuntu 7.2.0-8ubuntu3) 7.2.0
```
#### Install curl: _sudo apt-get install curl_
```sh
$ curl --version
curl 7.55.1 (x86_64-pc-linux-gnu) libcurl/7.55.1 OpenSSL/1.0.2g zlib/1.2.11 libidn2/2.0.2 libpsl/0.18.0 (+libidn2/2.0.2) librtmp/2.3
```

#### Install pkg-config: _sudo apt-get install pkg-config_
```sh
$ pkg-config --version
0.29.1
```

#### Install Flex: _sudo apt-get install flex_
```sh
$ flex --version
flex 2.6.1
```

Install OpenJDK: _sudo apt-get install openjdk-8-jdk_
```sh
$ java -version
openjdk version "1.8.0_171"
```
#### Step 2: 
- Set the JAVA_HOME environment variable: export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64/
   > this export must be made in the bashrc file (~/.barshrc)
   > Also in the terminal without the bash file : to make sure that the everything is ok, do `echo $JAVA_HOME_PATH` in the terminal 
- Run the build script: ./install-script


#### Step 3: 
#### **_Amazon Kinesis Video Streams CPP Producer, GStreamer Plugin and JNI_**
#### - Build
**Download**
To download run the following command:
`git clone https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp.git`

**Configure**
Prepare a build directory in the newly checked out repository:
```sh
$ mkdir -p amazon-kinesis-video-streams-producer-sdk-cpp/build 
$ cd amazon-kinesis-video-streams-producer-sdk-cpp/build
```

**On Ubuntu and Raspberry Pi OS you can get the libraries by running**
```sh
$ sudo apt-get install libssl-dev libcurl4-openssl-dev liblog4cplus-dev libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev gstreamer1.0-plugins-base-apps gstreamer1.0-plugins-bad gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly gstreamer1.0-tools
```

**CMake Arguments**
You can pass the following options to `cmake .. `
| Command | Description |
| ------- | ----------- |
|-DBUILD_GSTREAMER_PLUGIN |-- Build kvssink GStreamer plugin |
-DBUILD_JNI |-- Build C++ wrapper for JNI to expose the functionality to Java/Android
-DBUILD_DEPENDENCIES |-- Build depending libraries from source
-DBUILD_TEST=TRUE |-- Build unit/integration tests, may be useful for confirm support for your device. ./tst/producerTest
-DCODE_COVERAGE |-- Enable coverage reporting
-DCOMPILER_WARNINGS |-- Enable all compiler warnings
-DADDRESS_SANITIZER |-- Build with AddressSanitizer
-DMEMORY_SANITIZER |-- Build with MemorySanitizer
-DTHREAD_SANITIZER |-- Build with ThreadSanitizer
-DUNDEFINED_BEHAVIOR_SANITIZER | -- Build with UndefinedBehaviorSanitizer
-DALIGNED_MEMORY_MODEL |-- Build for aligned memory model only devices. Default is OFF.

**NOW YOU SHOULD Include JNI**
JNI examples are NOT built by default. If you wish to build JNI you MUST add `-DBUILD_JNI=TRUE` when running cmake:
```sh
$ cmake -DBUILD_JNI=TRUE
```

**THEN YOU SHOULD Include Building GStreamer Sample Programs**
The GStreamer plugin and samples are NOT built by default. If you wish to build them you MUST add `-DBUILD_GSTREAMER_PLUGIN=TRUE` when running cmake:
```sh
$ cmake -DBUILD_GSTREAMER_PLUGIN=TRUE
```

**Compiling**
After running cmake, in the same build directior run make:
```sh
$ make
```
In your build directory you will now have shared objects for all the targets you have selected.

To load this plugin set the following environment variables. This should be run from the root of the repo, NOT the build directory.

`export GST_PLUGIN_PATH='pwd'/build`
`export LD_LIBRARY_PATH='pwd'/open-source/local/lib`

when you type `pwd` in the terminal in the build directory, you should get the  directory of the Build folder.
The same for the lib folder. 
> YOU SHOULD INCLUDE THESES TWO LINES OF EXPORT IN THE bashrc file
> ```sh
> $ sudo nano ~/.bashrc
> ```
> if it does not work you can go to the roor user and add them in this file also  `sudo su`.


Nox you should execute  `gst-inspect-1.0 kvssink `  and you should get information on the plugin like

- **_Factory Details:_**
  Rank                     primary + 10 (266)
  Long-name                KVS Sink
  Klass                    Sink/Video/Network
  Description              GStreamer AWS KVS plugin
  Author                   AWS KVS <kinesis-video-support@amazon.com>

- **_Plugin Details:_**
  Name                     kvssink
  Description              GStreamer AWS KVS plugin
  Filename                 /Users/seaduboi/workspaces/amazon-kinesis-video-streams-producer-sdk-cpp/build/libgstkvssink.so
  Version                  1.0
  License                  Proprietary
  Source module            kvssinkpackage
  Binary package           GStreamer
  Origin URL               http://gstreamer.net
  
If the build failed, or GST_PLUGIN_PATH is not properly set you will get output like:
`No such element or plugin 'kvssink'`

**Using Element**
The kvssink element has the following required parameters: 

| Command | Description |
| ------ | ------ |
|`stream-name` |-- The name of the destination Kinesis video stream.
|storage-size |-- The storage size of the device in megabytes. For information about configuring device storage, see StorageInfo.
|access-key |-- The AWS access key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
|secret-key |-- The AWS secret key that is used to access Kinesis Video Streams. You must provide either this parameter or credential-path.
|credential-path |-- A path to a file containing your credentials for accessing Kinesis Video Streams. For example credential files, see Sample Static Credential and Sample Rotating Credential. For more information on rotating credentials, see Managing Access Keys for IAM Users. You must provide either this parameter or access-key and secret-key.


For examples of common use cases you can look at Example: [Kinesis Video Streams Producer SDK GStreamer Plugin](https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/examples-gstreamer-plugin.html)

### Sources
- https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/producer-sdk-cpp.html
- https://github.com/awslabs/amazon-kinesis-video-streams-producer-sdk-cpp

**Free Software, Hell Yeah!**

