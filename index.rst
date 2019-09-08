.. GENE CUDA documentation master file, created by
   sphinx-quickstart on Tue Sep  3 15:51:29 2019.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

GENE CUDA Notes
=====================================

Let start with some general comments and caveats:

* While I started out hand-implementing CUDA kernels at a time, I got
  pretty carried away later, so realize that some of the stuff below is pretty out
  there. So this is really meant just as a basis for discussions.

* This work currently only supports the case where the whole x-y plane
  is available on a single proc. That's not a fundamental limitation,
  but other cases will require communication of things that are in
  device memory, so CUDA-aware MPI probably would be helpful there.

* The test case I've been running is arbitrarily chosen, it's
  basically ``big-8``. YMMV.

* While my work-in-progress implementation is working, it's ugly,
  missing lots of pieces, etc., and generally very far from being in
  mergeable shape.

.. toctree::
   :maxdepth: 2
   :caption: Contents

   cmake
   nonlinearity
   array
   case1/case1
