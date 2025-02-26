# Integrate image segmenters

Image segmenters predict whether each pixel of an image is associated with a
certain class. This is in contrast to
<a href="../../examples/object_detection/overview">object detection</a>,
which detects objects in rectangular regions, and
<a href="../../examples/image_classification/overview">image
classification</a>, which classifies the overall image. See the
[image segmentation overview](../../examples/segmentation/overview) for more
information about image segmenters.

Use the Task Library `ImageSegmenter` API to deploy your custom image segmenters
or pretrained ones into your mobile apps.

## Key features of the ImageSegmenter API

*   Input image processing, including rotation, resizing, and color space
    conversion.

*   Label map locale.

*   Two output types, category mask and confidence masks.

*   Colored label for display purpose.

## Supported image segmenter models

The following models are guaranteed to be compatible with the `ImageSegmenter`
API.

*   The
    [pretrained image segmentation models on TensorFlow Hub](https://tfhub.dev/tensorflow/collections/lite/task-library/image-segmenter/1).

*   Custom models that meet the
    [model compatibility requirements](#model-compatibility-requirements).

## Run inference in Java

See the
[Image Segmentation reference app](https://github.com/tensorflow/examples/tree/master/lite/examples/image_segmentation/android/)
for an example of how to use `ImageSegmenter` in an Android app.

### Step 1: Import Gradle dependency and other settings

Copy the `.tflite` model file to the assets directory of the Android module
where the model will be run. Specify that the file should not be compressed, and
add the TensorFlow Lite library to the module’s `build.gradle` file:

```java
android {
    // Other settings

    // Specify tflite file should not be compressed for the app apk
    aaptOptions {
        noCompress "tflite"
    }

}

dependencies {
    // Other dependencies

    // Import the Task Vision Library dependency (NNAPI is included)
    implementation 'org.tensorflow:tensorflow-lite-task-vision:0.3.0'
    // Import the GPU delegate plugin Library for GPU inference
    implementation 'org.tensorflow:tensorflow-lite-gpu-delegate-plugin:0.3.0'
}
```

Note: starting from version 4.1 of the Android Gradle plugin, .tflite will be
added to the noCompress list by default and the aaptOptions above is not needed
anymore.

### Step 2: Using the model

```java
// Initialization
ImageSegmenterOptions options =
    ImageSegmenterOptions.builder()
        .setBaseOptions(BaseOptions.builder().useGpu().build())
        .setOutputType(OutputType.CONFIDENCE_MASK)
        .build();
ImageSegmenter imageSegmenter =
    ImageSegmenter.createFromFileAndOptions(context, modelFile, options);

// Run inference
List<Segmentation> results = imageSegmenter.segment(image);
```

See the
[source code and javadoc](https://github.com/tensorflow/tflite-support/blob/master/tensorflow_lite_support/java/src/java/org/tensorflow/lite/task/vision/segmenter/ImageSegmenter.java)
for more options to configure `ImageSegmenter`.

## Run inference in C++

```c++
// Initialization
ImageSegmenterOptions options;
options.mutable_base_options()->mutable_model_file()->set_file_name(model_file);
std::unique_ptr<ImageSegmenter> image_segmenter = ImageSegmenter::CreateFromOptions(options).value();

// Run inference
const SegmentationResult result = image_segmenter->Segment(*frame_buffer).value();
```

See the
[source code](https://github.com/tensorflow/tflite-support/blob/master/tensorflow_lite_support/cc/task/vision/image_segmenter.h)
for more options to configure `ImageSegmenter`.

## Example results

Here is an example of the segmentation results of
[deeplab_v3](https://tfhub.dev/tensorflow/lite-model/deeplabv3/1/metadata/1), a
generic segmentation model available on TensorFlow Hub.

<img src="images/plane.jpg" alt="plane" width="50%">

```
Color Legend:
 (r: 000, g: 000, b: 000):
  index       : 0
  class name  : background
 (r: 128, g: 000, b: 000):
  index       : 1
  class name  : aeroplane

# (omitting multiple lines for conciseness) ...

 (r: 128, g: 192, b: 000):
  index       : 19
  class name  : train
 (r: 000, g: 064, b: 128):
  index       : 20
  class name  : tv
Tip: use a color picker on the output PNG file to inspect the output mask with
this legend.
```

The segmentation category mask should looks like:

<img src="images/segmentation-output.png" alt="segmentation-output" width="30%">

Try out the simple
[CLI demo tool for ImageSegmenter](https://github.com/tensorflow/tflite-support/tree/master/tensorflow_lite_support/examples/task/vision/desktop#image-segmenter)
with your own model and test data.

## Model compatibility requirements

The `ImageSegmenter` API expects a TFLite model with mandatory
[TFLite Model Metadata](../../convert/metadata). See examples of creating
metadata for image segmenters using the
[TensorFlow Lite Metadata Writer API](../../convert/metadata_writer_tutorial.ipynb#image_segmenters).

*   Input image tensor (kTfLiteUInt8/kTfLiteFloat32)

    -   image input of size `[batch x height x width x channels]`.
    -   batch inference is not supported (`batch` is required to be 1).
    -   only RGB inputs are supported (`channels` is required to be 3).
    -   if type is kTfLiteFloat32, NormalizationOptions are required to be
        attached to the metadata for input normalization.

*   Output masks tensor: (kTfLiteUInt8/kTfLiteFloat32)

    -   tensor of size `[batch x mask_height x mask_width x num_classes]`, where
        `batch` is required to be 1, `mask_width` and `mask_height` are the
        dimensions of the segmentation masks produced by the model, and
        `num_classes` is the number of classes supported by the model.
    -   optional (but recommended) label map(s) can be attached as
        AssociatedFile-s with type TENSOR_AXIS_LABELS, containing one label per
        line. The first such AssociatedFile (if any) is used to fill the `label`
        field (named as `class_name` in C++) of the results. The `display_name`
        field is filled from the AssociatedFile (if any) whose locale matches
        the `display_names_locale` field of the `ImageSegmenterOptions` used at
        creation time ("en" by default, i.e. English). If none of these are
        available, only the `index` field of the results will be filled.
