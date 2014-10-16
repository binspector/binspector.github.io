---
layout: page
title: "installation"
date: 2014-10-15 21:44
comments: true
sharing: true
footer: true
---

# Dependencies

For all platforms, the following dependencies are needed. On the Mac these can be automatically aquired and placed where necessary. On Windows this is a manual process:

 1. [Binspector](https://github.com/binspector/binspector)
 1. [Adobe Source Libraries](https://github.com/stlab/adobe_source_libraries)
 1. [Adobe Platform Libraries](https://github.com/stlab/adobe_platform_libraries)
 1. [Double-Conversion](https://github.com/stlab/double-conversion)
 1. [Boost 1.54.0](http://sourceforge.net/projects/boost/files/boost/1.54.0/) (in a folder called `boost_libraries`)

# Mac

If you are running a Mac, installation is straightforward. Open up a terminal window to the folder you want to contain your Binspector installation. Be aware that several support folders will be created as siblings to the main `binspector/` folder.

Copy and paste the following into a terminal window:

```
git clone https://github.com/binspector/binspector.git && ./binspector/smoke_test.sh
```

Now go get some coffee. When the process completes the top-level folder should have a `bin/` folder within that contains the Binspector executable. The smoke test will download two sample images (a PNG and a JPEG) from Wikipedia and validate them against the current PNG and JPEG format grammars.

# Windows

Unfortunately Windows installation is not as streamlined as it is on the Mac. You will have to download a number of dependencies and execute the Boost build tool manually.

The dependencies needed should all be in a folder together as siblings. The directory structure should look like the following:

```
my_directory\
  adobe_platform_libraries\
  adobe_source_libraries\
  binspector\
  boost_libraries\
  double-conversion\
```

 1. Run `.\boost_libraries\bootstrap.bat` to build the Boost build tool `b2`.
 1. `cd` into `binspector`
 1. Run `..\boost_libraries\b2`
