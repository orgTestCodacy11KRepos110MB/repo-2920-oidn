Open Image Denoise API
======================

Open Image Denoise provides a C99 API (also compatible with C++) and a
C++11 wrapper API as well. For simplicity, this document mostly refers to the
C99 version of the API.

The API is designed in an object-oriented manner, e.g. it contains device
objects (`OIDNDevice` type), buffer objects (`OIDNBuffer` type), and filter
objects (`OIDNFilter` type). All objects are reference-counted, and handles
can be released by calling the appropriate release function (e.g.
`oidnReleaseDevice`) or retained by incrementing the reference count (e.g.
`oidnRetainDevice`).

An important aspect of objects is that setting their parameters do not have
an immediate effect (with a few exceptions). Instead, objects with updated
parameters are in an unusable state until the parameters get explicitly
committed to a given object. The commit semantic allows for batching up
multiple small changes, and specifies exactly when changes to objects will
occur.

All API calls are thread-safe, but operations that use the same device will be
serialized, so the amount of API calls from different threads should be minimized.

Examples
--------

To have a quick overview of the C99 and C++11 APIs, see the following
simple example code snippets.

### Basic denoising (C99 API)

    #include <OpenImageDenoise/oidn.h>
    ...
    // Create an Intel Open Image Denoise device
    OIDNDevice device = oidnNewDevice(OIDN_DEVICE_TYPE_DEFAULT);
    oidnCommitDevice(device);

    // Create a filter for denoising a beauty (color) image using optional auxiliary images too
    OIDNFilter filter = oidnNewFilter(device, "RT"); // generic ray tracing filter
    oidnSetSharedFilterImage(filter, "color",  colorPtr,
                             OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0); // beauty
    oidnSetSharedFilterImage(filter, "albedo", albedoPtr,
                             OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0); // auxiliary
    oidnSetSharedFilterImage(filter, "normal", normalPtr,
                             OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0); // auxiliary
    oidnSetSharedFilterImage(filter, "output", outputPtr,
                             OIDN_FORMAT_FLOAT3, width, height, 0, 0, 0); // denoised beauty
    oidnSetFilter1b(filter, "hdr", true); // beauty image is HDR
    oidnCommitFilter(filter);

    // Filter the image
    oidnExecuteFilter(filter);

    // Check for errors
    const char* errorMessage;
    if (oidnGetDeviceError(device, &errorMessage) != OIDN_ERROR_NONE)
      printf("Error: %s\n", errorMessage);

    // Cleanup
    oidnReleaseFilter(filter);
    oidnReleaseDevice(device);

### Basic denoising (C++11 API)

    #include <OpenImageDenoise/oidn.hpp>
    ...
    // Create an Intel Open Image Denoise device
    oidn::DeviceRef device = oidn::newDevice();
    device.commit();

    // Create a filter for denoising a beauty (color) image using optional auxiliary images too
    oidn::FilterRef filter = device.newFilter("RT"); // generic ray tracing filter
    filter.setImage("color",  colorPtr,  oidn::Format::Float3, width, height); // beauty
    filter.setImage("albedo", albedoPtr, oidn::Format::Float3, width, height); // auxiliary
    filter.setImage("normal", normalPtr, oidn::Format::Float3, width, height); // auxiliary
    filter.setImage("output", outputPtr, oidn::Format::Float3, width, height); // denoised beauty
    filter.set("hdr", true); // beauty image is HDR
    filter.commit();

    // Filter the image
    filter.execute();

    // Check for errors
    const char* errorMessage;
    if (device.getError(errorMessage) != oidn::Error::None)
      std::cout << "Error: " << errorMessage << std::endl;

