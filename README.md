<h1 align="center"> AndroidCameraX </h1>

<p align="center">
  <img src='https://github.com/ayhanunal/AndroidCameraX/blob/main/ss/1.jpg' width=200 heihgt=300> 
</p>

### Gradle
Add the dependency you want to use to your `build.gradle` file
```Gradle
def camerax_version = "1.0.0"
// CameraX core library using camera2 implementation
implementation "androidx.camera:camera-camera2:$camerax_version"
// CameraX Lifecycle Library
implementation "androidx.camera:camera-lifecycle:$camerax_version"
// CameraX View class
implementation "androidx.camera:camera-view:1.0.0-alpha24"
```

### Manifest
Add the permission you want to use to your `Manifest` file
```xml
<uses-feature android:name="android.hardware.camera.any" />
<uses-permission android:name="android.permission.CAMERA" />
```
`.any` means that it can be a front camera or a back camera.

## MainActivity.kt
Request camera permission
```kotlin
override fun onRequestPermissionsResult(
   requestCode: Int, permissions: Array<String>, grantResults:
   IntArray) {
   if (requestCode == REQUEST_CODE_PERMISSIONS) {
       if (allPermissionsGranted()) {
           startCamera()
       } else {
           Toast.makeText(this,
               "Permissions not granted by the user.",
               Toast.LENGTH_SHORT).show()
           finish()
       }
   }
}
```
In a camera application, the viewfinder is used to let the user preview the photo they will be taking. You can implement a viewfinder using the CameraX Preview class. To use Preview, you'll first need to define a configuration, which then gets used to create an instance of the use case. The resulting instance is what you bind to the CameraX lifecycle.
```kotlin
private fun startCamera() {
  val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

  cameraProviderFuture.addListener(Runnable {
      // Used to bind the lifecycle of cameras to the lifecycle owner
      val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()

      // Preview
      val preview = Preview.Builder()
          .build()
          .also {
              it.setSurfaceProvider(viewFinder.surfaceProvider)
          }

      imageCapture = ImageCapture.Builder()
          .build()

      val imageAnalyzer = ImageAnalysis.Builder()
          .build()
          .also {
              it.setAnalyzer(cameraExecutor, LuminosityAnalyzer { luma ->
                  Log.d(TAG, "Average luminosity: $luma")
              })
          }

      // Select back camera as a default
      val cameraSelector = CameraSelector.DEFAULT_BACK_CAMERA

      try {
          // Unbind use cases before rebinding
          cameraProvider.unbindAll()

          // Bind use cases to camera
          cameraProvider.bindToLifecycle(
              this, cameraSelector, preview, imageCapture, imageAnalyzer)

      } catch(exc: Exception) {
          Log.e(TAG, "Use case binding failed", exc)
      }

  }, ContextCompat.getMainExecutor(this))
}
```

Other use cases work in a very similar way as Preview. First, you define a configuration object that is used to instantiate the actual use case object. To capture photos, you'll implement the takePhoto() method, which is called when the Take photo button is pressed .


```kotlin
private fun takePhoto() {
    // Get a stable reference of the modifiable image capture use case
    val imageCapture = imageCapture ?: return

    // Create time-stamped output file to hold the image
    val photoFile = File(
        outputDirectory,
        SimpleDateFormat(FILENAME_FORMAT, Locale.US
        ).format(System.currentTimeMillis()) + ".jpg")

    // Create output options object which contains file + metadata
    val outputOptions = ImageCapture.OutputFileOptions.Builder(photoFile).build()

    // Set up image capture listener, which is triggered after photo has
    // been taken
    imageCapture.takePicture(
        outputOptions, ContextCompat.getMainExecutor(this), object : ImageCapture.OnImageSavedCallback {
            override fun onError(exc: ImageCaptureException) {
                Log.e(TAG, "Photo capture failed: ${exc.message}", exc)
            }

            override fun onImageSaved(output: ImageCapture.OutputFileResults) {
                val savedUri = Uri.fromFile(photoFile)
                val msg = "Photo capture succeeded: $savedUri"
                Toast.makeText(baseContext, msg, Toast.LENGTH_SHORT).show()
                Log.d(TAG, msg)
            }
        })
}
```
A great way to make your camera app more interesting is using the ImageAnalysis feature. It allows you to define a custom class that implements the ImageAnalysis.Analyzer interface, and which will be called with incoming camera frames. You won't have to manage the camera session state or even dispose of images; binding to our app's desired lifecycle is sufficient, like with other lifecycle-aware components.

## create private class 'LuminosityAnalyzer'

```kotlin
private class LuminosityAnalyzer(private val listener: LumaListener) : ImageAnalysis.Analyzer {

   private fun ByteBuffer.toByteArray(): ByteArray {
       rewind()    // Rewind the buffer to zero
       val data = ByteArray(remaining())
       get(data)   // Copy the buffer into a byte array
       return data // Return the byte array
   }

   override fun analyze(image: ImageProxy) {

       val buffer = image.planes[0].buffer
       val data = buffer.toByteArray()
       val pixels = data.map { it.toInt() and 0xFF }
       val luma = pixels.average()

       listener(luma)

       image.close()
   }
}
```

## and open Logcat
<p align="center">
  <img src='https://github.com/ayhanunal/AndroidCameraX/blob/main/ss/2.png'> 
</p>









