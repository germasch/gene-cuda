
cmake
=====

Based on Markus' work, there is now the possibility to build GENE with
cmake. It has its own list of caveats in terms of incompleteness, etc,
but it's definitely very helpful to Markus' and my work, and hopefully
will be to others. While it might be worthwhile to entirely switch to
cmake at some point, I don't have the ambition to try to get there at
this time, but it'd be helpful to have this available, and in fact I'd
advocate for supporting the GPU build only with cmake (after making
sure that it works for all of the GPU team.) It's very un-invasive, it
just adds a couple of files but doesn't change any existing files.