### Denoising with prefiltering (C++11 API)

    // Create a filter for denoising a beauty (color) image using prefiltered auxiliary images too
    oidn::FilterRef filter = device.newFilter("RT"); // generic ray tracing filter
    filter.setImage("color",  colorPtr,  oidn::Format::Float3, width, height); // beauty
    filter.setImage("albedo", albedoPtr, oidn::Format::Float3, width, height); // auxiliary
    filter.setImage("normal", normalPtr, oidn::Format::Float3, width, height); // auxiliary
    filter.setImage("output", outputPtr, oidn::Format::Float3, width, height); // denoised beauty
    filter.set("hdr", true); // beauty image is HDR
    filter.set("cleanAux", true); // auxiliary images will be prefiltered
    filter.commit();

    // Create a separate filter for denoising an auxiliary albedo image (in-place)
    oidn::FilterRef albedoFilter = device.newFilter("RT"); // same filter type as for beauty
    albedoFilter.setImage("albedo", albedoPtr, oidn::Format::Float3, width, height);
    albedoFilter.setImage("output", albedoPtr, oidn::Format::Float3, width, height);
    albedoFilter.commit();

    // Create a separate filter for denoising an auxiliary normal image (in-place)
    oidn::FilterRef normalFilter = device.newFilter("RT"); // same filter type as for beauty
    normalFilter.setImage("normal", normalPtr, oidn::Format::Float3, width, height);
    normalFilter.setImage("output", normalPtr, oidn::Format::Float3, width, height);
    normalFilter.commit();

    // Prefilter the auxiliary images
    albedoFilter.execute();
    normalFilter.execute();

    // Filter the beauty image
    filter.execute();


Device
------

Intel Open Image Denoise supports a device concept, which allows different
components of the application to use the Open Image Denoise API without
interfering with each other. An application first needs to create a device with

    OIDNDevice oidnNewDevice(OIDNDeviceType type);

where the `type` enumeration maps to a specific device implementation, which
can be one of the following:

Name                       Description
-------------------------- -----------------------------------------------------
`OIDN_DEVICE_TYPE_DEFAULT` select the approximately fastest device
`OIDN_DEVICE_TYPE_CPU`     CPU device (requires SSE4.1 support or Apple Silicon)
-------------------------- -----------------------------------------------------
: Supported device types, i.e., valid constants of type `OIDNDeviceType`.

Once a device is created, you can call

    void oidnSetDevice1b(OIDNDevice device, const char* name, bool value);
    void oidnSetDevice1i(OIDNDevice device, const char* name, int  value);
    bool oidnGetDevice1b(OIDNDevice device, const char* name);
    int  oidnGetDevice1i(OIDNDevice device, const char* name);

to set and get parameter values on the device. Note that some parameters are
constants, thus trying to set them is an error. See the tables below for the
parameters supported by devices.

----------- -------------- -------- --------------------------------------------
Type        Name            Default Description
----------- -------------- -------- --------------------------------------------
`const int` `version`               combined version number (major.minor.patch)
                                    with two decimal digits per component

`const int` `versionMajor`          major version number

`const int` `versionMinor`          minor version number

`const int` `versionPatch`          patch version number

`int`       `verbose`               0 verbosity level of the console output
                                    between 0--4; when set to 0, no output is
                                    printed, when set to a higher level more
                                    output is printed
----------- -------------- -------- --------------------------------------------
: Parameters supported by all devices.

------ -------------- -------- -------------------------------------------------
Type   Name            Default Description
------ -------------- -------- -------------------------------------------------
`int`  `numThreads`          0 maximum number of threads which the library
                               should use; 0 will set it automatically to get
                               the best performance

`bool` `setAffinity`      true enables thread affinitization (pinning software
                               threads to hardware threads) if it is necessary
                               for achieving optimal performance
------ -------------- -------- -------------------------------------------------
: Additional parameters supported only by CPU devices.

Note that the CPU device heavily relies on setting the thread affinities to
achieve optimal performance, so it is highly recommended to leave this option
enabled. However, this may interfere with the application if that also sets
the thread affinities, potentially causing performance degradation. In such
cases, the recommended solution is to either disable setting the affinities
in the application or in Intel Open Image Denoise, or to always set/reset
the affinities before/after each parallel region in the application (e.g.,
if using TBB, with `tbb::task_arena` and `tbb::task_scheduler_observer`).

Once parameters are set on the created device, the device must be committed with

    void oidnCommitDevice(OIDNDevice device);

