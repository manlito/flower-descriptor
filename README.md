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

Consider kernel:

``` 
     x  k12  x
k = k21 k22 k23
     x  k32  x
```

Keypoint will be marked as maxima at some gaussian pyramid level if the following returns true:

```
bool is_max = 
    (
        k22 == max(abs(k12 - k22)) +
        k22 == max(abs(k32 - k22))
    ) >= 1
    (
        k22 == max(abs(k21 - k22)) +
        k22 == max(abs(k23 - k22))
    ) >= 1
) == 2
```

Or in words: we have a local maxima if response is max in x or y direction.

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
