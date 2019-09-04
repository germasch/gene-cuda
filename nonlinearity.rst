
Nonlinearity
============

Performance
------------------------

Starting out from the FFTs, I worked outward until I got all of the
nonlinearity calculation ported, ie. ``calc_arakawa_nonlinearity_df``.

I ran the ``big-8`` case (IIRC) on a single node on Summit, and I used
only 4 MPI procs, so either 4 GPUs or 4 CPU cores
(baseline). Obviously, that's an unfair comparison, so I'll divide the
CPU timing by 7 in the following to account for that one nodes has 42
cores / 6 GPUs.

The CPU code, measured with score-p/scalasca took 222s/7 = 32s. The
GPU version took 25s. At least that's faster, but obviously not by
much. In my implementation, I explicitly do host -> device copies and
v.v. At the point where most of the code is ported, these will move
out (go away?), so to look at the potential of the GPU port, I tried
to figure out how much time is spent in the actual kernel
computations. Unfortunately, for whatever reason, scalasca didn't want
to give me that information. So I used nvprof and I got the following::

                    0.15%  104.39ms       168  621.38us  407.68us  847.55us  void composite_2way_fft_c2r<unsigned int=72, unsigned int=8, unsigned int=16, padding_t=1, twiddle_t=0, loadstore_modifier_t=2, unsigned int=9, layout_t=0, unsigned int, double>(kernel_arguments_t<unsigned int>)
                    0.12%  85.837ms       168  510.93us  332.09us  819.77us  _Z13kernel_assignIN6thrust7complexIdEEZN71_INTERNAL_49_tmpxft_0000fc59_00000000_6_cuda_nonlinear_cpp1_ii_35ab1dc933launch_transpose_and_extend_cmplxIS2_EEv9ArrayViewIT_Li3EN5Space6DeviceEES9_EUliiiE_EvS9_T0_
                    0.11%  74.173ms        84  883.01us  879.58us  885.43us  void kernel_assign<double, Minus<Multiply<ArrayView<double, int=3, Space::Device>, ArrayView<double, int=3, Space::Device>>, Multiply<ArrayView<double, int=3, Space::Device>, ArrayView<double, int=3, Space::Device>>>>(ArrayView<double, int=3, Space::Device>, double)
                    0.07%  52.319ms         1  52.319ms  52.319ms  52.319ms  ij_deriv_kernel(int, int, int, double2 const *, int, double const *, double2 const *, double2*)
                    0.06%  44.891ms        42  1.0688ms  1.0666ms  1.0711ms  void kernel_assign<double, Plus<Plus<Plus<Plus<Plus<MultiplyScalar<double, ArrayView<double, int=3, Space::Device>>, MultiplyScalar<double, ArrayView<double, int=3, Space::Device>>>, MultiplyScalar<double, ArrayView<double, int=3, Space::Device>>>, MultiplyScalar<double, ArrayView<double, int=3, Space::Device>>>, MultiplyScalar<double, ArrayView<double, int=3, Space::Device>>>, Plus<Multiply<UnaryMinus<ArrayView<double, int=3, Space::Device>>, ArrayView<double, int=3, Space::Device>>, Multiply<ArrayView<double, int=3, Space::Device>, ArrayView<double, int=3, Space::Device>>>>>(ArrayView<double, int=3, Space::Device>, double)
                    0.05%  32.995ms        84  392.80us  391.52us  396.32us  void composite_2way_fft_r2c<unsigned int=72, unsigned int=8, unsigned int=16, padding_t=1, twiddle_t=0, loadstore_modifier_t=2, unsigned int=9, layout_t=0, unsigned int, double>(kernel_arguments_t<unsigned int>)
                    0.03%  20.699ms        84  246.42us  245.92us  247.10us  void kernel_assign<thrust::complex<double>, MultiplyScalar<double, ArrayView<thrust::complex<double>, int=3, Space::Device>>>(ArrayView<double, int=3, Space::Device>, thrust::complex<double>)
                    0.02%  14.250ms        42  339.28us  338.78us  339.87us  _Z13kernel_assignIN6thrust7complexIdEEZN12NonLinearity9transposeI14MultiplyScalarId4PlusI9ArrayViewIS2_Li3EN5Space6DeviceEESA_EEEEvSA_RK10expressionIT_EEUliiiE_EvS7_ISE_Li3ES9_ET0_
                    0.02%  14.153ms        42  336.99us  336.54us  337.60us  _Z13kernel_assignIN6thrust7complexIdEEZN9Prefactor13multiply_withE9ArrayViewIS2_Li3EN5Space6DeviceEES7_EUliiiE_EvS4_IT_Li3ES6_ET0_
                    0.01%  9.8096ms        42  233.56us  232.99us  234.30us  _Z13kernel_assignIN6thrust7complexIdEEZN12NonLinearity7y_derivE9ArrayViewIS2_Li3EN5Space6DeviceEEEUliiiE_EvS4_IT_Li3ES6_ET0_

This adds up to about 450 ms total. This number needs to be multiplied by
4 GPUs, so that's 1.8s for the nonlinearity calculation, which still
compares to the estimated 32s on 7 CPU cores, so a speed-up of about
18x, which I think sounds very promising.

Changes to existing code
------------------------

Getting good performance on the GPU requires lots of parallelism. The
current r.h.s. computation is using tiling that's optimized for CPUs
and their caches. There is an outermost loop over blocks, where the
granularity can be easily changed by the ``nblocks`` parameter. However,
at the next level down, each blocks is processed xy-slice by slice,
e.g., FFTs are executed for one xy slice at a time. That is too little
work to effectively occupy a GPU, beside the fact that overheads for
kernel launches etc also likely become significant at that point. So
while I started at the single FFT level, I ended up moving the
implementation up until I had enough work, which is at the
``calc_arakawa_nonlinearity_df`` level. The way it's currently hooked
in is not particularly pretty and still needs work, but to give some
idea, here's the relevant diff:

.. literalinclude:: p1.diff
    :linenos:
    :language: diff
   
.. literalinclude:: p2.diff
    :linenos:
    :language: diff

In the end, I think it might be possible to just have another subclass
of `nonlinear_term_t` that forwards to the C/C++ side.