This device can then be used to construct further objects, such as buffers and
filters. Note that a device can be committed only once during its lifetime.
Before the application exits, it should release all devices by invoking

    void oidnReleaseDevice(OIDNDevice device);

Note that Intel Open Image Denoise uses reference counting for all object types,
so this function decreases the reference count of the device, and if the count
reaches 0 the device will automatically get deleted. It is also possible to
increase the reference count by calling

    void oidnRetainDevice(OIDNDevice device);

An application typically creates only a single device. If required differently,
it should only use a small number of devices at any given time.

### Error Handling

Each user thread has its own error code per device. If an error occurs when
calling an API function, this error code is set to the occurred error if it
stores no previous error. The currently stored error can be queried by the
application via

    OIDNError oidnGetDeviceError(OIDNDevice device, const char** outMessage);

where `outMessage` can be a pointer to a C string which will be set to a more
descriptive error message, or it can be `NULL`. This function also clears the
error code, which assures that the returned error code is always the first
error occurred since the last invocation of `oidnGetDeviceError` on the current
thread. Note that the optionally returned error message string is valid only
until the next invocation of the function.

Alternatively, the application can also register a callback function of type

    typedef void (*OIDNErrorFunction)(void* userPtr, OIDNError code, const char* message);

via

    void oidnSetDeviceErrorFunction(OIDNDevice device, OIDNErrorFunction func, void* userPtr);

to get notified when errors occur. Only a single callback function can be
registered per device, and further invocations overwrite the previously set
callback function, which do *not* require also calling the `oidnCommitDevice`
function. Passing `NULL` as function pointer disables the registered callback
function. When the registered callback function is invoked, it gets passed the
user-defined payload (`userPtr` argument as specified at registration time),
the error code (`code` argument) of the occurred error, as well as a string
(`message` argument) that further describes the error. The error code is always
set even if an error callback function is registered. It is recommended to
always set a error callback function, to detect all errors.

When the device construction fails, `oidnNewDevice` returns `NULL` as device.
To detect the error code of a such failed device construction, pass `NULL` as
device to the `oidnGetDeviceError` function. For all other invocations of
`oidnGetDeviceError`, a proper device handle must be specified.

The following errors are currently used by Intel Open Image Denoise:

--------------------------------- ----------------------------------------------
Name                              Description
--------------------------------- ----------------------------------------------
`OIDN_ERROR_NONE`                 no error occurred

`OIDN_ERROR_UNKNOWN`              an unknown error occurred

`OIDN_ERROR_INVALID_ARGUMENT`     an invalid argument was specified

`OIDN_ERROR_INVALID_OPERATION`    the operation is not allowed

`OIDN_ERROR_OUT_OF_MEMORY`        not enough memory to execute the operation

`OIDN_ERROR_UNSUPPORTED_HARDWARE` the hardware (e.g., CPU) is not supported

`OIDN_ERROR_CANCELLED`            the operation was cancelled by the user
--------------------------------- ----------------------------------------------
: Possible error codes, i.e., valid constants of type `OIDNError`.


Buffer
------

Large data like images can be passed to Intel Open Image Denoise either via
pointers to memory allocated and managed by the user (this is the recommended,
often easier and more efficient approach, if supported by the device) or by
creating buffer objects (supported by all devices). To create a new data buffer
with memory allocated and owned by the device, holding `byteSize` number of
bytes, use

    OIDNBuffer oidnNewBuffer(OIDNDevice device, size_t byteSize);

The created buffer is bound to the specified device (`device` argument). The
specified number of bytes are allocated at buffer construction time and
deallocated when the buffer is destroyed.

It is also possible to create a "shared" data buffer with memory allocated and
managed by the user with

    OIDNBuffer oidnNewSharedBuffer(OIDNDevice device, void* ptr, size_t byteSize);

where `ptr` points to the user-managed memory and `byteSize` is its size in
bytes. At buffer construction time no buffer data is allocated, but the buffer
data provided by the user is used. The buffer data must remain valid for as
long as the buffer may be used, and the user is responsible to free the buffer
data when no longer required.

