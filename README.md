# ClearCLIJ - OpenCL based GPU image processing from ImageJ macro

Image processing in modern GPUs allows for accellerating processing speeds massively. This page introduces how to do image processing in the graphics processing unit (GPU) using [OpenCL](https://www.khronos.org/opencl/) from ImageJ macro inside [Fiji](http://fiji.sc). 
It is not necessary to learn OpenCL itself. 
Preprogrammed routines are supposed to do GPU image processing for you with given ImageJ macro programming experience. \
The list of preprogrammed routines might be extended depending on the communities needs.

## Before we start
The presented software is in early developmental stage. Yet it is unclear where this project is heading towards. 
In order to know if it makes sense to develop ClearCLIJ up to a degree where people can _just_ use it, I need your help:

* **Share efforts** I will not ask you to support coding this thing. However, you are welcome if you feel like sharing efforts.

* **Talk about your motiviation** It is very important for me to know who would use ClearCLIJ and for what. 
This is kind of necessary to justify efforts. 
Developing things like that is not my job. But I'm happy doing it; if people use it.
Please drop me a mail (rhaase@mpi-cbg.de), create a [forum post](http://image.sc) or tweet to me [@haesleinhuepf](http://twitter.com/haesleinhuepf) if you find this tool useful. 
Tell me your story. What do you do with it? How do you like it?

* **Guide development** Think about functionality you would like to have. [Formulate a github issue](https://github.com/ClearControl/clearclij/issues) or again, drop me as message.

* **Communication is key** I do work a lot with ImageJ/Fiji and I do work a lot with OpenCL. Building the bridge between both might be a low hanging apple for me. 
But again, I need to know other people views and opinions in order to make this project successful. 

Thanks for reading. Now, let's get started.


## Installation
Download and install [Fiji from its website](https://fiji.sc/Downloads).
Add the update site `http://sites.imagej.net/clij` to your Fiji installation. [Read more about how to activate update sites]( https://imagej.net/Following_an_update_site).
Restart Fiji. ClearCLIJ is successfully installed, if you find a menu entry _Plugins > CLIJ_.

**Please note:** This is experimental software, it may have side effects on your Fiji installation and may break other plugins. 
It is strongly recommended to not install ClearCLIJ in a production environment. 
Use a freshly downloaded Fiji installation for testing experimental plugins like this. 
[Read the BSD license file](http://github.com/haesleinhuepf/ClearCLIJ/license.txt) for more details.

## A first macro
The first step is selecting the OpenCL device / GPU you would like to use to process your images. 
Start the macro recorder to record your choice by clicking Fijis menu _Plugins > Macro > Record_. 
Afterwards, click the menu _Plugins > CLIJ > CLIJ Macro Extensions_.
The selection dialog will open. Make your choice and click ok:

![Image](images/device_dialog.png)

Ignore the error message which comes up. Have a look in the macro recorder and click it's _Create_ button.

![Image](images/macro_recorder.png)

The first macro is [help.ijm](https://github.com/haesleinhuepf/clearclij/blob/master/src/main/macro/help.ijm).
It will assist us to get an overview which methods are supported by ClearCLIJ to process images. 
It looks like this:

```java
run("CLIJ Macro Extensions", "cl_device=[Intel(R) UHD Graphics 620]");
Ext.CLIJ_help("add");
```

By executing it, you will find a list of commands containing the term `add` in their names:

```java
Found 3 method(s) containing the pattern "add":
Ext.CLIJ_addPixelwise(Image summand1, Image summand2, Image destination);
Ext.CLIJ_addScalar(Image source, Image destination, Number scalar);
Ext.CLIJ_addWeightedPixelwise(Image summand1, Image summand2, Image destination, Number factor1, Number factor2);
```

Keep this example program, you might need it later again if you want to search for help on ClearCLIJ methods. 
It also tells you which parameters the methods need in order to run.

## Transferring images between ImageJ and the GPU
In order to allow images to be processed by the GPU, you need to transfer them into the memory of the GPU. 
In order to view images which were processed by the GPU, you need to transfer them back to ImageJ. 
The two methods for doing this are called `push(image)` and `pull(image)`. 
You can remove a single image from the GPUs memory by using the `release(image)` method. 
Finally, you can remove all images from the GPU with the `clear()` method.
Importantly, you cannot create images in the GPU _yet_.
In order to process an image _A_ to an image _B_, both images _A_ and _B_ need to be pushed to the GPU. 
Only then you can run methods to fill the image _B_ with values resulting from _A_. 
Finally, after processing, you pull _B_ back from the GPU to ImageJ in order to show it.

Let's have a look at an example which loads an image and blurs it using the push-pull mechanism.

```java
// Get test data
run("T1 Head (2.4M, 16-bits)");
input = getTitle();
getDimensions(width, height, channels, slices, frames);

// create an emtpy image to put the blurred pixels in
newImage("Untitled", "16-bit black", width, height, slices);
rename("Blurred");
blurred = getTitle();

// Init GPU
run("CLIJ Macro Extensions", "cl_device=[Intel(R) UHD Graphics 620]");
Ext.CLIJ_clear();

// push images to GPU
Ext.CLIJ_push(input);
Ext.CLIJ_push(blurred);

// cleanup ImageJ
run("Close All");

// Blur in GPU
Ext.CLIJ_blur3d(input, blurred, 20, 20, 1, 10, 10, 1);

// Get results back from GPU
Ext.CLIJ_pull(blurred);

// Cleanup by the end
Ext.CLIJ_clear();
```

## Sparing time with GPU based image processing
The overall goal for processing images in the GPU is sparing time. 
GPUs can process images faster because they can calculate pixel values of many pixels in parallel. 
Furthermore, images in memory of modern GPUs can be accessed faster than in ImageJ. 
However, there is a drawback: pushing/pulling the images to/from the GPU takes time. 
Thus, overall efficiency can only be achieved if whole pipelines are processed in the GPU. 
Furthermore, repeatedly using the same operations on a GPU pays off because operations are cached. Reusing them is faster than using other methods. 

Let's compare the `Mean 3D` filter of ImageJ with it's counterpart in ClearCLIJ.
The example macro is [benchmarking.ijm](https://github.com/haesleinhuepf/clearclij/blob/master/src/main/macro/benchmarking.ijm). 
It executes both operations ten times and measures the time each operation takes. 
This is just an excerpt of the macro:

```java
// Local mean filter in CPU
for (i = 1; i <= 10; i++) {
	time = getTime();
	run("Mean 3D...", "x=3 y=3 z=3");
	print("CPU mean filter no " + i + " took " + (getTime() - time));
}
```

```java
// Local mean filter in GPU
for (i = 1; i <= 10; i++) {
	time = getTime();
	Ext.CLIJ_mean3d(input, blurred, 3, 3, 3);
	print("GPU mean filter no " + i + " took " + (getTime() - time));
}
```

When executing the macro on an Intel HD card, the output is:

```java
CPU mean filter no 1 took 3292
CPU mean filter no 2 took 3062
CPU mean filter no 3 took 3076
CPU mean filter no 4 took 3129
CPU mean filter no 5 took 3074
CPU mean filter no 6 took 3077
CPU mean filter no 7 took 4001
CPU mean filter no 8 took 4305
CPU mean filter no 9 took 4217
CPU mean filter no 10 took 4431
GPU mean filter no 1 took 585
GPU mean filter no 2 took 26
GPU mean filter no 3 took 26
GPU mean filter no 4 took 22
GPU mean filter no 5 took 21
GPU mean filter no 6 took 21
GPU mean filter no 7 took 20
GPU mean filter no 8 took 27
GPU mean filter no 9 took 24
GPU mean filter no 10 took 23
```

**Heureka, we can spare 98% of the time by executing the operation on the GPU!**
Side note: ImageJs mean filter runs inplace. That means the result is stored in the same memory as the input image. 
Thus, the with every iteration in the for loop, the image becomes more and more blurry. 
The OpenCL operation in the GPU always starts from the _input_ image and puts its result in the _blurred_ image. 
Thus, the resulting images will look different. 
Be a sceptical scietist when processing images in the GPU. Check that the workflow is indeed doing the right thing. This is especially important when working with experimental software.

When executing the same code on an AMD Ryzen 3 CPU / Vega 3 GPU:


These are just rough benchmarks. 
When ClearCLIJ matures, I might do a more detailed benchmarking of several methods. 
This example here should just motivate you to test your workflow on a GPU and guide you how to evaluate its performance.

Again, please let me know what you think about ClearCLIJ, create github issues to guide its further development and [visit the project page]() to stay up-to-date.

Happy coding!

Cheers,

Robert _@haesleinhuepf_ Haase
