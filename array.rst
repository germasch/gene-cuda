
Multi-d CUDA-aware Arrays
==========================

There are manu multi-d array classes for C++, but simple self-contained one that works with CUDA (that I'm aware of), so I started implementing my own, following what `xtensor <https://xtensor.readthedocs.io>`_ does. It makes interfacing with Fortran relatively straightforward:

.. code-block:: c++

   void c_nonlinearity_1(NonLinearity* nl, complex_double* gy_chi_data,
                       const int* gy_chi_shape, complex_double* g2d_data,
                       const int* g2d_shape, complex_double* vexy_data,
                       const int* vexy_shape, complex_double* dgdxy_data,
                       const int* dgdxy_shape, complex_double* localrhs_data,
                       const int* localrhs_shape, bool first,
                       std::array<double, 2>& vexy_max)
   {
     auto gy_chi = gt::view<3>(gy_chi_data, gy_chi_shape);
     auto g2d = gt::view<3>(g2d_data, g2d_shape);
     auto vexy = gt::view<4>(vexy_data, vexy_shape);
     auto dgdxy = gt::view<4>(dgdxy_data, dgdxy_shape);
     auto localrhs = gt::view<3>(localrhs_data, localrhs_shape);
     nl->nonlinearity_1(gy_chi, g2d, vexy, dgdxy, localrhs, first, vexy_max);
   }

These arrays can be accessed in a Fortran-like syntax:

.. code-block:: c++

   out(j, i, b) = gy_chi_re(j, i, b) * dgdxy_re(j, i, 0, b) -
                  g2d_re(j, i, b) * vexy_re(j, i, 0, b);

The arrays could just based around CUDA managed memory allocations, though currently I'm using separate host and device versions and copies are explicit, as in ``copy(nonlin, d_nonlin)``.

Generating kernels
------------------

While I initially wrote kernels by hand, I switched to a lambda-function based approach used by a number of different project in this space, so one can create a kernel like this:

.. code-block:: c++

    launch_functor(tmp_x.shape(), GT_LAMBDA(int j, int i, int b) {
      if (j < tmp_x.shape(0) && i < tmp_x.shape(1)) {
        tmp_x(j, i, b) = std::abs(vexy_re(j, i, 0, b) * pnl_1d(i));
        tmp_y(j, i, b) = std::abs(vexy_re(j, i, 1, b) * pnl_1d(i));
      }
    });


This code creates and launches a CUDA kernel. Obviously, performance will not always be optimal, but it helps getting code ported quickly and after finding performance bottlenecks, one can obviously go back and replace them by more highly-tuned versions. In many cases, they will probably perform okay, since it looks to me like the code is going to be mostly memory-bound, anyway.

Auto-generating kernels
-----------------------

As I was doing this, I realized that the ideas that `xtensor` implements actually fit a lot of what's happening in GENE rather well, and they can be made to work on GPUs. This essentially allows one to write Fortran/Python-like array expressions in C++ and have kernel code be generated from them at compile time, e.g.:

.. code-block:: c++

    auto vex_re = gt::view<3>(vexy_re, {gt::all(), gt::all(), 0});
    auto dgdx_re = gt::view<3>(dgdxy_re, {gt::all(), gt::all(), 0});
    
    nonlin_re = x_deriv_2d_nonlin_transp(nonlin2_wb) +
                (-vex_re * dgdy_re + vey_re * dgdx_re);
    tmp_arr = fftAndTruncate(nonlin_re);

This is pretty expressive (except for the ugliness that comes from C++ not supporting the ``:`` based slice syntax), and it also should help with performance in that it allows to create larger, merged kernels.