Similar to device objects, buffer objects are also reference-counted and can be
retained and released by calling the following functions:

    void oidnRetainBuffer(OIDNBuffer buffer);
    void oidnReleaseBuffer(OIDNBuffer buffer);

The size of the buffer in bytes can be queried using

    size_t oidnGetBufferSize(OIDNBuffer buffer);

Accessing the data stored in a buffer object is possible by mapping it into the
address space of the application using

    void* oidnMapBuffer(OIDNBuffer buffer, OIDNAccess access, size_t byteOffset, size_t byteSize)

where `access` is the desired access mode of the mapped memory, `byteOffset` is
the offset to the beginning of the mapped memory region in bytes, and
`byteSize` is the number of bytes to map. The function returns a pointer to
the mapped buffer data. If the specified `byteSize` is 0, the maximum
available amount of memory will be mapped. The `access` argument must be one of
the access modes in the following table:

Name                        Description
--------------------------- -------------------------------------------------------------
`OIDN_ACCESS_READ`          read-only access
`OIDN_ACCESS_WRITE`         write-only access
`OIDN_ACCESS_READ_WRITE`    read and write access
`OIDN_ACCESS_WRITE_DISCARD` write-only access but the previous contents will be discarded
--------------------------- -------------------------------------------------------------
: Access modes for memory regions mapped with `oidnMapBuffer`, i.e., valid
  constants of type `OIDNAccess`.

After accessing the mapped data in the buffer, the memory region must be
unmapped with

    void oidnUnmapBuffer(OIDNBuffer buffer, void* mappedPtr);

where `mappedPtr` must be a pointer returned by a call to `oidnMapBuffer` for
the specified buffer. Any change to the mapped data is guaranteed to take
effect only after unmapping the memory region.

It is also possible to get a pointer directly to the buffer data but this might be valid
only on the device the buffer was created on:

    void* oidnGetBufferData(OIDNBuffer buffer);

### Data Format

Buffers store opaque data and thus have no information about the type and
format of the data. Other objects, e.g. filters, typically require specifying
the format of the data stored in buffers or shared via pointers. This can be
done using the `OIDNFormat` enumeration type:

Name                     Description
------------------------ -------------------------------------------------------
`OIDN_FORMAT_UNDEFINED`  undefined format
`OIDN_FORMAT_FLOAT`      32-bit floating-point scalar
`OIDN_FORMAT_FLOAT[234]` 32-bit floating-point [234]-element vector
`OIDN_FORMAT_HALF`       16-bit floating-point scalar
`OIDN_FORMAT_HALF[234]`  16-bit floating-point [234]-element vector
------------------------ -------------------------------------------------------
: Supported data formats, i.e., valid constants of type `OIDNFormat`.


Filter
------

Filters are the main objects in Intel Open Image Denoise that are responsible
for the actual denoising. The library ships with a collection of filters which
are optimized for different types of images and use cases. To create a filter
object, call

    OIDNFilter oidnNewFilter(OIDNDevice device, const char* type);

where `type` is the name of the filter type to create. The supported filter
types are documented later in this section. Once created, filter objects can be
retained and released with

    void oidnRetainFilter(OIDNFilter filter);
    void oidnReleaseFilter(OIDNFilter filter);

After creating a filter, it needs to be set up by specifying the input and
output images, and potentially setting other parameter values as well.

To bind images to the filter, you can use one of the following functions:

    void oidnSetFilterImage(OIDNFilter filter, const char* name,
                            OIDNBuffer buffer, OIDNFormat format,
                            size_t width, size_t height,
                            size_t byteOffset,
                            size_t bytePixelStride, size_t byteRowStride);

    void oidnSetSharedFilterImage(OIDNFilter filter, const char* name,
                                  void* ptr, OIDNFormat format,
                                  size_t width, size_t height,
                                  size_t byteOffset,
                                  size_t bytePixelStride, size_t byteRowStride);

It is possible to specify either a data buffer object (`buffer` argument) with
the `oidnSetFilterImage` function, or directly a pointer to shared user-managed
data (`ptr` argument) with the `oidnSetSharedFilterImage` function.

