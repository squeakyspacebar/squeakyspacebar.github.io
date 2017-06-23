---
layout: post
title: "ranlib Issue While Using CheckInstall"
date: 2012-07-10
---
I've been using [CheckInstall](http://asic-linux.com.mx/~izto/checkinstall/)
regularly now as a handy tool to keep track of all my source installations, but
I've run into some occasional issues. At some points, CheckInstall outputs the
following:

```
ranlib: could not create temporary file whilst writing archive: No more archived
files
```

I encountered this issue when attempting to use CheckInstall with OpenSSL and
APR-Util, and found [a solution by Ivan Borodin](http://checkinstall.izto.org/
cklist/msg40472.html). Essentially, although the exact errors for each
application may differ, they seem to be related to an inability to create,
modify, or access directories. If a directory is missing, all you have to do is
figure out which one the installation is attempting to access and create it.

For the OpenSSL issue I was encountering (outlined in the linked solution
thread), the ranlib error message was preceded by this (my issue was similar):

```
make[3]: Nothing to be done for `install-exec-am'.
test -z "/usr/lib/enlightenment/preload" || mkdir -p --
"/usr/lib/enlightenment/preload"
 /bin/sh ../../libtool --mode=install /usr/local/bin/install -c 
'e_precache.la' '/usr/lib/enlightenment/preload/e_precache.la'
/usr/local/bin/install -c .libs/e_precache.so
/usr/lib/enlightenment/preload/e_precache.so
/usr/local/bin/install -c .libs/e_precache.lai
/usr/lib/enlightenment/preload/e_precache.la
/usr/local/bin/install -c .libs/e_precache.a
/usr/lib/enlightenment/preload/e_precache.a
chmod 644 /usr/lib/enlightenment/preload/e_precache.a
ranlib /usr/lib/enlightenment/preload/e_precache.a
```

When I attempted to use CheckInstall with APR-Util, the error I received was:

```
mkdir /usr/local/apr/lib/apr-util-1

ranlib: could not create temporary file whilst writing archive: No more archived
files
```

Which I solved simply by manually creating the directory using:

```
sudo mkdir /usr/local/apr/lib/apr-util-1
```

I ran CheckInstall again and voilà, everything completed successfully.
