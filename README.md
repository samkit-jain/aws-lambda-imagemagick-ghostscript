# Use ImageMagick and Ghostscript in AWS Lambda

**Runtime:** Python 3.8<br/>
**TL;DR:** Download the ZIP [here](lambda-layer.zip) and skip to step 14.

My use-case was to convert pages in a PDF to PNG via `wand`. I followed the instructions provided by @chaddjohnson at https://gist.github.com/bensie/56f51bc33d4a55e2fc9a#gistcomment-3133859 but was getting errors. Following are the instructions on how I was able to resolve those:
1. Start an EC2 instance and SSH into it. I used the AMI `amzn2-ami-hvm-2.0.20210126.0-x86_64-gp2`.
2. Download ImageMagick 6.9.11.
    ```bash
    wget https://download.imagemagick.org/ImageMagick/download/ImageMagick-6.9.11-60.tar.gz
    ```
3. Extract the folder.
    ```bash
    tar zxvf ImageMagick-6.9.11-60.tar.gz
    ```
4. `cd` into the extracted folder.
    ```bash
    cd ImageMagick-6.9.11-60
    ```
5. Edit the `policy.xml` file to allow PDF to PNG conversion.
    ```bash
    nano config/policy.xml
    ```
    I copy-pasted the following content but you can modify it as needed.
    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE policymap [
    <!ELEMENT policymap (policy)+>
    <!ELEMENT policy (#PCDATA)>
    <!ATTLIST policy domain (delegate|coder|filter|path|resource) #IMPLIED>
    <!ATTLIST policy name CDATA #IMPLIED>
    <!ATTLIST policy rights CDATA #IMPLIED>
    <!ATTLIST policy pattern CDATA #IMPLIED>
    <!ATTLIST policy value CDATA #IMPLIED>
    ]>
    <!--
      Configure ImageMagick policies.

      Domains include system, delegate, coder, filter, path, or resource.

      Rights include none, read, write, and execute.  Use | to combine them,
      for example: "read | write" to permit read from, or write to, a path.

      Use a glob expression as a pattern.

      Suppose we do not want users to process MPEG video images:

        <policy domain="delegate" rights="none" pattern="mpeg:decode" />

      Here we do not want users reading images from HTTP:

        <policy domain="coder" rights="none" pattern="HTTP" />

      Lets prevent users from executing any image filters:

        <policy domain="filter" rights="none" pattern="*" />

      The /repository file system is restricted to read only.  We use a glob
      expression to match all paths that start with /repository:
      
        <policy domain="path" rights="read" pattern="/repository/*" />

      Let's prevent possible exploits by removing the right to use indirect reads.

        <policy domain="path" rights="none" pattern="@*" />

      Any large image is cached to disk rather than memory:

        <policy domain="resource" name="area" value="1GB"/>

      Define arguments for the memory, map, area, width, height, and disk resources
      with SI prefixes (.e.g 100MB).  In addition, resource policies are maximums
      for each instance of ImageMagick (e.g. policy memory limit 1GB, -limit 2GB
      exceeds policy maximum so memory limit is 1GB).
    -->
    <policymap>
      <!-- <policy domain="resource" name="temporary-path" value="/tmp"/> -->
      <policy domain="resource" name="memory" value="256MiB"/>
      <policy domain="resource" name="map" value="512MiB"/>
      <policy domain="resource" name="width" value="16KP"/>
      <policy domain="resource" name="height" value="16KP"/>
      <policy domain="resource" name="area" value="128MB"/>
      <policy domain="resource" name="disk" value="1GiB"/>
      <!-- <policy domain="resource" name="file" value="768"/> -->
      <!-- <policy domain="resource" name="thread" value="4"/> -->
      <!-- <policy domain="resource" name="throttle" value="0"/> -->
      <!-- <policy domain="resource" name="time" value="3600"/> -->
      <!-- <policy domain="system" name="precision" value="6"/> -->
      <!-- not needed due to the need to use explicitly by mvg: -->
      <!-- <policy domain="delegate" rights="none" pattern="MVG" /> -->
      <!-- use curl -->
      <policy domain="delegate" rights="none" pattern="URL" />
      <policy domain="delegate" rights="none" pattern="HTTPS" />
      <policy domain="delegate" rights="none" pattern="HTTP" />
      <!-- in order to avoid to get image with password text -->
      <policy domain="path" rights="none" pattern="@*"/>
      <policy domain="cache" name="shared-secret" value="passphrase" stealth="true"/>
      <!-- disable ghostscript format types -->
      <policy domain="coder" rights="none" pattern="PS" />
      <policy domain="coder" rights="none" pattern="EPI" />
      <policy domain="coder" rights="read|write" pattern="PDF" />
      <policy domain="coder" rights="none" pattern="XPS" />
      <policy domain="coder" rights="read|write" pattern="LABEL" />
    </policymap>
    ```
6. Configure and install ImageMagick.
    ```bash
    ./configure --prefix=/var/task/imagemagick --sysconfdir=/etc --datadir=/usr/share --includedir=/usr/include --libdir=/usr/lib64 --libexecdir=/usr/libexec --localstatedir=/var --sharedstatedir=/var/lib --mandir=/usr/share/man --infodir=/usr/share/info --enable-shared=no --enable-static=yes --with-modules --with-perl=no --with-x=no --with-gslib=no --with-lcms --without-rsvg --with-xml --without-dps --disable-hdri --with-quantum-depth=8 --disable-openmp
    make
    sudo make install
    ```
7. Copy the required `.so` files.
    ```bash
    mkdir lib
    cd /usr/lib64/
    cp -L libbz2.so.1 libexpat.so.1 libfontconfig.so.1 libfreetype.so.6 libgs.so.9 libjbig.so.2.0 libjpeg.so.62 liblcms2.so.2 liblzma.so.5 libpng15.so.15 libtiff.so.5 libxml2.so.2 libMagickCore-6.Q16.so.6 libMagickWand-6.Q16.so.6 libXext.so.6 libXt.so.6 libltdl.so.7 libSM.so.6 libICE.so.6 libX11.so.6 libgomp.so.1 libuuid.so.1 libxcb.so.1 libXau.so.6 libMagickCore-6.Q8.so.6 libMagickWand-6.Q8.so.6 libm.so.6 libz.so.1 libjasper.so.1 /home/ec2-user/lib/
    cp -r ImageMagick-6.9.10/ ImageMagick-6.9.11/ /home/ec2-user/lib/
    cd /home/ec2-user
    tar zcf lib.tar.gz lib/
    ```
    Copy the `lib.tar.gz` file from the server to your local machine.
8. Copy the required binary files.
    ```bash
    cd /var/task/imagemagick
    sudo tar zcf bin.tar.gz bin/
    cp bin.tar.gz /home/ec2-user/bin.tar.gz 
    ```
    Copy the `bin.tar.gz` file from the server to your local machine.
9. Copy the XML files required by ImageMagick.
    ```bash
    cd /etc/
    sudo tar zcf etc.tar.gz ImageMagick-6/
    cp etc.tar.gz /home/ec2-user/etc.tar.gz
    ```
    Copy the `etc.tar.gz` file from the server to your local machine.
10. Close the SSH session.
11. On your local machine, extract the contents of the 3 `*.tar.gz` files.
12. Download ghostscript from https://github.com/ArtifexSoftware/ghostpdl-downloads/releases/download/gs9533/ghostscript-9.53.3-linux-x86_64.tgz and extract the ghostscript binary into the `bin/` folder and rename it to `gs`. Run `chmod +x bin/gs` to make it executable.
13. Compress the 3 - `lib`, `bin` and `etc` - folders into a ZIP file. The tree structure of the ZIP file would look like
    ```
    file.zip/
    |-- bin
    |   |-- convert
    |   |-- ...
    |   `-- gs
    |-- etc
    |   `-- ImageMagick-6
    |       |-- coder.xml
    |       |-- ...
    |       `-- type.xml
    `-- lib
        |-- ImageMagick-6.9.10
        |   |-- config-Q16
        |   |   `-- configure.xml
        |   `-- modules-Q16
        |       |-- coders
        |       |   |-- aai.la
        |       |   |-- ...
        |       |   `-- yuv.so
        |       `-- filters
        |           |-- analyze.la
        |           `-- analyze.so
        |-- ImageMagick-6.9.11
        |   |-- config-Q8
        |   |   `-- configure.xml
        |   `-- modules-Q8
        |       |-- coders
        |       |   |-- aai.la
        |       |   |-- ...
        |       |   `-- yuv.so
        |       `-- filters
        |           |-- analyze.la
        |           `-- analyze.so
        |-- libICE.so.6
        |-- ...
        `-- libz.so.1
    ```
    I have used `...` wherever the folder contained more than 2 files to denote that there are more files present.
14. Create a Python 3.8 runtime compatible layer on AWS Lambda and use the ZIP created in step 13.
15. Add the layer to your AWS Lambda function code.
16. Update environment variables in your lambda function.
    ```python
    import os

    os.environ["PATH"] = f"/opt/bin:{os.environ['PATH']}"
    os.environ["LD_LIBRARY_PATH"] = f"/opt/lib:{os.environ['LD_LIBRARY_PATH']}"
    os.environ["MAGICK_HOME"] = "/opt/"
    os.environ["WAND_MAGICK_LIBRARY_SUFFIX"] = "-6.Q8"
    os.environ["MAGICK_CONFIGURE_PATH"] = "/opt/etc/ImageMagick-6/"
    os.environ["MAGICK_CODER_MODULE_PATH"] = "/opt/lib/ImageMagick-6.9.11/modules-Q8/coders/"
    ```

Note: If the size of the uncompressed ZIP file is too large and you reach AWS Lambda size limits, remove the binaries that you don't need from the `bin/` folder. In my case, I only kept `Magick-config`, `MagickCore-config`, `MagickWand-config`, `Wand-config`, `convert` and `gs` and removed others.