In both cases, you must also specify the name of the image parameter to set
(`name` argument, e.g. `"color"`, `"output"`), the pixel format (`format`
argument), the width and height of the image in number of pixels (`width` and
`height` arguments), the starting offset of the image data (`byteOffset`
argument), the pixel stride (`bytePixelStride` argument) and the row stride
(`byteRowStride` argument), in number of bytes.

The row stride must be an integer multiple of the pixel stride. If the pixels
and/or rows are stored contiguously (tightly packed without any gaps), you can
set `bytePixelStride` and/or `byteRowStride` to 0 to let the library compute
the actual strides automatically, as a convenience.

Images support only the `OIDN_FORMAT_FLOAT3` and `OIDN_FORMAT_HALF3` pixel
formats. Custom image layouts with extra channels (e.g. alpha channel) or other
data are supported as well by specifying a non-zero pixel stride. This way,
expensive image layout conversion and copying can be avoided but the extra data
will be ignored by the filter.

To unbind a previously set image from the filter, call

    void oidnRemoveFilterImage(OIDNFilter filter, const char* name);

Some special data used by filters are opaque/untyped (e.g. trained model weights
blobs), which can be specified with the `oidnSetSharedFilterData` function:

    void oidnSetSharedFilterData(OIDNFilter filter, const char* name,
                                 void* ptr, size_t byteSize);

Modifying the contents of an opaque data parameter after binding it to a filter
is allowed but the filter needs to be notified that the data has been updated by
calling

    void oidnUpdateFilterData(OIDNFilter filter, const char* name);

Unbinding opaque data from the filter can be performed with

    void oidnRemoveFilterData(OIDNFilter filter, const char* name);

Filters may have parameters other than buffers as well, which you can set and
get using the following functions:

    void  oidnSetFilter1b(OIDNFilter filter, const char* name, bool  value);
    void  oidnSetFilter1i(OIDNFilter filter, const char* name, int   value);
    void  oidnSetFilter1f(OIDNFilter filter, const char* name, float value);
    bool  oidnGetFilter1b(OIDNFilter filter, const char* name);
    int   oidnGetFilter1i(OIDNFilter filter, const char* name);
    float oidnGetFilter1f(OIDNFilter filter, const char* name);

Filters support a progress monitor callback mechanism that can be used to report
progress of filter operations and to cancel them as well. Calling
`oidnSetFilterProgressMonitorFunction` registers a progress monitor callback
function (`func` argument) with payload (`userPtr` argument) for the specified
filter (`filter` argument):

    typedef bool (*OIDNProgressMonitorFunction)(void* userPtr, double n);

    void oidnSetFilterProgressMonitorFunction(OIDNFilter filter,
                                              OIDNProgressMonitorFunction func,
                                              void* userPtr);

Only a single callback function can be registered per filter, and further
invocations overwrite the previously set callback function. Passing `NULL` as
function pointer disables the registered callback function. Once registered,
Intel Open Image Denoise will invoke the callback function multiple times during
filter operations, by passing the payload as set at registration time
(`userPtr` argument), and a `double` in the range [0, 1] which estimates the
progress of the operation (`n` argument). When returning `true` from the
callback function, Intel Open Image Denoise will continue the filter operation
normally. When returning `false`, the library will cancel the filter operation
with the `OIDN_ERROR_CANCELLED` error code.

After setting all necessary parameters for the filter, the changes must be
commmitted by calling

    void oidnCommitFilter(OIDNFilter filter);

The parameters can be updated after committing the filter, but it must be
re-committed for any new changes to take effect. Committing major changes to the
filter (e.g. setting new image parameters, changing the image resolution) can
be expensive, and thus should not be done frequently (e.g. per frame).

Finally, an image can be filtered by executing the filter with

    void oidnExecuteFilter(OIDNFilter filter);

which will read the input image data from the specified buffers and produce the
denoised output image.

In the following we describe the different filters that are currently
implemented in Intel Open Image Denoise.

### RT

