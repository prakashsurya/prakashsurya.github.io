+++
date = "2017-09-08T00:00:00-08:00"
lastmod = "2017-09-08T00:00:00-08:00"
title = "Performance Testing Results for OpenZFS #447"
draft = false
+++

The following are links to the Jupyter notebooks that describe the
performance testing that I did for [OpenZFS #447][pr], and the results
of that testing:

 - [Max Rate Submit on HDDs][max-rate-submit-hdd]
 - [Max Rate Submit on SSDs][max-rate-submit-ssd]
 - [Fixed Rate Submit on HDDs][fixed-rate-submit-hdd]
 - [Fixed Rate Submit on SSDs][fixed-rate-submit-ssd]

Additionally, a compressed tarball with all the raw data used to
generate those Jupyter notebooks can be found [here][tarball].

[pr]: https://github.com/openzfs/openzfs/pull/447
[tarball]: openzfs-447-perf.tar.xz
[max-rate-submit-hdd]: {{< nbviewer "openzfs-447-perf-max-rate-submit-hdd.ipynb" >}}
[max-rate-submit-ssd]: {{< nbviewer "openzfs-447-perf-max-rate-submit-ssd.ipynb" >}}
[fixed-rate-submit-hdd]: {{< nbviewer "openzfs-447-perf-fixed-rate-submit-hdd.ipynb" >}}
[fixed-rate-submit-ssd]: {{< nbviewer "openzfs-447-perf-fixed-rate-submit-ssd.ipynb" >}}
