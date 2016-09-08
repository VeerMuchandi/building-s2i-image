#Source to Image Builder Images

### Download the Source2Image Binary
You will need a RHEL box that is subscribed to Redhat Network.

Download latest version from 
https://github.com/openshift/source-to-image/releases/

```
wget source-to-image-v1.1.1-724c0dd-linux-386.tar.gz

tar -xvzf source-to-image-v1.1.1-724c0dd-linux-386.tar.gz
```

It will have two executables s2i and sti

```
mv s2i /usr/local/bin	  
mv sti /usr/local/bin	  
```
or add PATH the folder were the execs are untarred

```
PATH=$PATH:/home/<myuser>/source-to-image-binaries
```

Verify by running

```
$ s2i
Source-to-image (S2I) is a tool for building repeatable docker images.

A command line interface that injects and assembles source code into a docker image.
Complete documentation is available at http://github.com/openshift/source-to-image

Usage:
  s2i [flags]
  s2i [command]

Available Commands:
  version           Display version
  build             Build a new image
  rebuild           Rebuild an existing image
  usage             Print usage of the assemble script associated with the image
  create            Bootstrap a new S2I image repository
  genbashcompletion Generate Bash completion for the s2i command

Flags:
      --ca="": Set the path of the docker TLS ca file
      --cert="": Set the path of the docker TLS certificate file
  -h, --help[=false]: help for s2i
      --key="": Set the path of the docker TLS key file
      --loglevel=0: Set the level of log output (0-5)
  -U, --url="unix:///var/run/docker.sock": Set the url of the docker socket to use

Use "s2i [command] --help" for more information about a command.

```

The command below creates a directory named "s2i-lighttpd" where the artifacts for the S2I builder will be added. The target s2i builder image will be "lighthttpd-rhel".

```
s2i create lighttpd-rhel s2i-lighttpd
```

Creates the following structure:
`s2i-lighttpd/`

* `Dockerfile` – This is a standard Dockerfile where we’ll define the builder image
* `Makefile` – a helper script for building and testing the builder image
* `test/`
	* `run` – test script, testing if the builder image works correctly
	* `test-app/` – directory for your test application
* `.s2i/bin/`
	* `assemble` – script responsible for building the application
	* `run` – script responsible for running the application
	* `save-artifacts` – script responsible for incremental builds
	* `usage` – script responsible for printing the usage of the builder image




### Updating Assemble and Run scripts:


Since Lighttpd is just an httpserver, the only thing we need to do is to copy the source files into the directory from which the Lighttpd server will serve. The resultant script would look like this.

By default the s2i build places the application source in /tmp/src directory. This directory is where the source and other assets will be placed for the build process. You can modify this location by setting the io.openshift.s2i.destination label or passing --destination flag, in which case the sources will be placed in the src subdirectory of the directory you specified. The destination in the above command (./) is using working directory set in the `rhscl/s2i-base-rhel7` image

```
#!/bin/bash -e
#
# S2I assemble script for the 'lighttpd-rhel' image.
# The 'assemble' script builds your application source so that it is ready to run.
#
# For more information refer to the documentation:
#	https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

if [[ "$1" == "-h" ]]; then
	# If the 'lighttpd-rhel' assemble script is executed with '-h' flag,
	# print the usage.
	exec /usr/libexec/s2i/usage
fi

# Restore artifacts from the previous build (if they exist).
#
if [ "$(ls /tmp/artifacts/ 2>/dev/null)" ]; then
  echo "---> Restoring build artifacts..."
  mv /tmp/artifacts/. ./
fi

echo "---> Installing application source..."
cp -Rf /tmp/src/. ./

echo "---> Building application from source..."
# TODO: Add build steps for your application, eg npm install, bundle install

```

Change the run script to start the Lighttpd server as shown below:

```
#!/bin/bash -e
#
# S2I run script for the 'lighttpd-rhel' image.
# The run script executes the server that runs your application.
#
# For more information see the documentation:
#	https://github.com/openshift/source-to-image/blob/master/docs/builder_image.md
#

#exec <start your server here>
exec lighttpd -D -f /opt/app-root/etc/lighttpd.conf
```
Since are not running incremental builds, let's remove `save-artifacts` script

```
rm save-artifacts
```

Also run
 
```
chmod 755 .s2i/bin/*
```

### Add Configuration File

Lighttpd requires a configuration file that is required to run this server. Create this file as `s2i-lighttpd/etc/lighttpd.conf` with the following minimal content for Lighttpd to run