The `RT` (**r**ay **t**racing) filter is a generic ray tracing denoising filter
which is suitable for denoising images rendered with Monte Carlo ray tracing
methods like unidirectional and bidirectional path tracing. It supports depth
of field and motion blur as well, but it is *not* temporally stable. The filter
is based on a convolutional neural network (CNN), and it aims to provide a
good balance between denoising performance and quality. The filter comes with a
set of pre-trained CNN models that work well with a wide range of ray tracing
based renderers and noise levels.

![Example noisy beauty image rendered using unidirectional path tracing (4
samples per pixel). *Scene by Evermotion.*][imgMazdaColor]

![Example output beauty image denoised using prefiltered auxiliary feature
images (albedo and normal) too.][imgMazdaDenoised]

For denoising *beauty* images, it accepts either a low dynamic range (LDR) or
high dynamic range (HDR) image (`color`) as the main input image. In addition
to this, it also accepts *auxiliary feature* images, `albedo` and `normal`,
which are optional inputs that usually improve the denoising quality
significantly, preserving more details.

It is possible to denoise auxiliary images as well, in which case only the
respective auxiliary image has to be specified as input, instead of the beauty
image. This can be done as a *prefiltering* step to further improve the quality
of the denoised beauty image.

The `RT` filter has certain limitations regarding the supported input images.
Most notably, it cannot denoise images that were not rendered with ray tracing.
Another important limitation is related to anti-aliasing filters. Most
renderers use a high-quality pixel reconstruction filter instead of a trivial
box filter to minimize aliasing artifacts (e.g. Gaussian, Blackman-Harris). The
`RT` filter does support such pixel filters but only if implemented with
importance sampling. Weighted pixel sampling (sometimes called *splatting*)
introduces correlation between neighboring pixels, which causes the denoising
to fail (the noise will not be filtered), thus it is not supported.

The filter can be created by passing `"RT"` to the `oidnNewFilter` function
as the filter type. The filter supports the parameters listed in the table
below. All specified images must have the same dimensions. The output image
can be one of the input images (i.e. in-place denoising is supported). See
section [Examples] for simple code snippets that demonstrate the usage of
the filter.

----------- ------------- ---------- ---------------------------------------------------------------
Type        Name             Default Description
----------- ------------- ---------- ---------------------------------------------------------------
`Image`     `color`       *optional* input beauty image (3 channels, LDR values in [0, 1] or HDR
                                     values in [0, +∞), values being interpreted such that, after
                                     scaling with the `inputScale` parameter, a value of 1
                                     corresponds to a luminance level of 100 cd/m²)

`Image`     `albedo`      *optional* input auxiliary image containing the albedo per pixel (3
                                     channels, values in [0, 1])

`Image`     `normal`      *optional* input auxiliary image containing the shading normal per pixel
                                     (3 channels, world-space or view-space vectors with arbitrary
                                     length, values in [-1, 1])

`Image`     `output`                 output image (3 channels); can be one of the input images

`bool`      `hdr`              false whether the main input image is HDR

`bool`      `srgb`             false whether the main input image is encoded with the sRGB (or 2.2
                                     gamma) curve (LDR only) or is linear; the output will be
                                     encoded with the same curve

`float`     `inputScale`         NaN scales values in the main input image before filtering, without
                                     scaling the output too, which can be used to map color or
                                     auxiliary feature values to the expected range, e.g. for
                                     mapping HDR values to physical units (which affects the quality
                                     of the output but *not* the range of the output values); if set
                                     to NaN, the scale is computed implicitly for HDR images or set
                                     to 1 otherwise

`bool`      `cleanAux`         false whether the auxiliary feature (albedo, normal) images are
                                     noise-free; recommended for highest quality but should *not* be
                                     enabled for noisy auxiliary images to avoid residual noise

`Data`      `weights`     *optional* trained model weights blob

`int`       `maxMemoryMB`       3000 approximate maximum scratch memory to use in megabytes (actual
                                     memory usage may be higher); limiting memory usage may cause
                                     slower denoising due to internally splitting the image into
                                     overlapping tiles

