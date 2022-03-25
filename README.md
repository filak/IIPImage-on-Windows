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

> Append modifiers for your build target:  :x64-windows  or  :x86-windows  

```
vcpkg install zlib:x64-windows
vcpkg install tiff:x64-windows
vcpkg install openjpeg:x64-windows
vcpkg install fastcgi:x64-windows
```

> libjpeg-turbo is installed as part of tiff package

- run the integrate command: 

```
>  vcpkg integrate install
```

## Download IIPImage server

Clone/download IIPImage repo 
https://github.com/ruven/iipsrv 

to **\<IIP_HOME\>** folder

### (optional) Download libmemcached

Source:  https://github.com/awesomized/libmemcached

Binaries: https://artifacts.m6w6.name/libmemcached

Download latest binaries - copy *libmemcached-awesome-...* sub-folders (bin, include, lib) to: 

    <IIP_HOME\>\libmemcached
    
> This library is 64bit only - so you need to target x64 build

## Build IIPImage server in VS 2017

Start Visual Studio and open the project file:

     <IIP_HOME>\windows\Visual Studio 2017\iipsrv.vcxproj

Select **Release** and target: **x64**

Adjust:   Project properties -> C/C++ -> General -> AdditionalIncludeDirectories

     <IIP_HOME>\fcgi\include;<VCPKG_ROOT>\installed\x64-windows\include;%(AdditionalIncludeDirectories)
     
If you need memcached use:     

     <IIP_HOME>\fcgi\include;<VCPKG_ROOT>\installed\x64-windows\include;<IIP_HOME>\libmemcached\include;%(AdditionalIncludeDirectories)
     
Check/Adjust:   Project properties -> C/C++ -> Preprocessor -> Preprocessor Definitions

     WIN32;NDEBUG;_CONSOLE;HAVE_OPENJPEG;VERSION ...
     
- if you need memcached add ;HAVE_MEMCACHED 
     
Check/Adjust:   Project properties -> Linker -> Input -> Additional Dependencies

     jpeg.lib;libfcgi.lib;tiff.lib;zlib.lib;openjp2.lib;%(AdditionalDependencies)
     
If you need memcached use:

     jpeg.lib;libfcgi.lib;tiff.lib;zlib.lib;openjp2.lib;libmemcached.lib;%(AdditionalDependencies)

If you need memcached adjust:   Project properties -> VC++ Directories -> Library Directories - add:

     ;<IIP_HOME>\libmemcached\lib     

Run Build iipsrv & Pray - successful build is located in: 

     <IIP_HOME>\windows\Visual Studio 2017\Release\...

## Install Apache and IIPImage server

Define in your *etc\hosts* file:

     iip.test   127.0.0.1

Create **\<IIPDATA\>** folder for your testing images and copy there some files
  
- Use either JPEG2000 or Tiled Multi-Resolution TIFF

> For details see: https://iipimage.sourceforge.io/documentation/images/

Visit ApacheLounge https://www.apachelounge.com and download Zip files:
  
- A) Apache 2.4.x 
- B) mod_fcgid 

Unpack A to **\<APACHE_HOME\>** folder
  
Unpack B\\mod_fcgid...\\mod_fcgid.so to \<APACHE_HOME\>\\modules folder

Create <APACHE_HOME>\iipsrv folder

Copy iipsrv.fcgi and \*.dll files from
  
    <IIP_HOME>\windows\Visual Studio 2017\Release\WinXX\ 

to your 
  
    <APACHE_HOME>\iipsrv folder
    
\*.dll files 

- jpeg62.dll
- libfcgi.dll
- liblzma.dll
- openjp2.dll
- tiff.dll
- zlib1.dll

(memcached support:)

- libhashkit.dll
- libmemcached.dll
- libmemcachedprotocol.dll
- libmemcachedutil.dll

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

> For details see: https://iipimage.sourceforge.io/documentation/server/
  
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
