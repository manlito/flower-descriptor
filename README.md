# Flower Keypoint Detector and Descriptor

A simple, GPU first feature detector and descriptor inspired in DaisyDescriptor. CPU version also provided, but it is mostly provided so you can get the same outputs even without a GPU.

## Motivation

I want to detect descriptors as robustly as SIFT, but more densely in a regular grid. In this sense, instead of first detecting good features, I'll go with first doing descriptor computation densly. Then, I can use descriptor data to actually find solid keypoints.

## Algorithm

1. Daisy descriptor computation part: Use image gradientes to response to, lets say, 8 orientations, as in the original method, and 3 different gaussian pyramid lelvels. Do not normalize yet as that would artificially increase reponses. Gaussian levels provide scale invariance. For video, it is recommed to only use one level.
2. Given some cell size, let's say 10x10 pixels, lets compute our K best descriptors and interpolate positions (more on this bellow)
3. Extract descriptor data using interpolated keypoint position.
4. Transform/Unorient/Normalize descriptor data. Such transforms give us orientation and illuminacion invariance. This is step is optional and not needed if you are matching video, where these effects may be neglegted.

### Max response estimation

Because we will compute the descriptor for the whole image similar to original Daisy descriptor, we will use that (before normalization) to compute the points with the best response to 8 orientations, although encoded using only 4 values. 

Consider kernel:

``` 
    k11 k12 k13
k =  x  k22 k23
     x   x   x
```

One way to compute reponse is computed at pixel `i` with some coordinates is:

```
  r_i = abs(k23 - k22) + abs(k13 - k22) + abs(k12 - k22) + abs (k11 - k22)
```

However, given that convolution is separable, we can also compute 2 gradient images for the image, call them `Gx` and `Gy`, and then:

```
  r_i = abs(Gx_i * cos(  0) + Gy_i * sin(  0)) +
        abs(Gx_i * cos( 45) + Gy_i * sin( 45)) +
        abs(Gx_i * cos( 90) + Gy_i * sin( 90)) + 
        abs(Gx_i * cos(135) + Gy_i * sin(135)) 
```

In both cases, not the use of abs value. This is because we are measure response, so polarity is not important. However, descriptor encoding we prefer actually has polarity.

After we have a response for each pixel position, then we look for a local maxima as

```
bool is_max = max(r_11, r_12 ... r_33) == r_22 == r_i
```
Where `r_11` and up to `r_33` are the responses for the 3 x 3 neighbors of pixel `i`, which is also denotes as `r_22`. We simply check that pixel is max. For this step, we don't care if contiguos points have the same response.

#### Multiscale validation

We will search for local maxima at different Gaussian pyramid levels and perform exactly the same operation. 

### Keypoint position interpolation

We will compute local maxima using a 3x3 kernel. We say some position is maxima when sum of responses at pixel is greater than neighbors. We record all local maximas within cell and optionally sort them, if we only are looking for best K keypoints.

Next, we need to compute subpixel positions. We will do x and y separately. To do in each, we will go a quadratic fit taking y as the sum of responses and x as pixel position. Degenerate cases will be dealt as follows:

- 3 points having exactly same response, take central as the max.
- 2 points having same reponse, just take aveage pixel position
- If quadratic fit returns a position outside range or is NaN, then reject keypoint.

### Descriptor encoding

In order to compare descriptors, for a general case, we need to make them rotation invariant/illumination/scale. We will apply the following:

- Scale. For each of the 3 scale levels, we take the scale with the highest response.
- Orientation. We will search for the strongest angle (of the 8 plus an smoothing kernel). Then, we will apply a shift such that we start with the strongest response.
- Illumination. Each descriptor will be normalized at each coordinate, such that from their 8 orientations orientations will add up 255 (or 1.0 is storing as float).

## Interfaces

```
/**
 * Main image type
 */
template <typename DataType, int width, int height, int NumChannels>
Image {
    *DataType GetData();
    const*const DataType GetData() const;
    DataType data[width][height][channels]
}

/**
 * Provides a way to serially read and open an image. The Goal
 * of tiling is fitting images (and its sideproducts) into device
 * memory, supporting arbitrary image sizes and descriptor sizes
 */
template <typename ImageType>
ImageTiler {
    Open(string path);
    Next(ImageType &image)
}

TilerSettings {
    int tile_size {1024}
}

FlowerDescriptorSettings {
    int levels {3}
    int orientations {8}
}

template <int levels, int orientations>
FlowerKeypointAndDescriptor {
    float x;
    float y;
    descriptor [levels * orientations]
}

FlowerDescriptorExtractor {
    Extract(Image &image, vector<FlowerKeypointAndDescriptor>) {
        ComputeDaisyDescriptor // GPU or CPU
        ExtractInterpolatedKeypoints // GPU or CPU
        EncodeDescriptor // GPU or CPU
        virtual SaveDescriptors // CPU
    }
}

```

## Project Organization

```
+ app
  + DetectKeypointAndDescriptor
  + MatchImages
    - kernels
      + BruteforceMatcher
+ lib
    + cmake
      * place here anything needed to build our lib *
    + flower_descriptor
      + core
        - Image
      + image
        - ImageReader
        - JPEGImageReader
      + descriptor
        + kernels
          - DaisyDescriptorComputation
          - FlowerExtractKeypoints
          - DaisyDescriptorNormalize
          - FlowerDescriptor
        - FlowerDescriptorExtractor
      + tiling
        - ImageTiler
      + interfaces
        - OpenMVGKeypointWriter
        - GenericKeypointReader
        - GenericKeypointWriter
        - GenericDescriptorReader
        - GenericDescriptorWriter
      + visualization
        - KeypointVisualizer
        - DescriptorVisualization
+ tests
  + data
  - test_MODULE_NAME.cpp

* Top level project configuration, CI and other  *

```

### Workplan

- [ ] Boilerplate
- [ ] Write `core` and `image` module, and its `test_image.cpp` file
- [ ] Generic keypoint reader and writer and visualizer for keypoints
- [ ] DaisyDescriptorComputation, writer and visualization
- [ ] FlowerExtractKeypoints
- [ ] Main apps
- [ ] More examples, docker and binary apps
