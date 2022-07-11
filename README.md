## gpujsutils

![gpujsutils-status](https://img.shields.io/npm/v/gpujsutils.svg) 
![gpujsutils-downloads](https://img.shields.io/npm/dt/gpujsutils.svg)
![gpujsutils-l](https://img.shields.io/npm/l/gpujsutils)

[gpu.js](https://github.com/gpujs) is an amazing library for writing gpu kernels in plain js. This wrapper makes life easier to use it with a stable gpu.js bundle and adds some flexible macros. This revolve around persistent kernels with resizable i/o on-the-fly so we can make the best use of the performance benefits of parallelization. This is even better with web workers (see our MagicWorker library)

See GPU.js docs for information on how to write kernels

```
npm i gpujsutils
```
#### Create a GPU util instance, + general utility functions
```js
import {gpuUtils, makeKrnl, makeCanvasKrnl} from 'gpuUtils'

const gpuutils = new gpuUtils();
```

#### Add a GPU utilty function which can be called in kernels (see gpujs docs for more)
```js

function rms(arr, mean, len) { //root mean square error
    var est = 0;
    var vari = 0;
    for (var i = 0; i < len; i++) {
        vari = arr[i]-mean;
        est += vari*vari;
    }
    return Math.sqrt(est/len);
}

gpuutils.addFunction(rms);

```

#### Create a GPU kernel with default settings
```js

function transpose2DKern(mat2) { //Transpose a 2D matrix, meant to be combined
    return mat2[this.thread.y][this.thread.x];
}

let kernel = makeKrnl(
    gpuutils.gpu, //the actual GPUjs instance, used normally  
    transpose2DKern, //the kernel
{ //kernel settings, meant to use all of the flexibility features by default (e.g. dynamic sizing)
  setDynamicOutput: true,
  setDynamicArguments: true,
  setPipeline: true,
  setImmutable: true,
  setGraphical: false
}); //makeKrnl(gpuutils.gpu);

let mat2 = [[1,2,3,4],[5,6,7,8],[8,9,10,11],[12,13,14,15]];

let result = kernel(mat2)

//OR to add to the gpu utilities

kernel = gpuutils.addKernel('transpose2D', transpose2DKern);
result = gpuutils.callKernel('transpose2D',mat2);


```

#### Create a kernel that renders to a canvas
```js

//From a gpujs observable: https://observablehq.com/@robertleeplummerjr/video-convolution-using-gpu-js
function ImgConv2DKern(img, width, height, kernel, kernelLength) {
    let kernelRadius = (Math.sqrt(kernelLength) - 1) / 2;
    const kSize = 2 * kernelRadius + 1;
    let r = 0, g = 0, b = 0;

    let i = -kernelRadius;
    let kernelOffset = 0;
    while (i <= kernelRadius) {
        if (this.thread.x + i < 0 || this.thread.x + i >= width) {
            i++;
            continue;
        }

        let j = -kernelRadius;
        while (j <= kernelRadius) {
            if (this.thread.y + j < 0 || this.thread.y + j >= height) {
                j++;
                continue;
            }

            kernelOffset = (j + kernelRadius) * kSize + i + kernelRadius;
            const weights = kernel[kernelOffset];
            const pixel = img[this.thread.y + i][this.thread.x + j];
            r += pixel.r * weights;
            g += pixel.g * weights;
            b += pixel.b * weights;
            j++;
        }
        i++;
    }

    this.color(r, g, b);
}


let kernel = makeCanvasKernel( 
    gpu,
    ImgConv2DKern,
    {
        output: [300,300],
        setDynamicArguments: true,
        setDynamicOutput: true,
        setPipeline: false,
        setImmutable: true,
        setGraphical: true
    },
    divId //id of the div to append a new canvas to. Leave blank to append to body
);

kernel(video); //input an image or video from a source

//OR to add to the gpu utilities

kernel = gpuutils.addCanvasKernel('imgConv', ImgConv2DKern);
gpuutils.callCanvasKernel('imgConv',video, [video.width,video.height]);



```


### Combine Kernels
```js

//adapted from gpujs tutorial
const add = gpuutils.addKernel('add',function(a, b) {
  return a[this.thread.x] + b[this.thread.x];
}).setOutput([20]);

const multiply = gpuutils.addKernel('multiply',function(a, b) {
  return a[this.thread.x] * b[this.thread.x];
}).setOutput([20]);

//multi-step operations
const superKernel = gpuutils.combineKernels(
    'superKernel', //name the combined kernel
    ['add','multiply'], 
    function(a, b, c) {
        return multiply(add(a, b), c);
    }
);

let result = gpuutils.callKernel('superKernel',[3,4,5]);

```

### Default Macros
```js

//pass signal buffer to receive an ordered amplitude spectrum based 0.5x the size of sample rate (nyquist frequency)
gpuutils.gpuDFT(
    signalBuffer, //the signal buffer
    nSeconds, //number of seconds of data. sample rate = length/nSeconds 
    scalar, //can apply a scalar multiplier to the amplitudes
    texOut //receive a texture map out instead of arrays?
);

//or
gpuutils.gpuFFT(...); //faster


// pass a 1D array of evenly-sized signal buffers to receive a list of ordered amplitude spectrums 0.5x the size of sample rate (nyquist frequency)
gpuutils.MultiChannelDFT(
    signalBuffer,  //1D list of evenly sized signal buffers
    nSeconds, //number of seconds of data. sample rate = length/nSeconds 
    scalar, //can apply a scalar multiplier to the amplitudes
    false //receive a texture map out instead of arrays?
);

//or 
gpuutils.MultiChannelFFT(...); //faster

// pass a 1D array of evenly-sized signal buffers to receive a list of ordered amplitude spectrums 0.5x the size of sample rate (nyquist frequency), between the two frequencies. Better with more seconds or higher samplerate
gpuutils.MultiChannelDFT_Bandpass(
    signalBuffer,  //1D list of evenly sized signal buffers
    nSeconds, //number of seconds of data. sample rate = length/nSeconds 
    freqStart, //start frequency e.g. 3Hz
    freqEnd, //end Frequency e.g. 100Hz
     scalar, //can apply a scalar multiplier to the amplitudes
    false //receive a texture map out instead of arrays?
);

//or 
gpuutils.MultiChannelFFT_BandPass(...);

```


### Default Kernels
```js

    gpuutils.dft //discrete fourier transform
    gpuutils.idft //inverse DFT (untested :P)
    gpuutils.dft_windowed //DFT between two frequencies, need sufficient sample rate or number of seconds of data
    gpuutils.idft_windowed //inverse dft to reverse the bandpassed dft (untested)
    gpuutils.fft //fast fourier transform, simply decimates the number of samples used to compute lower frequency amplitudes
    gpuutils.ifft //inverse fft (untested)
    gpuutils.fft_windowed //bandpassed fft
    gpuutils.ifft_windowed  //inverse bandpassed fft (untested)
    gpuutils.listdft2D  //pass an array of arrays in of the same length to return a list of DFTs (broken) 
    gpuutils.listdft1D   //pass a single array of evenly sized sample sets of data to return the DFTs. 
    gpuutils.listdft1D_windowed  //etc
    gpuutils.listfft1D             //etc
    gpuutils.listfft1D_windowed     //tc
    gpuutils.listidft1D_windowed  //untested
    gpuutils.listifft1D_windowed  //untested
    gpuutils.bulkArrayMul 
    gpuutils.correlograms; //cross correlations, untested
    gpuutils.correlogramsPC; //precomputed means and estimators

```

There's a few other things used internally but they're not that useful.




Joshua Brewster & Dovydas Stirpeika
License: AGPL v3.0