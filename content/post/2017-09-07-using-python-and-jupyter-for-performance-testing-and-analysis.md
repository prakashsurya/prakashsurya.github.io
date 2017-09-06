+++
date = "2017-09-07T00:00:00-08:00"
lastmod = "2017-09-07T00:00:00-08:00"
title = "Using Python and Jupyter for Performance Testing and Analysis"
draft = false
+++

## Introduction

I recently worked on some changes to the OpenZFS ZIL (see [here][pr]),
and in the context of working on that project, I discovered some new
tools that helped me run my performance tests and analyze their
results. What follows is some notes on the tools that I used, and how I
used them.

## Quick Overview

Before I dive into the details of how I used these tools, I wanted to
quickly go over what the tools were:

 - [fio][fio] was used to generate the workload, and provide statistics
   about the performance from the application's perspective.

 - [Jupyter][jupyter] was used for analysis of the test results; e.g.
   performing data manipulations, generating visualizations, and
   presenting the data/visualizations along with text explanations in
   a unified document.

 - [Python][python] was used for everything; e.g. running the tests,
   capturing the results, analysis of the data, and generating
   visualizations were all driven by python scripts. In addition to the
   core language, the following modules were used:
   [sh](https://amoffat.github.io/sh/),
   [pandas](http://pandas.pydata.org/),
   [numpy](http://www.numpy.org/),
   [seaborn](https://seaborn.pydata.org/),
   and [matplotlib](https://matplotlib.org/).

 - [pyenv][pyenv] and [pyenv-virtualenv][pyenv-virtualenv] were used to
   provide an isolated Python environment.

 - [GitHub][github] and [nbviewer][nbviewer] were used to share the
   Jupyter notebooks, which contained the results of the tests.

## Install Prerequisites on Ubuntu 16.04

Before we can use the above referenced tools, we first must download,
build, and/or install them such that they are available for us to use.
Below is some instructions for how to do that on an Ubuntu 16.04 VM that
I used; if using another operating system, the specific commands may not
work, but hopefully what's included here can be easily adapted.

### Install Build Dependencies

    $ sudo apt install -y git build-essential zlib1g-dev \
        libbz2-dev libssl-dev libreadline-dev libsqlite3-dev

### Install "pyenv" and "pyenv virtualenv"

    $ git clone https://github.com/pyenv/pyenv ~/.pyenv
    $ git clone https://github.com/pyenv/pyenv-virtualenv.git \
        ~/.pyenv/plugins/pyenv-virtualenv

    $ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bash_profile
    $ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bash_profile
    $ echo 'eval "$(pyenv init -)"' >> ~/.bash_profile
    $ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bash_profile

### Install Python 3 and Create Virtual Environment

    $ pyenv install 3.6.2
    $ pyenv virtualenv
    $ pyenv virtualenv 3.6.2 jupyter-example
    $ mkdir ~/jupyter-example
    $ cd ~/jupyter-example
    $ echo jupyter-example > .python-version
    $ pip install jupyter pandas numpy seaborn matplotlib sh

### Build and Install "fio"

    $ git clone https://github.com/axboe/fio ~/fio
    $ cd ~/fio
    $ make
    $ sudo make install

### Install Additional Utilities

    $ sudo apt install -y zfsutils-linux jq sysstat

## Generate Results Data Required for Analysis

Now that all of the necessary tools have been installed, we can write a
Python script that uses these tools to run the tests and collect any
data that will be needed for proper analysis.

### Generate "fio" Test Configuration

We'll be using "fio" to generate the workload. Let's first create the
configuration file that will be passed to fio, which tells it how to
behave:

    $ cat > ~/jupyter-example/workload.fio <<EOF
    [global]
    group_reporting
    clocksource=cpu
    ioengine=psync
    fallocate=none
    rw=write
    blocksize=8k
    time_based
    iodepth=1
    thread=0
    direct=0
    sync=1

    [workload]
    EOF

### Generate Python Script to Run "fio" Tests

Now we'll create the python script that will be used to:

  1. create a ZFS pool and dataset from a known set of disks
  2. run "fio", such that it uses the ZFS dataset previously created
  3. run "iostat", collecting disk metrics concurrently while "fio" runs
  4. copy the data generated from "fio" and "iostat" to a "results"
     directory so they can be analyzed at a later time

Here's what the script may look like:

    $ cat > ~/jupyter-example/workload.py <<EOF
    #!/usr/bin/env python3

    import sh
    import tempfile

    def fio(directory, numjobs, disks, runtime=60):
      with tempfile.TemporaryDirectory(dir='/var/tmp') as tempdir:
        procs = []

        for d in disks:
          iostat = sh.iostat('-dxy', d, '1',
                             _piped=True, _bg=True, _bg_exc=False)
          procs.append(iostat)

          grep = sh.grep(iostat, d, _bg=True, _bg_exc=False,
                         _out='{:s}/iostat-{:s}.txt'.format(tempdir, d))
          procs.append(grep)

        sh.sudo.fio('--directory={:s}'.format(directory),
                    '--size={:.0f}M'.format(2**20 / numjobs),
                    '--numjobs={:d}'.format(numjobs),
                    '--runtime={:d}'.format(runtime),
                    '--output={:s}/{:s}'.format(tempdir, 'fio.json'),
                    '--output-format=json+',
                    './workload.fio')

        for p in procs:
          try:
            sh.sudo.kill(p.pid)
            p.wait
          except (sh.ErrorReturnCode_1, sh.SignalException_SIGTERM):
            pass

        directory = 'results/{:d}-disks/{:d}-jobs'.format(len(disks), numjobs)
        sh.mkdir('-p', directory)
        sh.rm('-rf', directory)
        sh.cp('-r', tempdir, directory)
        sh.chmod('755', directory)

    def test(disks):
      for numjobs in [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]:
        try:
          sh.sudo.zpool('create', '-f', 'tank', disks)
          sh.sudo.zfs('create', '-o', 'recsize=8k', 'tank/dozer')
          fio('/tank/dozer', numjobs, disks)
        finally:
          sh.sudo.zpool('destroy', 'tank')

    def main():
      sh.rm('-rf', 'results')
      sh.mkdir('results')

      test(['sdb'])
      test(['sdb', 'sdc'])
      test(['sdb', 'sdc', 'sdd'])

    if __name__ == '__main__':
      main()
    EOF

### Run Python Script to Generate Results

The python script created in the previous section can then be run, which
will generate a "results" directory with all of the raw data from "fio"
and "iostat":

    (jupyter-example) $ time python3 workload.py

    real    34m55.771s
    user    2m19.040s
    sys     12m20.656s

    (jupyter-example) $ ls results/*
    results/1-disks:
    1024-jobs  128-jobs  16-jobs  1-jobs  256-jobs  2-jobs  32-jobs  4-jobs  512-jobs  64-jobs  8-jobs

    results/2-disks:
    1024-jobs  128-jobs  16-jobs  1-jobs  256-jobs  2-jobs  32-jobs  4-jobs  512-jobs  64-jobs  8-jobs

    results/3-disks:
    1024-jobs  128-jobs  16-jobs  1-jobs  256-jobs  2-jobs  32-jobs  4-jobs  512-jobs  64-jobs  8-jobs

    (jupyter-example) $ ls results/*-disks/1024-jobs
    results/1-disks/1024-jobs:
    fio.json  iostat-sdb.txt

    results/2-disks/1024-jobs:
    fio.json  iostat-sdb.txt  iostat-sdc.txt

    results/3-disks/1024-jobs:
    fio.json  iostat-sdb.txt  iostat-sdc.txt  iostat-sdd.txt

As one can see, each unique test configuration (i.e. number of disks in
the zpool, and number of "fio" threads) has its own directory containing
the results for that specific test configuration. The "fio.json"
contains the JSON formatted output from "fio", and the "iostat-\*.txt"
files contains the output from "iostat" for each specific disk.

Here's a quick inspection of one of the "fio" files:

    (jupyter-example) $ head results/2-disks/32-jobs/fio.json
    {
      "fio version" : "fio-3.0-48-g83a9",
      "timestamp" : 1504768846,
      "timestamp_ms" : 1504768846239,
      "time" : "Thu Sep  7 07:20:46 2017",
      "global options" : {
        "directory" : "/tank/dozer",
        "size" : "32768M",
        "runtime" : "60",
        "clocksource" : "cpu",

    (jupyter-example) $ jq -Mr .jobs[0].write.lat_ns results/2-disks/32-jobs/fio.json
    {
      "min": 712505,
      "max": 312646560,
      "mean": 5412690.837107,
      "stddev": 7953187.998254
    }

    (jupyter-example) $ jq -Mr .jobs[0].write.iops results/2-disks/32-jobs/fio.json
    5906.296339

From this, we can see that on average, each write made by fio took
roughly 5ms to complete. Additionally, fio averaged about 5.9K IOPs
during that specific test's runtime.

Similarly, we can briefly look at the iostat data collected for that
same test configuration:

    (jupyter-example) $ head results/2-disks/32-jobs/iostat-sdb.txt
    sdb               0.00     0.00    0.00  304.00     0.00 18116.50   119.19     0.54    1.78    0.00    1.78   1.45  44.00
    sdb               0.00     0.00    0.00  667.00     0.00 60725.50   182.09     1.23    1.84    0.00    1.84   1.27  84.40
    sdb               0.00     0.00    0.00  500.00     0.00 49707.50   198.83     0.92    1.86    0.00    1.86   1.58  78.80
    sdb               0.00     0.00    0.00  502.00     0.00 45371.00   180.76     0.91    1.80    0.00    1.80   1.50  75.20
    sdb               0.00     0.00    0.00  523.00     0.00 47920.50   183.25     1.07    2.06    0.00    2.06   1.48  77.20
    sdb               0.00     0.00    0.00  598.00     0.00 58390.50   195.29     1.38    2.29    0.00    2.29   1.55  92.80
    sdb               0.00     0.00    0.00  393.00     0.00 37456.00   190.62     0.98    2.49    0.00    2.49   2.06  80.80
    sdb               0.00     0.00    0.00  527.00     0.00 48050.00   182.35     1.08    2.05    0.00    2.05   1.53  80.40
    sdb               0.00     0.00    0.00  590.00     0.00 55418.00   187.86     1.24    2.10    0.00    2.10   1.48  87.60
    sdb               0.00     0.00    0.00  663.00     0.00 64226.50   193.75     1.18    1.78    0.00    1.78   1.32  87.20

    (jupyter-example) $ head results/2-disks/32-jobs/iostat-sdc.txt
    sdc               0.00     0.00    0.00  295.00     0.00 25714.00   174.33     0.50    1.69    0.00    1.69   1.40  41.20
    sdc               0.00     0.00    0.00  640.00     0.00 59921.00   187.25     1.22    1.90    0.00    1.90   1.33  85.20
    sdc               0.00     0.00    0.00  509.00     0.00 53581.50   210.54     1.00    1.96    0.00    1.96   1.70  86.40
    sdc               0.00     0.00    0.00  512.00     0.00 41799.00   163.28     0.95    1.85    0.00    1.85   1.48  75.60
    sdc               0.00     0.00    0.00  538.00     0.00 48975.50   182.07     1.09    2.03    0.00    2.03   1.50  80.80
    sdc               0.00     0.00    0.00  601.00     0.00 50568.00   168.28     1.36    2.25    0.00    2.25   1.50  90.40
    sdc               0.00     0.00    0.00  395.00     0.00 38467.00   194.77     1.03    2.61    0.00    2.61   2.14  84.40
    sdc               0.00     0.00    0.00  540.00     0.00 49031.00   181.60     1.03    1.91    0.00    1.91   1.45  78.40
    sdc               0.00     0.00    0.00  580.00     0.00 53762.50   185.39     1.28    2.21    0.00    2.21   1.49  86.40
    sdc               0.00     0.00    0.00  648.00     0.00 57146.50   176.38     1.19    1.83    0.00    1.83   1.31  85.20

These files contain the output of "iostat -dxy" for each device in the
ZFS pool used for the specific test configuration. In this case, the
pool consisted of 2 disks, "sdb" and "sdc". Each line in the file
represents a 1 second interval.

For completeness, since the iostat column headers are not included in
these files, here's what they are:

    Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util

Thus, we can see that usually an IO request sent to these disks was
serviced in less than 2ms, judging by the `svctm` column.

## Analyzing Results Data with Pandas and Jupyter

Now that the data had been gathered from "fio" and "iostat" for all of
the test configurations, it was time to parse and analyze the data.

### Parsing "fio" Results with Pandas

To parse the "fio" results files, I used a combination of the previously
used "sh" Python module, and the "jq" command. Here's some code that
would parse the the IOPs reported by fio for each test configuration,
and print the results as a table:

    (jupyter-example) $ python3
    Python 3.6.2 (default, Sep  6 2017, 21:23:54)
    [GCC 5.4.0 20160609] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import pandas
    >>> import sh
    >>>
    >>> numjobs = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
    >>> numdisks = [1, 2, 3]
    >>>
    >>> jq = sh.jq.bake('-M', '-r')
    >>> iops = pandas.DataFrame()
    >>> for i in numdisks:
    ...   tmp = []
    ...   for j in numjobs:
    ...     data = jq('.jobs[0].write.iops',
    ...               'results/{:d}-disks/{:d}-jobs/fio.json'.format(i, j))
    ...     tmp.append(float(data.strip()))
    ...   iops['{:d} disks'.format(i)] = pandas.Series(tmp, numjobs)
    ...
    >>> print(iops)
              1 disks       2 disks       3 disks
    1      693.355111    539.923004    399.353355
    2      946.218459    817.014473    465.543389
    4     1487.733742   1367.766810    817.556196
    8     2597.526832   2714.935671    902.661356
    16    3958.774225   4132.298054   3854.193054
    32    6192.515329   5906.296339   5730.420583
    64    7470.722522   8216.463923   8146.322770
    128   8649.000333   9370.233395   9525.552187
    256   8601.175804  10942.662150  10868.330780
    512   7327.968610  11595.399661  11321.241803
    1024  7792.668597  11137.351190  11391.167610

Similarly, here's code that would report the average write latency (in
nanoseconds) reported by fio for each test configuration:

    (jupyter-example) $ python3
    Python 3.6.2 (default, Sep  6 2017, 21:23:54)
    [GCC 5.4.0 20160609] on linux
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import pandas
    >>> import sh
    >>>
    >>> numjobs = [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
    >>> numdisks = [1, 2, 3]
    >>>
    >>> jq = sh.jq.bake('-M', '-r')
    >>> lat = pandas.DataFrame()
    >>> for i in numdisks:
    ...   tmp = []
    ...   for j in numjobs:
    ...     data = jq('.jobs[0].write.lat_ns.mean',
    ...               'results/{:d}-disks/{:d}-jobs/fio.json'.format(i, j))
    ...     tmp.append(float(data.strip()))
    ...   lat['{:d} disks'.format(i)] = pandas.Series(tmp, numjobs)
    ...
    >>> print(lat)
               1 disks       2 disks       3 disks
    1     1.438905e+06  1.848592e+06  2.500436e+06
    2     2.110580e+06  2.444135e+06  4.292531e+06
    4     2.685592e+06  2.919950e+06  4.889253e+06
    8     3.076803e+06  2.943452e+06  8.858595e+06
    16    4.038235e+06  3.868267e+06  4.148222e+06
    32    5.163324e+06  5.412691e+06  5.580323e+06
    64    8.561707e+06  7.783084e+06  7.850346e+06
    128   1.478921e+07  1.364914e+07  1.342665e+07
    256   2.973660e+07  2.333695e+07  2.352923e+07
    512   6.972930e+07  4.406545e+07  4.511673e+07
    1024  1.305812e+08  9.137294e+07  8.931305e+07

### Visualizing "fio" Results with Jupyter

While the text-based tables shown in the previous section are better
than nothing, Jupyter can be used to execute this parsing code, and
visualize the data using Python's "matplotlib" graphing module.

The Jupyter notebook software is easy to start up:

    (jupyter-example) $ jupyter notebook --ip=0.0.0.0

Then one can navigate to the server running the notebook using a web
browser; e.g. I would enter `http://ps-jupyter.dcenter.delphix.com:8888`
into my browser.

If the `jupyter` command is run from a local shell (e.g. on one's
workstation), the `--ip` option can be ommitted, and the command will
automatically attempt to open a new browser window with the notebook's
URL already populated.

Here's an [example][fio-notebook] Jupyter notebook, migrating the parsing
code from the prior section into the notebook, and adding some more logic
to generate graphs rather than text-based tables.

### Visualizing "iostat" Results with Jupyter

Similarly, the data from "iostat" can also be parsed and visualized just
like the "fio" data. Rather than repeat the explanations from the prior
sections, I'll simply link directly to the [example][iostat-notebook]
Jupyter notebook; which contains the code for both parsing the "iostat"
data files, as well as generating graphs from that data.

[pr]: https://github.com/openzfs/openzfs/pull/447
[fio]: https://github.com/axboe/fio
[jupyter]: http://jupyter.org/
[python]: https://www.python.org/
[pyenv]: https://github.com/pyenv/pyenv
[pyenv-virtualenv]: https://github.com/pyenv/pyenv-virtualenv
[github]: https://github.com/
[nbviewer]: https://nbviewer.jupyter.org/
[fio-notebook]: {{< nbviewer "visualizing-fio-results-with-jupyter.ipynb" >}}
[iostat-notebook]: {{< nbviewer "visualizing-iostat-results-with-jupyter.ipynb" >}}
