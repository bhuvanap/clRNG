# clRNG Library

clRNG is an OpenCL library that generates uniform random numbers. This library is
now available as open-source on GitHub. The addition of the clRNG library improves the 
coverage of OpenCL libraries available to developers.


## Introduction

clRNG offers flexibility in development of application code by providing 
interfaces to both the host and the device. The interfaces to the host are developed using 
native C APIs. These interfaces appropriately allocate and initialize data structures 
used on the device. The interfaces to the devices are developed 
using OpenCL C and let you directly invoke the kernel interfaces from your OpenCL kernels 
for random number requirements. This library can generate random numbers in bulk with just
host interfaces.

The clRNG library generates multiple streams—sequences of random numbers— and substreams 
of random numbers for parallel applications and simulations. Currently, the library 
implements four generators: MRG31k3p, MRG32k3a, LFSR113, and 
Philox-based generator (Philox-4×32-10). 

## Library semantic versioning

This clRNG release is part of AMD Compute Libraries (ACL) 1.0 beta 1. It is currently a 
beta release version v1.0.0. We expect developers to use it, evaluate it, and provide 
feedback. We encourage and appreciate feedback on various aspects of the library, such 
as the API design, functionality, implementation choices, and performance. Feedback 
can be provided through the issue tracker on GitHub. Work on the library is in 
progress and items planned for future versions are: addition of generic APIs that allow 
users to switch easily between generators, support for Gaussian and other distributions, 
and the development of C++ style kernel interfaces that OpenCL allows.

The pre-built binaries are available [here][binary_release].

#### clRNG library user documentation

For more information on clRNG, you may refer the following links:

- [**HTML Documentation**][html_document] generated with Doxygen 
- [**Design Document**][Doc_name] - Provides the research paper: "*clRNG*: A Random Number API with Multiple Streams for OpenCL" that details 
the design methodology used in developing the API

## Google Groups

Two mailing lists have been created for the clMath projects:

-   [clmath@googlegroups.com][] - group whose focus is to answer
    questions on using the library or reporting issues

-   [clmath-developers@googlegroups.com][] - group whose focus is for
    developers interested in contributing to the library code itself

## clRNG Wiki

The [project wiki][wiki_page] contains helpful documentation, 
including a [build primer][wiki_build].

## Contributing code

Please refer to and read the [Contributing] document for guidelines on how to contribute code to this open 
source project. The code in the /master branch is considered to be stable, and all pull-requests should 
be made against the /develop branch.


## License

The source for clRNG is licensed under BSD 2.0. 


## Examples

On GitHub, a few examples for using clRNG library are available in `src/client`. Five sample programs are 
included in the repository to illustrate how to use the library. Examples of the compiled client program 
are available in the `bin` subdirectory of the installation package (`$CLRNG_ROOT/bin` under Linux). Note 
that these examples require an OpenCL GPU device. 

The following example may be used as a quick reference to understand, how to use clRNG to generate random 
numbers by directly using the device headers (.clh) in your OpenCL kernel.