```
# directory where the documents will be served from
server.document-root = "/opt/app-root/src"

# port the server listens on
server.port = 8080

# default file if none is provided in the URL
index-file.names = ( "index.html" )

# configure specific mimetypes, otherwise application/octet-stream will be used for every file
mimetype.assign = (
  ".html" => "text/html",
  ".txt" => "text/plain",
  ".jpg" => "image/jpeg",
  ".png" => "image/png"
)

```


### Modify Dockerfile
Modify the `Dockerfile` for the following changes: 
 
* To use RHEL based builder image from Redhat's registry i.e., `registry.access.redhat.com/rhscl/s2i-base-rhel7`
* Added an environment variable to include LIGHTTPD version
* Docker Labels
* In order to install Lighttpd, you will need to install epel as well. Hence both are included in the `yum install`
* This label defines the location of S2I scripts
`io.openshift.s2i.scripts-url=image:///usr/libexec/s2i` and copy the s2i scripts to `/usr/libexec/sti`  
* Copy the configuration files from `/etc` into `/opt/app-root/etc` 
* Change the ownership of `/opt/app-root' to user 1001 and set the Docker USER to 1001
* This container would expose port 8080

The resultant code is also shown below: 

```

# lighttpd-rhel
FROM registry.access.redhat.com/rhscl/s2i-base-rhel7

# TODO: Put the maintainer name in the image metadata
# MAINTAINER Veer Muchandi <veer@redhat.com>

# TODO: Rename the builder environment variable to inform users about application you provide them
ENV LIGHTTPD_VERSION=1.4.35

# TODO: Set labels used in OpenShift to describe the builder image
LABEL io.k8s.description="Platform for serving static html pages" \
      io.k8s.display-name="Lighttpd 1.4.35" \
      io.openshift.expose-services="8080:http" \
      io.openshift.tags="builder,html,lighttpd"

# TODO: Install required packages here:
# RUN yum install -y ... && yum clean all -y
RUN rpm -Uvh http://dl.fedoraproject.org/pub/epel/7/x86_64/e/epel-release-7-7.noarch.rpm && \
    yum install -y lighttpd && \
# clean yum cache files, as they are not needed and will only make the image bigger in the end
    yum clean all -y




# Defines the location of the S2I
LABEL io.openshift.s2i.scripts-url=image:///usr/libexec/s2i

# TODO: Copy the S2I scripts to /usr/libexec/s2i
COPY ./.s2i/bin/ /usr/libexec/s2i

# TODO (optional): Copy the builder files into /opt/app-root
# COPY ./<builder_folder>/ /opt/app-root/
# Copy the lighttpd configuration file
COPY ./etc/ /opt/app-root/etc

# TODO: Drop the root user and make the content of /opt/app-root owned by user 1001
RUN chown -R 1001:1001 /opt/app-root

# This default user is created in the openshift/base-centos7 image
USER 1001

# TODO: Set the default port for applications built using this image
EXPOSE 8080

# TODO: Set the default CMD for the image
CMD ["usage"]
```

### Build the S2I Builder Image

Change to `s2i-lighttpd` directory and run

```
make build
```

This will invoke DockerBuild and will use the Dockerfile above to create a docker image with name `lighttpd-rhel`

Upon successful execution of `make build` you can run `docker images` to verify the existence of the image.

### Testing the S2I Image
Now it’s time to test our builder image with a sample application. I’m creating an index.html file in the s2i-lighttpd/test/test-app directory with the following contents:

```
<!doctype html>
<html>
  <head>
    <title>test-app</title>
  </head>
<body>
  <h1>Hello from lighttpd served index.html!</h1>
</body>
```

With this file in place, we can now do our first S2I build. Let’s invoke the following command from the s2i-lighttpd directory:

```
s2i build test/test-app/ lighttpd-rhel sample-app
```

We are building from the `test/testapp` directory, using `lighttpd-rhel` builder image that we just created. The resulting application image that includes your html will be named as `sample-app`. S2I invokes the build process as defined in the `assemble` script and displays the output.

Once the image is created you can check the existence by running `docker images` again to find `sample-app`

 
Now run the image with ``docker run`` command to test the image at `http://localhost:8080` 

```
docker run -p 8080:8080 sample-app 
```

### Pushing the image into OpenShift Registry

