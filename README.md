#Building Tensorflow 1.9.0 from source for Artik board running ubuntu 16.04

1. Preparation:
you need a few dependencies for the installation:
	sudo apt-get update
	sudo apt-get upgrade
	sudo apt-get install pkg-config zip unzip g++ zlib1g-dev default-jdk autoconf automake libtool python-dev python-pip python-setuptools git

	sudo apt-get install python-opencv	

2. Install a memory drive as swap for compiling (https://gist.github.com/EKami/9869ae6347f68c592c5b5cd181a3b205)
	First, put insert your USB drive, and find the /dev/XXX path for the device.
		sudo blkid
	As an example, my drive's path was /dev/sda1
	Once you've found your device, unmount it by using the umount command.
		sudo umount /dev/XXX
	Format your USB drive with the following command (the swap area will be 2GB: 1024 * 2048 = 2097152):
		sudo dd if=/dev/zero of=/dev/sda bs=1024 count=2097152
	Find it back with this command:
		sudo fdisk -l
	Then flag your device to be swap:
		sudo mkswap /dev/XXX
	If the previous command outputted an alphanumeric UUID, copy that now. Otherwise, find the UUID by running blkid again. Copy the UUID associated with /dev/XXX
		sudo blkid
	Now edit your /etc/fstab file to register your swap file. (I'm a Vim guy, but Nano is installed by default)
		sudo nano /etc/fstab
	On a separate line, enter the following information. Replace the X's with the UUID (without quotes)
		UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX none swap sw,pri=5 0 0
	Save /etc/fstab, exit your text editor, and run the following command:
		sudo swapon -a
	If you get an error claiming it can't find your UUID, go back and edit /etc/fstab. Replace the UUID=XXX.. bit with the original /dev/XXX information.
		sudo nano /etc/fstab
		# Replace the UUID with /dev/XXX
		/dev/XXX none swap sw,pri=5 0 0
	Alright! You've got swap! Don't throw out the /dev/XXX information yet- you'll need it to remove the device safely later on.
	
3. Build bazel (version 0.15)
	get the bazel
		wget --no-check-certificate https://github.com/bazelbuild/bazel/releases/download/0.15.1/bazel-0.15.1-dist.zip
	upzip it
		unzip -d bazel bazel-0.15.1-dist.zip
	enter to bazel
		cd bazel
	Now we need to change the permissions of every files in the bazel project with:
		sudo chmod u+w ./* -R
	Before building Bazel, we need to set the javac maximum heap size for this job, or else we'll get an OutOfMemoryError. 
	To do this, we need to make a small addition to bazel/scripts/bootstrap/compile.sh. (Shout-out to @SangManLINUX for pointing this out..
		vi scripts/bootstrap/compile.shcd
	Move down to line 117, where you'll see the following block of code:
		run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      			-d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      			-encoding UTF-8 "@${paramfile}"
	change to this
		run "${JAVAC}" -classpath "${classpath}" -sourcepath "${sourcepath}" \
      			-d "${output}/classes" -source "$JAVA_VERSION" -target "$JAVA_VERSION" \
      			-encoding UTF-8 "@${paramfile}" -J-Xmx500M
	Now we can build Bazel! Warning: This takes a really, really long time. Several hours (in artik board, it took about 2 hours)
		./compile.sh
	When the build finishes, you end up with a new binary, output/bazel. Copy that to your /usr/local/bin directory.
		sudo cp output/bazel /usr/local/bin/bazel
	To make sure it's working properly, run bazel on the command line and verify it prints help text. Note: this may take 15-30 seconds to run, so be patient!
		bazel

Usage: bazel <command> <options> ...

Available commands:
  analyze-profile     Analyzes build profile data.
  build               Builds the specified targets.
  canonicalize-flags  Canonicalizes a list of bazel options.
  clean               Removes output files and optionally stops the server.
  dump                Dumps the internal state of the bazel server process.
  fetch               Fetches external repositories that are prerequisites to the targets.
  help                Prints help for commands, or the index.
  info                Displays runtime info about the bazel server.
  mobile-install      Installs targets to mobile devices.
  query               Executes a dependency graph query.
  run                 Runs the specified target.
  shutdown            Stops the bazel server.
  test                Builds and runs the specified test targets.
  version             Prints version information for bazel.

Getting more help:
  bazel help <command>
                   Prints help and options for <command>.
  bazel help startup_options
                   Options for the JVM hosting bazel.
  bazel help target-syntax
                   Explains the syntax for specifying targets.
  bazel help info-keys
                   Displays a list of keys used by the info command.

	Move out of the bazel directory, and we'll move onto the next step.
		cd ..

4. Build tensorflow (version 1.9) 
refer this link 
http://zhiyisun.github.io/2017/02/15/Running-Google-Machine-Learning-Library-Tensorflow-On-ARM-64-bit-Platform.html
https://stackoverflow.com/questions/46653465/tensorflow-compilation-on-odroid-xu4
	First things first, clone the TensorFlow repository and move into the newly created directory.
		git clone https://github.com/tensorflow/tensorflow
		git checkout v1.9.0 (optional)
	Or download the zip file with the wget
		wget --no-check-certificate https://github.com/tensorflow/tensorflow/archive/r1.9.zip
		unzip id tensorflow r1.9.zip
	enter to tensorflow folder		
		cd tensorflow
	Now we have to write a nifty one-liner that is incredibly important. The next line goes through all files and changes references 
	of 64-bit program implementations (which we don't have access to) to 32-bit implementations. Neat!
		grep -Rl 'lib64' | xargs sed -i 's/lib64/lib/g'
	confirm the certificate:
		path to home jdk
		/usr/lib/jvm/java-8-openjdk-arm64/jre/lib/security
		keytool -import -alias bazel.build  -keystore cacerts -file ./bazel.build.cer -trustcacerts
		keytool -import -alias github.com  -keystore cacerts -file ./github.com.cer -trustcacerts
	perform the configuration:
		./configuration
	
	check the date:
		date
	setup the time:	
		sudo date --set "12 Jul 2018 09:33:20"
	COMPILE		
		bazel build -c opt --copt="-funsafe-math-optimizations" --copt="-ftree-vectorize" --copt="-fomit-frame-pointer" --local_resources 1024,1.0,1.0 --verbose_failures tensorflow/tools/pip_package:build_pip_package
		check this link to solve the -mfpu=neon problem with gcc:
			(https://github.com/tensorflow/tensorflow/issues/17001) check this file /tensorflow/contrib/lite/kernels/internal/BUILD		
		build tensorflow lite:
			bazel build  --linkopt='-lrt' -c opt //tensorflow/tools/pip_package:build_pip_package
	Build pip package
		bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/tensorflow_pkg
	
	Install tensorflow pip package
	sudo pip install /tmp/tensorflow_pkg/tensorflow-1.0.0rc2-cp27-cp27mu-linux_aarch64.whl