```c
#include <stdlib.h>
#include <string.h>

#include "clRNG.h"
#include "mrg31k3p.h"

int main( void )
{
    cl_int err;
    cl_platform_id platform = 0;
    cl_device_id device = 0;
    cl_context_properties props[3] = { CL_CONTEXT_PLATFORM, 0, 0 };
    cl_context ctx = 0;
    cl_command_queue queue = 0;
    cl_program program = 0;
    cl_kernel kernel = 0;
    cl_event event = 0;
    cl_mem bufIn, bufOut;
    float *out;
    char *clrng_root;
    char include_str[1024];
    char build_log[4096];
    size_t i = 0;
    size_t numWorkItems = 64;
    clrngMrg31k3pStream *streams = 0;
    size_t streamBufferSize = 0;
    size_t kernelLines = 0;

    /* Sample kernel that calls clRNG device-side interfaces to generate random numbers */
    const char *kernelSrc[] = {
    "    #define CLRNG_SINGLE_PRECISION                                   \n",
    "    #include <mrg31k3p.clh>                                          \n",
    "                                                                     \n",
    "    __kernel void example(__global clrngMrg31k3pHostStream *streams, \n",
    "                          __global float *out)                       \n",
    "    {                                                                \n",
    "        int gid = get_global_id(0);                                  \n",
    "                                                                     \n",
    "        clrngMrg31k3pStream workItemStream;                          \n",
    "        clrngMrg31k3pCopyOverStreamsFromGlobal(1, &workItemStream,   \n",
    "                                                     &streams[gid]); \n",
    "                                                                     \n",
    "        out[gid] = clrngMrg31k3pRandomU01(&workItemStream);          \n",
    "    }                                                                \n",
    "                                                                     \n",
    };

    /* Setup OpenCL environment. */
    err = clGetPlatformIDs( 1, &platform, NULL );
    err = clGetDeviceIDs( platform, CL_DEVICE_TYPE_GPU, 1, &device, NULL );

    props[1] = (cl_context_properties)platform;
    ctx = clCreateContext( props, 1, &device, NULL, NULL, &err );
    queue = clCreateCommandQueue( ctx, device, 0, &err );

    /* Make sure CLRNG_ROOT is specified to get library path */
    clrng_root = getenv("CLRNG_ROOT");
    if(clrng_root == NULL) printf("\nSpecify environment variable CLRNG_ROOT as described\n");
    strcpy(include_str, "-I ");
    strcat(include_str, clrng_root);
    strcat(include_str, "/cl/include");

    /* Create sample kernel */
    kernelLines = sizeof(kernelSrc) / sizeof(kernelSrc[0]);
    program = clCreateProgramWithSource(ctx, kernelLines, kernelSrc, NULL, &err);
    err = clBuildProgram(program, 1, &device, include_str, NULL, NULL);
    if(err != CL_SUCCESS)
    {
        printf("\nclBuildProgram has failed\n");
        clGetProgramBuildInfo(program, device, CL_PROGRAM_BUILD_LOG, 4096, build_log, NULL);
        printf("%s", build_log);
    }
    kernel = clCreateKernel(program, "example", &err);

    /* Create streams */
    streams = clrngMrg31k3pCreateStreams(NULL, numWorkItems, &streamBufferSize, (clrngStatus *)&err);

    /* Create buffers for the kernel */
    bufIn = clCreateBuffer(ctx, CL_MEM_READ_ONLY | CL_MEM_COPY_HOST_PTR, streamBufferSize, streams, &err);
    bufOut = clCreateBuffer(ctx, CL_MEM_WRITE_ONLY | CL_MEM_HOST_READ_ONLY, numWorkItems * sizeof(cl_float), NULL, &err);

    /* Setup the kernel */
    err = clSetKernelArg(kernel, 0, sizeof(bufIn),  &bufIn);
    err = clSetKernelArg(kernel, 1, sizeof(bufOut), &bufOut);

    /* Execute the kernel and read back results */
    err = clEnqueueNDRangeKernel(queue, kernel, 1, NULL, &numWorkItems, NULL, 0, NULL, &event);
    err = clWaitForEvents(1, &event);
    out = (float *)malloc(numWorkItems * sizeof(out[0]));
    err = clEnqueueReadBuffer(queue, bufOut, CL_TRUE, 0, numWorkItems * sizeof(out[0]), out, 0, NULL, NULL);

    /* Release allocated resources */
    clReleaseEvent(event);
    free(out);
    clReleaseMemObject(bufIn);
    clReleaseMemObject(bufOut);

    clReleaseKernel(kernel);
    clReleaseProgram(program);

    clReleaseCommandQueue(queue);
    clReleaseContext(ctx);

    return 0;
}
```

## Build dependencies

The following lists the build dependecies of the clRNG package:

### Library for Windows

To develop the clRNG library code on a Windows operating system, ensure that the 
following packages are installed on your system:

- Windows(R) 7/8
- Visual Studio 2012 or later
- An OpenCL SDK, such as APP SDK 2.9
- Latest CMake

### Library for Linux

To develop the clRNG library code on a Linux operating system, ensure that the 
following packages are installed on your system:

- GCC 4.6 and onwards
- An OpenCL SDK, such as APP SDK 2.9
- Latest CMake

  [binary_release]: https://github.com/clMathLibraries/clRNG/releases
  [html_document]: http://clmathlibraries.github.io/clRNG/htmldocs/index.html
  [Doc_name]: http://clmathlibraries.github.io/clRNG/docs/clrng-api.pdf
  [clmath@googlegroups.com]: https://github.com/clMathLibraries/clRNG/wiki  
  [clmath-developers@googlegroups.com]: https://github.com/clMathLibraries/clRNG/wiki/Build 
  [wiki_page]: https://github.com/clMathLibraries/clRNG/wiki]
  [wiki_build]: https://github.com/clMathLibraries/clFFT/wiki/Build


 