title: Where are we?
class: big
build_lists: true

CPU frequencies have stalled for a while.

- But transistor density continues to increase accordingly Moore's Law
- CPU: vectorization and multi-core architectures
- GPU: floating-point processing at the cost of flexibility
- The free lunch is over!

---

title: Image Processing is privileged in this new world
build_lists: true

- Lots of parallelism
- Floating-point calculations
- Rigid and predictable memory accesses (most of the times...)

---

title: But...
subtitle: Performance-aware imaging code is a mess
class: big
build_lists: true

- Vectorization: Auto-vectorization, Intrinsics
- Multi-threading: OpenMP, pthreads, etc
- GPU: CUDA, OpenCL, GLSL
- A combination the above for an application that wants to take the best of the machine

---

title: Halide DSL
class: segue dark nobackground

---

title: Domain-Specific Languages for Image Processing
class: big
build_lists: true

- Not a new development
- But previous approaches have support just for pixelwise operations or didn't work in both CPU and GPU
- Impossible to express convolutions, sampling, etc

---

title: Halide
class: big
build_lists: true

- Algorithm specification in a pure functional language that is architecture-agnostic
- Execution configuration (schedule) that tells how this algorithm is going to run and is very dependent on the target architecture

---

title: Specification of a Box-Blur filter

<pre class="prettyprint" data-lang="cpp">
UniformImage input(UInt(16), 3);
Var x,y,c;
Func clamped, blur_x, blur_y;

// Don't access image out of the bounds
clamped(x,y,c) = input(clamp(x, 0,input.width() -1),
                       clamp(y, 0,input.height()-1), c);

// Horizontal Pass
blur_x(x,y,c) = (  clamped(x-1,y,c)
                 + clamped(x  ,y,c)
                 + clamped(x+1,y,c))/3.0f;

// Vertical Pass
blur_y(x,y,c) = (  blur_x(x,y-1,c)
                 + blur_x(x,y  ,c)
                 + blur_x(x,y+1,c))/3.0f;
</pre>

---

title: Schedule

<pre class="prettyprint" data-lang="cpp">
blur_x.root();
blur_y.root();

blur_x.root().parallel(y);
blur_y.root().parallel(y);

blur_y.split(y, y_out, y_in, 8);
blur_y.parallel(y_out);
blur_y.vectorize(x, 8);
blur_x.chunk(y_out, y_in);
blur_x.vectorize(x, 8);
</pre>

---

title: Box-Blur performance
class: image

![Box Blur](images/box-blur.png)

---

title: Case Studies
class: segue dark nobackground

---

title: RGB to CIELAB
class: big
build_lists: true

- CIELAB color space is perceptually uniform
- Pixelwise but very expensive operation (many trancendental functions: exponentiation, cubic root)
- How does Halide perform in a compute-bound task?

---

title: Motion-Blur
class: big
build_lists: true

- Simulate an integration of multiple images in the camera sensors along the moving path of the camera.
- Large convolution
- How does Halide perform in a memory-bound task?

---

title: Motion-Blur
subtitle: Example
class: image

![Motion-Blur](images/mb.png)

---

title: Bilateral Filtering
class: big

> The Bilateral Filter is a non-linear technique that can blur an image while respecting strong edges.
> Its ability to decompose an image into different scales without causing haloes after modification has
> made it ubiquitous in computational photography applications such as tone mapping, style transfer,
> relighting, and denoising.

__Paris et al. Bilateral filtering: Theory and applications__

---

title: Bilateral Filtering
class: big
build_lists: true

- Validation against original Halide paper
- Somewhat complex filter pipeline
- Many choices for implementation in the GPU: local memory and texture memory
- The lack of these features is a performance problem in Halide?

---

title: Bilateral Filtering
subtitle: Example
class: image

![Bilateral-Filter](images/bf.png)

---

title: Results
class: segue dark nobackground

---

title: Performance Results
class: image

![Performance](images/perf.png)

---

title: Analysis
subtitle: CPU
class: big
build_lists: true

- CIELAB: large performance drop in Halide because of the lack of Common Subexpression Elimination (CSE) optimization pass
- Motion-Blur: gcc is unable to auto-vectorize some loops, but Halide is able to do so with an appropriate schedule
- Bilateral Filter: generated code by Halide is almost similar to GCC's

---

title: Analysis
subtitle: GPU
class: big
build_lists: true

- More similar results between OpenCL and Halide in general:
    * Image Processing applications in the GPU are usually bandwidth-limited
    * GPU superior floating-point processing capabilites compared to the CPU
    * ⇒ These two evens the playing field for both implementations in the GPU
- Bilateral Filter: using local and texture memory can make some difference

---

title: Code complexity
subtitle: Number of lines of each implementation
class: big


  ...         |  CPU  |   OpenCL (GPU)  |   Halide (CPU)  |  Halide (GPU)
------------- | ---- -| --------------- | --------------- | --------------
CIELAB        |  52   |  50             |  16 (2)         |  16 (1)
Motion-Blur   |  48   |  54             |  26 (2)         |  26 (3)
Bil. Filter   |  155  |  306            |  36 (6)         |  36 (8)

---

title: Conclusion
class: big
build_lists: true

- Halide can match very optimized implementations in C++ and OpenCL
- It's easier to learn than current traditional alternatives for optimized imaging code
- Still a research effort
- But none of the problems we found are ineherent to the language

---

title: References
class: big

- K. O. W. Group et al., “The opencl specification,” A. Munshi, Ed, 2008.
- I. C. on Illumination, Recommendations on Uniform Color Spaces, Color-difference Equations, Psychometric Color Terms, ser. Publication CIE. Bureau Central de la CIE, 1978.
- S. Paris and F. Durand, “A fast approximation of the bilateral filter using a signal processing approach,” ECCV 2006
- Decoupling Algorithms from Schedules for Easy Optimization of Image Processing Pipelines Jonathan Ragan-Kelley, Andrew Adams, Sylvain Paris, Marc Levoy, Saman Amarasinghe, Frédo Durand. SIGGRAPH 2012