This step assumes that you have an OpenShift cluster setup, you have a registry running and the registry is exposed using a URL as explained here: 
[https://docs.openshift.com/enterprise/3.2/install_config/install/docker_registry.html#exposing-the-registry]()
Make a note of the docker registry url (eg: registry.apps.example.com)

You also need a user-id that has access to push images into this docker registry. See here for how to create and configure that user 
[https://docs.openshift.com/enterprise/3.2/install_config/install/docker_registry.html#access-user-prerequisites]()

1. Login as the user that has access to push to the docker registry (`system:image-builder` role as explained in the above link) using the `oc login` command.

2.  Create a new project with name `s2itest-<userid>` where you want to push the S2I image to. Substitute <userid> with your user name. It makes your project name unique.

```
oc new-project s2itest-<userid>
```


3. Find the oauth token assigned at login. We will use this same token to log into docker registry.
```
oc whoami -t
```
Make a note of the resultant oauth-token id.

4. Log into 	the docker registry using the oauth-token from the last step, user-id, your email, and the registry url for your exposed registry. **Substitute appropriate values below**

```
docker login -u <user-id> -e <email> -p <oauth-token> <registry url>
```

5. Tag docker image to point to the exposed docker registry. **Substitute the registry URL and project name with your values**

```
docker tag lighttpd-rhel <registry url>/s2itest-<username>/lighttpd-rhel
```
Run `docker images` to verify the newly tagged image. Make a note of the complete image name (eg: registry.apps.example.com/s2itest-veer/lighttpd-rhel:latest)

6. Create a json ImageStream file with the following contents. You can name it as `lighttpd-rhel-is.json`

**Substitute the namespace, image-name with your values**

```
{
    "kind": "ImageStream",
    "apiVersion": "v1",
    "metadata": {
        "name": "lighttpd-rhel",
        "namespace": "s2itest-veer"
    },
    "spec": {
        "tags": [
            {
                "name": "latest",
                "annotations": {
                    "description": "Run HTML",
                    "iconClass": "icon-tomcat",
                    "tags": "builder,lighttpd"
                },
            "from": {
              "kind": "DockerImage",
              "name": "registry.apps.example.com/s2itest-veer/lighttpd-rhel:latest"
            }
            }
        ]
    }
}
```
Note the `tags` section that uses the `builder` tag. This is required for OpenShift to recognize this imagestream as a builder image and display the same on the catalog when you try to deploy on application using webconsole.

7. Now create an imagestream in your project using this file as shown below -

```
oc create -f lighttpd-rhel-is.json
```

If you look at the image streams list using `oc get is` in the scope of your project you should now start seeing an image with name `lighttpd-rhel`. However, the tags should be empty since we did not push the docker image into the registry yet.

8. Now push the docker image into the docker registry on your openshift cluster. **Substitute the username and registry-url with appropriate values**

```
docker push <registry-url>/s2itest-<username>/lighttpd-rhel
```

9. Check the image stream in your project and you should see the `lighthttpd-rhel` image with appropriate tags as shown below

```
$ oc get is lighttpd-rhel -o yaml
apiVersion: v1
kind: ImageStream
metadata:
  annotations:
    openshift.io/image.dockerRepositoryCheck: 2016-08-10T13:40:48Z
  creationTimestamp: 2016-08-10T13:40:47Z
  generation: 3
  name: lighttpd-rhel
  namespace: s2itest-veer
  resourceVersion: "17110911"
  selfLink: /oapi/v1/namespaces/s2itest-veer/imagestreams/lighttpd-rhel
  uid: 0b43be6a-5f00-11e6-9126-fa163e38132c
spec:
  tags:
  - annotations:
      description: Run HTML
      iconClass: icon-tomcat
      tags: builder,lighttpd
    from:
      kind: DockerImage
      name: registry.apps.example.com/s2itest-veer/lighttpd-rhel:latest
    generation: 2
    importPolicy: {}
    name: latest
status:
  dockerImageRepository: 172.30.12.99:5000/s2itest-veer/lighttpd-rhel
  tags:
  - items:
    - created: 2016-08-10T13:41:19Z
      dockerImageReference: 172.30.12.99:5000/s2itest-veer/lighttpd-rhel@sha256:ea56d389358ead149ad8e51c0760900d72b763899ec03ad81730621c9759fcff
      generation: 2
      image: sha256:ea56d389358ead149ad8e51c0760900d72b763899ec03ad81730621c9759fcff
    tag: latest
```

Note that tags at the end that refer to the docker image id. If you don't see that the image stream won't work. 


### Test deploying an application
 It is time to use the S2I builder image that we just created to test from web console.
 
 * Log into the web console and switch to the project `s2itest-<username>`
 
 * Select `Add to Project` button on the top. 
 ![](images/addtoproject.tiff)

 * You should start seeing `lighthttpd-rhel:latest` image in the catalog. If you are not seeing it readily, you can always filter the list by typing `lighttpd`
![](images/imagestream.tiff)

* Select this image, give a name and the following git-url to try out a simple webpage to deploy
[https://github.com/VeerMuchandi/simple]()

This should build and deploy the application. 

Enjoy!!!


 




  






