+++
date = "2017-12-01T00:00:00-08:00"
lastmod = "2017-12-01T00:00:00-08:00"
title = "Principles of OS Debugging"
draft = false
+++

1. The most important bugs occur at customer sites, cause downtime for
   mission-critical machines, and cannot be deterministically
   reproduced

2. Customers should not and will not perform experiments and debugging
   tasks for us

3. We must be able to perform root-cause analysis from a single crash
   dump and deliver an appropriate fix

4. Debugging support for new subsystems is required at the same time as
   project integration

The above points were taken from slide 5, of Mike Shapiro's presentation
"A Brief Tour of the Modular Debugger" ca. 2000. Slides of the
presentation available [here](mdb-public.pdf).