`const int` `alignment`              when manually denoising in tiles, the tile size and offsets
                                     should be multiples of this amount of pixels to avoid
                                     artifacts; when denoising HDR images `inputScale` *must* be set
                                     by the user to avoid seam artifacts

`const int` `overlap`                when manually denoising in tiles, the tiles should overlap by
                                     this amount of pixels

----------- ------------- ---------- ---------------------------------------------------------------
: Parameters supported by the `RT` filter.

Using auxiliary feature images like albedo and normal helps preserving fine
details and textures in the image thus can significantly improve denoising
quality. These images should typically contain feature values for the first
hit (i.e. the surface which is directly visible) per pixel. This works well for
most surfaces but does not provide any benefits for reflections and objects
visible through transparent surfaces (compared to just using the color as
input). However, this issue can be usually fixed by storing feature values for
a subsequent hit (i.e. the reflection and/or refraction) instead of the first
hit. For example, it usually works well to follow perfect specular (*delta*)
paths and store features for the first diffuse or glossy surface hit instead
(e.g. for perfect specular dielectrics and mirrors). This can greatly improve
the quality of reflections and transmission. We will describe this approach in
more detail in the following subsections.

The auxiliary feature images should be as noise-free as possible. It is not a
strict requirement but too much noise in the feature images may cause residual
noise in the output. Ideally, these should be completely noise-free. If this
is the case, this should be hinted to the filter using the `cleanAux` parameter
to ensure the highest possible image quality. But this parameter should be used
with care: if enabled, any noise present in the auxiliary images will end up in
the denoised image as well, as residual noise. Thus, `cleanAux` should be
enabled only if the auxiliary images are guaranteed to be noise-free.

Usually it is difficult to provide clean feature images, and some residual
noise might be present in the output even with `cleanAux` being disabled. To
eliminate this noise and to even improve the sharpness of texture details, the
auxiliary images should be first denoised in a prefiltering step, as mentioned
earlier. Then, these denoised auxiliary images could be used for denoising the
beauty image. Since these are now noise-free, the `cleanAux` parameter should be
enabled. See section [Denoising with prefiltering (C++11 API)] for a simple
code example. Prefiltering makes denoising much more expensive but if there are
multiple color AOVs to denoise, the prefiltered auxiliary images can be reused
for denoising multiple AOVs, amortizing the cost of the prefiltering step.

Thus, for final frame denoising, where the best possible image quality is
required, it is recommended to prefilter the auxiliary features if they are
noisy and enable the `cleanAux` parameter. Denoising with noisy auxiliary
features should be reserved for previews and interactive rendering.

All auxiliary images should use the same pixel reconstruction filter as the
beauty image. Using a properly anti-aliased beauty image but aliased albedo or
normal images will likely introduce artifacts around edges.

#### Albedo

The albedo image is the feature image that usually provides the biggest quality
improvement. It should contain the approximate color of the surfaces
independent of illumination and viewing angle.

![Example albedo image obtained using the first hit. Note that the albedos of
all transparent surfaces are 1.][imgMazdaAlbedoFirstHit]

![Example albedo image obtained using the first diffuse or glossy (non-delta)
hit. Note that the albedos of perfect specular (delta) transparent surfaces
are computed as the Fresnel blend of the reflected and transmitted
albedos.][imgMazdaAlbedoNonDeltaHit]

For simple matte surfaces this means using the diffuse color/texture as the
albedo. For other, more complex surfaces it is not always obvious what is
the best way to compute the albedo, but the denoising filter is flexibile to
a certain extent and works well with differently computed albedos. Thus it is
not necessary to compute the strict, exact albedo values but must be always
between 0 and 1.

For metallic surfaces the albedo should be either the reflectivity at normal
incidence (e.g. from the artist friendly metallic Fresnel model) or the
average reflectivity; or if these are constant (not textured) or unknown, the
albedo can be simply 1 as well.

