# IIPImage on Windows
Build [IIPImage server](https://github.com/ruven/iipsrv) and install it with Apache on Windows 

For feedback please use the [Issues](https://github.com/filak/IIPImage-on-Windows/issues).

> **\<...\>** is a placeholder for absolute path to your specific folder

## Install VS 2017 Community
https://visualstudio.microsoft.com/cs/vs/older-downloads/

Be sure to have selected the following components in installer:
- VC++ 2017 version... latest
- Visual Studio C++ core features
- Windows 10 SDK
- Windows Universal CRT SDK
- Windows Universal C Runtime

## Install vcpkg
https://github.com/microsoft/vcpkg 

Clone/download the repo to **\<VCPKG_ROOT\>** folder (simple path - no spaces!)

Open Command prompt and run:

    >  cd <VCPKG_ROOT>

- run the bootstrap script:  

```
>  bootstrap-vcpkg -disableMetrics
```
 
- install packages: 

```
vcpkg install zlib
vcpkg install tiff
vcpkg install libjpeg-turbo
vcpkg install openjpeg
vcpkg install fastcgi
```

- run the integrate command: 

```
>  vcpkg integrate install
```

## Build IIPImage server
Clone/download IIPImage repo 
https://github.com/ruven/iipsrv 

to **\<IIP_HOME\>** folder

Copy 

    <VCPKG_ROOT>\installed\x86-windows\lib\*.lib 
    
to 

     <IIP_HOME>\windows\dependencies\includes\ 

Start Visual Studio and open the project file:

     <IIP_HOME>\windows\Visual Studio 2017\iipsrv.vcxproj

Select **Release** and target **x86**

Adjust Project properties -> C/C++ -> AdditionalIncludeDirectories:

     <IIP_HOME>\windows\dependencies\includes;<IIP_HOME>\fcgi\include;%(AdditionalIncludeDirectories)

Adjust Project properties -> C/C++ -> Preprocessor -> Preprocessor Definitions - remove:

     HAVE_KAKADU; HAVE_MEMCACHED; HAVE_TIME_H;

Run Build & Pray - successful build is located in: 

     <IIP_HOME>\windows\Visual Studio 2017\Release\Win32

## Install Apache and IIPImage server

Define in your *etc\hosts* file:

     iip.test   127.0.0.1

Create **\<IIPDATA\>** folder for your testing images and copy there some files
  
- Use either JPEG2000 or Tiled Multi-Resolution TIFF - see https://iipimage.sourceforge.io/documentation/images/

Visit ApacheLounge https://www.apachelounge.com/download/ and download Zip files:
  
- A) Apache 2.4.x Win32
- B) mod_fcgid Win32

Unpack A to **\<APACHE_HOME\>** folder
  
Unpack B\\mod_fcgid...\\mod_fcgid.so to \<APACHE_HOME\>\\modules folder

Update \<APACHE_HOME\>\\conf\\httpd.conf:

```  
Define SRVROOT "<APACHE_HOME>"

## Uncomment:
LoadModule headers_module modules/mod_headers.so

## Add:
LoadModule fcgid_module modules/mod_fcgid.so

## Append to end of file:
Define IIPDATA "<IIPDATA>"

<IfModule fcgid_module>
# IIPImageServer configuration
Include conf/httpd-iipsrv.conf
</IfModule>

<VirtualHost *:80>
  ServerName iip.test
  DocumentRoot "${IIPDATA}"

  ## Disable caching to avoid IIP crashing on Windows
  ## - see https://github.com/ruven/iipsrv/issues/208
  Header always set Cache-Control "max-age=0, no-cache, no-store, must-revalidate"
  Header always set Pragma "no-cache"

  <Directory "${IIPDATA}">
     Require all granted
  </Directory>
</VirtualHost>
```  

Create *httpd-iipsrv.conf* and place it next to httpd.conf

https://iipimage.sourceforge.io/documentation/server/
  
```
ScriptAlias /fcgi-bin/ "${SRVROOT}/iipsrv/"

<Directory "${SRVROOT}/iipsrv/">
  Require all granted
  # Set the module handler
  AddHandler fcgid-script .fcgi
</Directory>

# Set our environment variables for the IIP server
FcgidInitialEnv FILESYSTEM_PREFIX "${IIPDATA}"
## VERBOSITY: 1 to 6
FcgidInitialEnv VERBOSITY "6"
FcgidInitialEnv LOGFILE "${SRVROOT}/logs/iipsrv.log"
FcgidInitialEnv MAX_IMAGE_CACHE_SIZE "50"
FcgidInitialEnv JPEG_QUALITY "90"
FcgidInitialEnv MAX_CVT "3500"
FcgidInitialEnv MAX_LAYERS "-1"
#FcgidInitialEnv MEMCACHED_SERVERS "127.0.0.1:11211"
FcgidInitialEnv ALLOW_UPSCALING "1"
FcgidInitialEnv EMBED_ICC "0"

# Define the idle timeout as unlimited and the number of processes we want
FcgidIdleTimeout 0
FcgidMaxProcessesPerClass 10  
```  

Copy \*.dll and iipsrv.fcgi files from
  
    <IIP_HOME>\windows\Visual Studio 2017\Release\Win32\ 

to your 
  
    <APACHE_HOME>\iipsrv folder

Open Command prompt and run:

```
>  cd <APACHE_HOME>\bin
>  httpd.exe
```

Open browser and check some testing images:
      
- http://iip.test/fcgi-bin/iipsrv.fcgi?FIF=/a.jp2&HEI=128&CVT=jpeg
- http://iip.test/fcgi-bin/iipsrv.fcgi?IIIF=/a.jp2/full/700,/0/default.jpg
- http://iip.test/fcgi-bin/iipsrv.fcgi?FIF=/PalaisDuLouvre.tif&HEI=128&CVT=jpeg      
- http://iip.test/fcgi-bin/iipsrv.fcgi?IIIF=/PalaisDuLouvre.tif/full/700,/0/default.jpg

Check IIPImage log:

      <APACHE_HOME>\logs\iipsrv.log

> Stop Apache webserver with: CTRL+C
