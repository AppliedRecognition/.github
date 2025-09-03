# Ver-ID SDK for Android

The Ver-ID SDK is a collection of libraries for face detection, face recognition and spoof detection.

| Library | Description |
| --- | --- |
| [**Face capture**](https://github.com/AppliedRecognition/Face-Capture-Android) | Capture faces that can be used for face recognition |
| [**Face template registry**](https://github.com/AppliedRecognition/Face-Template-Registry-Android) | Handles face registration, authentication and identification |
| [**Face detection**](https://github.com/AppliedRecognition/Face-Detection-RetinaFace-Android) | Detect faces in images |
| [**Spoof device detection**](https://github.com/AppliedRecognition/Spoof-Device-Detection-Ver-ID-3-Android) | Detect spoof devices in images |
| [**Face recognition ArcFace**](https://github.com/AppliedRecognition/Face-Recognition-ArcFace-Android) | Extract face templates from detected faces and compare them |
| [**Face recognition Dlib** (legacy)](https://github.com/AppliedRecognition/Face-Recognition-Dlib-Android) |Extract face templates from detected faces and compare them |
| [**Passport reader**](https://github.com/AppliedRecognition/Passport-Reader-Android) | Read information and image from passport NFC chip |
| [**Ver-ID 2–3 migration**](https://github.com/AppliedRecognition/Ver-ID-2-3-Migration-Android) | Migrate face templates from Ver-ID 2.* to Ver-ID 3+ |
| [**Serialization**](https://github.com/AppliedRecognition/Ver-ID-3-Serialization-Android) | Serialize types using protocol buffers |
| [**Common types**](https://github.com/AppliedRecognition/Ver-ID-Common-Types-Android) | Facilitate library interoperability |
| [**Face classification**](https://github.com/AppliedRecognition/Face-Classification-Android) | Extract attributes from face, e.g. glasses, sunglasses, face covering | 

# Packages

The following packages are released on Maven Central. We advise using the Ver-ID bill of materials (BOM) platform as a contract to keep the libraries on compatible versions. To use the BOM, declare a platform dependency in your build file and omitting the version specification in other Ver-ID dependency declarations. For example:

```kotlin
implementation(platform("com.appliedrec:verid-bom:<bom-version>"))
implementation("com.appliedrec:verid-face-capture")
```

| Library | Package names |
| --- | --- |
| **Face capture** | `com.appliedrec:verid-face-capture` |
| **Face template registry** | `com.appliedrec:face-template-registry` |
| **Face detection RetinaFace** | `com.appliedrec:face-detection-retinaface` |
| **Spoof device detection*** | `com.appliedrec:spoof-device-detection-core`<br />`com.appliedrec:spoof-device-detection-cloud` |
| **Face recognition ArcFace*** | `com.appliedrec:face-recognition-arcface-core`<br />`com.appliedrec:face-recognition-arcface-cloud` |
| **Face recognition Dlib** (legacy) | `com.appliedrec:face-recognition-dlib` |
| **Passport reader** | `com.appliedrec:mtrd-reader` |
| **Ver-ID 2–3 migration** | `com.appliedrec:verid-2-3-migration` |
| **Serialization** | `com.appliedrec:verid-serialization` |
| **Common types** | `com.appliedrec:verid-common` |
| **BOM** (bill of materials) | `com.appliedrec:verid-bom` |

**Connects to a server backend. On-device (offline) libraries available. Please [contact Applied Recognition](mailto:info@appliedrecognition.com) for access and pricing.*

# Capturing a face

```kotlin
val activity: ComponentActivity // Activity in which the face capture will be launched
coroutineScope.launch(Dispatchers.Default) {
    val result = FaceCapture.captureFaces(activity) {
        // Configure the capture
        // Set the face detection engine
        createFaceDetection = { 
            FaceDetectionRetinaFace.create(activity)
        }
        // Configure spoof detection
        createFaceTrackingPlugins = {
            // Use spoof device detection
            val spoofDetection = SpoofDeviceDetection(activity)
            listOf(
                LivenessDetectionPlugin(arrayOf(spoofDetection)) as FaceTrackingPlugin<Any>
            )
        }
    }
    when (result) {
        is FaceCaptureSessionResult.Success -> {
            // Capture succeded, use result.capturedFaces.first() for face recognition
        }
        is FaceCaptureSessionResult.Failure -> {
            // Capture failed, check result.error
        }
        is FaceCaptureSessionResult.Cancelled -> {
            // Capture cancelled by user
        }
    }
}
```

# Face recognition

### Extracting face templates

Face recognition works by comparing face templates. Face templates are extracted from the image in which the face was detected. The image is cropped to the face region and aligned to the facial landmarks prior to being submitted to a machine-learning model. The model produces a vector that can be compared to other face template vectors.

```kotlin
coroutineScope.launch(Dispatchers.Default) {
    try {
        // Create a face recognition instance
        val template = FaceRecognitionArcFace(context).use { faceRecognition ->
            // Extract a face template from the captured face and image
            faceRecognition.createFaceRecognitionTemplates(
                listOf(capture.face), capture.image
            ).first()
        }
        // The template can be compared to other face templates
    } catch (e: Exception) {
    }
}
```

### Face template comparison

```kotlin
coroutineScope.launch(Dispatchers.Default) {
    try {
        // Create a face recognition instance
        val score = FaceRecognitionArcFace(context).use { faceRecognition ->
            // Compare templates
            faceRecognition.compareFaceRecognitionTemplates(
                listOf(template1),
                template2
            ).first()
        }
        // The score ranges from 0 to 1, where 1 indicates the exact same face
        // Typically any faces that compare with a score greater than 0.5 can
        // be considered to belong to the same person
    } catch (e: Exception) {
    }
}
```

# Face template registry

The registry library offers functions to register, authenticate and identify tagged face templates. Unlike in previous versions of the Ver-ID SDK, the face templates are no longer persisted by Ver-ID. The consumer supplies the most recent face template set to the face template registry and the registry outputs the faces it has added in the call results.

### Face template registry initialization

```kotlin
// Create face recognition instance
FaceRecognitionArcFace(context).use { faceRecognition ->
    // Example: convert your data type to TaggedFaceTemplate array
    val faceTemplates = taggedFaces.map { face in
        TaggedFaceTemplate(face.template, face.userName)
    }
    // Create the registry
    FaceTemplateRegistry(faceRecognition, faceTemplates).use { registry ->
        // Use registry here
    }
}
```

### Face registration

The registration iterates over all registered face templates and checks that the face template that's being registered is not too similar to a template registered under a different identifier. The registry also checks that the face template is not too different from other templates registered for the identifier. If either case is true the registry throws an error, which contains the extracted face template. This allows you to override the registry's decision and add the extracted face template to your set.

Otherwise the registry outputs the extracted face template that you can safely add to your set.

```kotlin
// Input is face and image captured using FaceCapture (see above) and an identifier
// of the user to be registered
coroutineScope.launch(Dispatchers.Default) {
    try {
        // See face template registry initialization above
        val template = FaceTemplateRegistry(faceRecognition, faceTemplates).use { registry ->
            registry.registerFace(face, image, userId)
        }
        // You can add template to your face set
    } catch (e: FaceTemplateRegistryException.SimilarFaceAlreadyRegistered) {
        // A similar face is already registered as e.registeredIdentifier
        // You can still add the e.faceTemplate to your faces set but identifications 
        // may be ambiguous
    } catch (e: FaceTemplateRegistryException.FaceDoesNotMatchExisting) {
        // The face doesn't match the existing faces registered as the user ID
        // You can still add the e.faceTemplate to your faces set but you may wish to check
        // that you have the correct user identifier.
    } catch (e: Exception) {
        // Something else went wrong
    }
}
```

### Face identification

The registry identifies the face templates in its set that best match a submitted face. The identification call returns an array of tagged face templates that meet a configured threshold. The results are oreder by the best matching face template and are uniqued by the user identifier.

```kotlin
// Input is face and image captured using FaceCapture (see above)
coroutineScope.launch(Dispatchers.Default) {
    try {
        // See face template registry initialization above
        val identifications = FaceTemplateRegistry(faceRecognition, faceTemplates).use { registry ->
            registry.identifyFace(face, image)
        }
        identifications.firstOrNull()?.taggedFaceTemplate?.identifier?.let { identifiedUserId ->
            if (identifications.size > 1) {
                // Identified multiple users, you may wish to confirm the user's identity
            } else {
                // Identified a user with id identifiedUserId
            }
        } ?: run {
            // None of the registered face templates matched the submitted face template
        }
    } catch (e: Exception) {
        // Something else went wrong
    }
}
```

### Face authentication

The registry compares the submitted face to the face templates registered under the submitted user identifier. The result contains the comparison result, the challenge face template extracted from the submitted face and image, the registered face template that best matched the challenge template and the score with which the templates matched.

If the registry doesn't have any face templates registered under the user identifier the call throws an error.

```kotlin
// Input is face and image captured using FaceCapture (see above) and an identifier
// of the user to be authenticated
coroutineScope.launch(Dispatchers.Default) {
    try {
        val result = FaceTemplateRegistry(faceRecognition, faceTemplates).use { registry ->
            registry.authenticateFace(face, image, userId)
        }
        // Check the result
        if result.authenticated {
            // The authentication succeeded
        } else {
            // The authentication failed
        }
    } catch (e: FaceTemplateRegistryException.IdentifierNotRegistered) {
        // The registry doesn't have any face templates tagged with the submitted identifier
    } catch (e: Exception) {
        // Something else went wrong
    }
}
```

# Face template multi-registry

Face recognition templates are not universally comparable. Only templates generated by the same face recognition algorithm can be compared to each other. The multi-registry coordinates between registries containing templates from different face recognition systems.

The multi-registry can assist with migrating from one face recognition system to another by automatically registering face templates at authentication or identification. The multi-registry also ensures that all the users in its registries can be safely compared to any given challenge face.

### Multi-registry initialization

```swift
// First face recognition instance
val faceRecognition1: FaceRecognitionArcFace
    
// Tagged face templates to populate the first registry
val faceTemplates1: List<TaggedFaceTemplate<FaceTemplateVersion24, FloatArray>>

// Create first registry instance
val registry1 = FaceTemplateRegistry(faceRecognition1, faceTemplates1)

// Second face recognition instance
val faceRecognition2: FaceRecognition3D
    
// Tagged face templates to populate the second registry
val faceTemplates2: List<TaggedFaceTemplate<FaceTemplateVersion3D1, FloatArray>>

// Create second registry instance
val registry2 = FaceTemplateRegistry(faceRecognition2, faceTemplates2)

// Create multi registry
FaceTemplateMultiRegistry(registry1.typeErased, registry2.typeErased).use { multiRegistry ->
    // You can now use multiRegistry
}
```

### Face registration

The face registration call has an identical signature to the `FaceTemplateRegistry.registerFace` call except it returns an array of face templates instead of one template.

The call will register the face in all the dependent registries.

### Face identification

The face identification call has 2 extra parameters that are not present in the `FaceTemplateRegistry.identifyFace` function:

- `safe` – boolean value that indicates whether to check that the face templates are all compared using one face recognition system. If set to `false`, the multi-registry will use its first dependent registry to identify the face. If that fails, it will proceed to the next registry.<br>If omitted, or set to `true`, the multi-registry ensures that at least one of its dependent registries contains all registered user identifiers. This registry will then be used for the face comparison. If no registry contains a full set of user identifiers the function throws `FaceTemplateRegistryException.IncompatibleFaceTemplates`.
- `autoEnrol` – if omitted, or set to `true` the multi-registry will automatically enrol the identified face in its dependent registries that are missing the identified user identifier. This is useful when migrating from one recognition system to another. As users are identified they are added to the registries where they're missing.

### Face authentication

The call returns a type-erased version of the `FaceTemplateRegistry`'s authentication result. The signature of the `authenticateFace` call is nearly identical to the `FaceTemplateRegistry.authenticateFace` call, with the exception of an additional `autoEnrol` flag. 

If the `autoEnrol` argument is omitted, or set to `true` the multi-registry will automatically enrol the authenticated face in its dependent registries that are missing the authenticated user identifier. This is useful when migrating from one recognition system to another. As users are autenticated they are added to the registries where they're missing.