The albedo for dielectric surfaces (e.g. glass) should be either 1 or, if the
surface is perfect specular (i.e. has a delta BSDF), the Fresnel blend of the
reflected and transmitted albedos. The latter usually works better but only
if it does not introduce too much noise or the albedo is prefiltered. If noise
is an issue, we recommend to split the path into a reflected and a transmitted
path at the first hit, and perhaps fall back to an albedo of 1 for subsequent
dielectric hits. The reflected albedo in itself can be used for mirror-like
surfaces as well.

The albedo for layered surfaces can be computed as the weighted sum of the
albedos of the individual layers. Non-absorbing clear coat layers can be simply
ignored (or the albedo of the perfect specular reflection can be used as well)
but absorption should be taken into account.

#### Normal

The normal image should contain the shading normals of the surfaces either in
world-space or view-space. It is recommended to include normal maps to
preserve as much detail as possible.

![Example normal image obtained using the first hit (the values are actually
in [−1, 1] but were mapped to [0, 1] for illustration
purposes).][imgMazdaNormalFirstHit]

![Example normal image obtained using the first diffuse or glossy (non-delta)
hit. Note that the normals of perfect specular (delta) transparent surfaces
are computed as the Fresnel blend of the reflected and transmitted
normals.][imgMazdaNormalNonDeltaHit]

Just like any other input image, the normal image should be anti-aliased (i.e.
by accumulating the normalized normals per pixel). The final accumulated
normals do not have to be normalized but must be in the [-1, 1] range (i.e.
normals mapped to [0, 1] are *not* acceptable and must be remapped to [−1, 1]).

Similar to the albedo, the normal can be stored for either the first
or a subsequent hit (if the first hit has a perfect specular/delta BSDF).

#### Weights

Instead of using the built-in trained models for filtering, it is also possible
to specify user-trained models at runtime. This can be achieved by passing the
model *weights* blob corresponding to the specified set of features and other
filter parameters, produced by the included training tool. See Section
[Training] for details.

### RTLightmap

The `RTLightmap` filter is a variant of the `RT` filter optimized for denoising
HDR and normalized directional (e.g. spherical harmonics) lightmaps. It does not
support LDR images.

The filter can be created by passing `"RTLightmap"` to the `oidnNewFilter`
function as the filter type. The filter supports the following parameters:

----------- ------------- ---------- ---------------------------------------------------------------
Type        Name             Default Description
----------- ------------- ---------- ---------------------------------------------------------------
`Image`     `color`                  input beauty image (3 channels, HDR values in [0, +∞),
                                     interpreted such that, after scaling with the `inputScale`
                                     parameter, a value of 1 corresponds to aluminance level of 100
                                     cd/m²; directional values in [-1, 1])

`Image`     `output`                 output image (3 channels); can be one of the input images

`bool`      `directional`      false whether the input contains normalized coefficients (in [-1, 1])
                                     of a directional lightmap (e.g. normalized L1 or higher
                                     spherical harmonics band with the L0 band divided out); if the
                                     range of the coefficients is different from [-1, 1], the
                                     `inputScale` parameter can be used to adjust the range without
                                     changing the stored values

`float`     `inputScale`         NaN scales input color values before filtering, without scaling the
                                     output too, which can be used to map color values to the
                                     expected range, e.g. for mapping HDR values to physical units
                                     (which affects the quality of the output but *not* the range of
                                     the output values); if set to NaN, the scale is computed
                                     implicitly for HDR images or set to 1 otherwise

`Data`      `weights`     *optional* trained model weights blob

`int`       `maxMemoryMB`       3000 approximate maximum scratch memory to use in megabytes (actual
                                     memory usage may be higher); limiting memory usage may cause
                                     slower denoising due to internally splitting the image into
                                     overlapping tiles

`const int` `alignment`              when manually denoising in tiles, the tile size and offsets
                                     should be multiples of this amount of pixels to avoid
                                     artifacts; when denoising HDR images `inputScale` *must* be set
                                     by the user to avoid seam artifacts

`const int` `overlap`                when manually denoising in tiles, the tiles should overlap by
                                     this amount of pixels

----------- ------------- ---------- ---------------------------------------------------------------
: Parameters supported by the `RTLightmap` filter.
