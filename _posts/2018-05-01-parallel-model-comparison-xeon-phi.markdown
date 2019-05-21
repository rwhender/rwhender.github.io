---
layout: post
title: Parallelized model comparison and the Intel Xeon Phi
author: Wesley Henderson
date: 2018-05-01 23:30:00 -0500
categories: dissertation modelcomparison xeonphi
---

---

*This post is part of a series detailing my efforts to develop and implement Bayesian model comparison approaches on several parallel computing platforms. I’ll be giving a tutorial on this topic in July at [MaxEnt 2018](https://max-ent.github.io/) in London.*

---

Over the past month or so, I have been working to get a nested sampling implementation, along with a simple example, working on an Intel Xeon Phi. We have a fairly old model, the [5110P](https://ark.intel.com/products/71992/Intel-Xeon-Phi-Coprocessor-5110P-8GB-1_053-GHz-60-core), a 60-core model for which Intel just recently dropped support. While the timing is somewhat unfortunate, the principles required to run code using the Phi should be applicable to newer models as well.

This version of the Phi is a PCI-Express expansion card that basically runs a separate computer within your workstation. The Phi is intended to be used in a similar way to a GPU for general purpose computation. That is, the developer offloads some intensive, ideally data parallel task to the hardware, and it drastically increases your computation speed. Contrary to a GPU, the Phi has the advantage of being composed of many general purpose CPUs. This allows developers to treat the platform simply as a many-core CPU, and code can be written without having to keep any special architecture limitations in mind, as with GPGPU development.

Hardware and operating system support for the Phi is limited. I did a fair amount of research, and ended up selecting a Dell workstation that we then bought specifically for the purpose of using the Phi. I originally used Windows 10, but support for Windows is spotty at best, so I’ve recently switched to CentOS 7. It’s not officially supported, but Red Hat Enterprise Linux (RHEL) is, and CentOS seems to work just as well so far.

### The Offload model

The specific nested sampling approach I’ve been trying to implement recently is the combined-chain approach described in a paper I co-authored that was published last year ([published version here](https://doi.org/10.1016/j.dsp.2017.07.021), [preprint here]({{'/pdfs/henderson goggans cao parallel nested sampling elsevier dsp.pdf' | prepend: site.baseurl }})). The gist of this approach is that several serial nested sampling runs are conducted, and the results are combined and processed to get a final result. This approach yields increased precision in the log-evidence estimate when compared to each serial result on its own.

This particular approach requires that each separate nested sampling process return its list of discarded samples. Depending on the language, I usually represent a sample as an object or structure that contains parameter coordinates (a 1D array of doubles), a log-likelihood value (a scalar double), and a log-weight (also a scalar double). Once all the separate processes are finished, the lists of samples are combined together, sorted by likelihood, and the log-evidence is computed.

The offload model so far seems like the most logical way to implement this on the Phi. In this model, all of your code is run on the host CPU except for parts which are specifically marked for offloading using (for C/C++),


    #pragma offload target(mic)
    {
	// Code to offload goes here   
    }


While this model nominally supports C++ code, only very simple data structures can be passed back and forth between the host and the Phi. Neither `std::vector` nor dynamically allocated 2D arrays are supported for offloading. The amount of data I need to extract from each of these nested sampling processes is potentially very large, so the arrays must be dynamically allocated to avoid stack overflows. The prohibition on dynamically allocated 2D arrays means that the results must be stored in a single 1D array.

Upon discovering this limitation, I thought I might still be able to get away with using the offload model. I would simply pass to the Phi a 1D array for storing all of the results from each process, and use 2D-to-1D index computation to store the results in a sensible way that can be recovered later. This approach does in fact work; however, the performance is horrendous. I’m not 100% sure what’s happening at this point, but my working theory is that because each process is trying to store its results in the same array, and only one process can access the array at a time to avoid corruption, we end up with what amounts to serial computation speeds. In fact, the runtime is probably actually worse than the purely serial runtime in this scenario due to the overhead involved in passing priority back and forth among all the running processes for accessing the results array.

It might be possible to mitigate this issue somewhat (or maybe almost completely) by using an array local to the process to store results until the process is finished. This way, the processes will do all their heavy lifting in parallel, and only start blocking each other for the amount of time it takes to copy their results into the final array. I’ll try this and report back later.

The other way around this problem would be to drop the offload programming model and try the shared memory approach. This approach creates a virtual shared memory space that both the host and the Phi can access, and allows use of more complex data structures, including some of those included in the C++ STL. Using this approach would require some pretty heavy retooling of my code, so I think I’ll only resort to this if the previous approach doesn’t work.

### Intel VTune Amplifier

The issue that finally got me to migrate from Windows to CentOS was support for Intel’s VTune Amplifier software profiling package. This is apparently the only tool that can be used to profile code running on the Phi, and I needed it to find the slow-down mentioned in the previous section. After trying everything I could think to try (and to google), I just could not get Amplifier to communicate with the Phi on Windows. After moving everything over to CentOS, Amplifier works just fine, and I was able to more easily diagnose the performance issue in my code. Perhaps others have had more luck getting Amplifier to run on Windows with the Phi; if so, I’d love to hear about it!

That wraps it up for today’s installment. More soon!
