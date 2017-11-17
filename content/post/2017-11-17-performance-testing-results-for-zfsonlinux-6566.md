+++
date = "2017-11-17T00:00:00-08:00"
lastmod = "2017-11-17T00:00:00-08:00"
title = "Performance Testing Results for ZFS on Linux #6566"
draft = false
+++

The following are links to the Jupyter notebooks that describe the
performance testing that I did for [ZFS on Linux #6566][pr], and the
results of that testing:

 - [Max Rate Submit on HDDs][max-rate-submit-hdd]
 - [Max Rate Submit on SSDs][max-rate-submit-ssd]
 - [Fixed Rate Submit on HDDs][fixed-rate-submit-hdd]
 - [Fixed Rate Submit on SSDs][fixed-rate-submit-ssd]

Additionally, a compressed tarball with all the raw data used to
generate those Jupyter notebooks can be found [here][tarball].

[pr]: https://github.com/zfsonlinux/zfs/pull/6566
[tarball]: zfsonlinux-6566-perf.tar.xz
[max-rate-submit-hdd]: {{< nbviewer "zfsonlinux-6566-perf-max-rate-submit-hdd.ipynb" >}}
[max-rate-submit-ssd]: {{< nbviewer "zfsonlinux-6566-perf-max-rate-submit-ssd.ipynb" >}}
[fixed-rate-submit-hdd]: {{< nbviewer "zfsonlinux-6566-perf-fixed-rate-submit-hdd.ipynb" >}}
[fixed-rate-submit-ssd]: {{< nbviewer "zfsonlinux-6566-perf-fixed-rate-submit-ssd.ipynb" >}}
