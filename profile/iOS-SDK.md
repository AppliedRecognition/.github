Applied Recognition is a foremost supplier of face recognition technology for time and attendance kiosks and KYC applications.

# Ver-ID SDK for iOS

The Ver-ID SDK is a collection of libraries for face detection, face recognition and spoof detection.

| Library | Description | Repositories |
| --- | --- | --- |
| **Face capture** | Capture faces that can be used for face recognition | [Face capture](https://github.com/AppliedRecognition/Face-Capture-Apple) |
| **Face recognition** | Extract face templates from detected faces and compare them | [Face recognition ArcFace](https://github.com/AppliedRecognition/Face-Recognition-ArcFace-Apple)<br>[Face recognition Dlib (legacy)](https://github.com/AppliedRecognition/Face-Recognition-Dlib-Apple) |
| **Face detection** | Detect faces in images | [Face detection RetinaFace](https://github.com/AppliedRecognition/Face-Detection-RetinaFace-Apple) |
| **Spoof detection** | Detect spoofs in images | [Spoof device detection](https://github.com/AppliedRecognition/Spoof-Device-Detection-Ver-ID-3-Apple)<br>[FASnet spoof detection](https://github.com/AppliedRecognition/Spoof-Detection-Fasnet-Apple)<br>[Fusion spoof detection](https://github.com/AppliedRecognition/Spoof-Detection-Fusion-Apple) |
| **Common types** | Facilitate library interoperability | [Common types](https://github.com/AppliedRecognition/Ver-ID-Common-Types-Apple) |
| **Serialization** | Serialize types using protocol buffers | [Serialization](https://github.com/AppliedRecognition/Ver-ID-3-Serialization-Apple) |

The libraries are vendored using Swift Package Manager. To consume the libraries add their Git repo address to your project's Swift Package dependencies and select the desired version constraints.

# Capturing a face

```swift
Task(priority: .utility) {
    let result = await FaceCapture.captureFaces { configuration in
        // Configure the capture
        // Set the face detection engine
        configuration.faceDetection = try FaceDetectionRetinaFace()
        // Configure spoof detection
        // Check if the device can capture depth
        if FaceCaptureSession.supportsDepthCaptureOnDeviceAt(.front) {
            // Use depth-based liveness detection
            configuration.faceTrackingPlugins = [DepthLivenessDetection()]
        } else {
            // Use spoof device detection
            let spoofDeviceDetection = SpoofDeviceDetection(apiKey: apiKey, url: url)
            configuration.faceTrackingPlugins = [
                try LivenessDetectionPlugin(spoofDetectors: [spoofDeviceDetection])
            ]
        }
    }
    // Check the result
    switch result {
    case .success(capturedFaces: let faces, metadata: _):
        let capturedFace = faces.first!
        // You can submit capturedFace.face and capturedFace.image
        // For face recognition
    case .failure(capturedFaces: _, metadata: _, error: let error):
        // Face capture failed
        break
    case .cancelled:
        break
    }
}
```

# Face recognition

### Extracting face templates

Face recognition works by comparing face templates. Face templates are extracted from the image in which the face was detected. The image is cropped to the face region and aligned to the facial landmarks prior to being submitted to a machine-learning model. The model produces a vector that can be compared to other face template vectors.

```swift
Task(priority: .utility) {
    do {
        // Create a face recognition instance
        let faceRecognition = FaceRecognitionArcFace(apiKey: apiKey, url: url)
        // Extract a face template from the captured face and image
        let template = try await faceRecognition
            .createFaceRecognitionTemplates(from: [capture.face], in: capture.image).first!
        // The template can be compared to other face templates
    } catch {
    }
}
```

### Face template comparison

```swift
Task(priority: .utility) {
    do {
        // Create a face recognition instance
        let faceRecognition = FaceRecognitionArcFace(apiKey: apiKey, url: url)
        // Compare templates
        let score = try await faceRecognition
            .compareFaceRecognitionTemplates([template1], to: template2).first!
        // The score ranges from 0 to 1, where 1 indicates the exact same face
        // Typically any faces that compare with a score greater than 0.5 can
        // be considered to belong to the same person
    } catch {
    }
}
```

# Face template registry

The registry library offers functions to register, authenticate and identify tagged face templates. Unlike in previous versions of the Ver-ID SDK, the face templates are no longer persisted by Ver-ID. The consumer supplies the most recent face template set to the face template registry and the registry outputs the faces it has added in the call results.

### Face template registry initialization

```swift
// Create face recognition instance
let faceRecognition = FaceRecognitionArcFace(apiKey: apiKey, url: url)
// Example: convert your data type to TaggedFaceTemplate array
let faceTemplates = taggedFaces.map { face in
    TaggedFaceTemplate(faceTemplate: face.template, identifier: face.userName)
}
// Create the registry
let registry = FaceTemplateRegistry(faceRecognition: faceRecognition, faceTemplates: faceTemplates)
```

### Face registration

The registration iterates over all registered face templates and checks that the face template that's being registered is not too similar to a template registered under a different identifier. The registry also checks that the face template is not too different from other templates registered for the identifier. If either case is true the registry throws an error, which contains the extracted face template. This allows you to override the registry's decision and add the extracted face template to your set.

Otherwise the registry outputs the extracted face template that you can safely add to your set.

```swift
// Input is face and image captured using FaceCapture (see above) and an identifier
// of the user to be registered
Task(priority: .utility) {
    let registry: FaceTemplateRegistry // see face template registry initialization
    do {
        let template = try await registry.registerFace(face, image: image, identifier: userId)
        // You can add template to your face set
    } catch FaceTemplateRegistryError.similarFaceAlreadyRegisteredAs(let existingUser, let template, _) {
        // A similar face is already registered as existingUser
        // You can still add the template to your faces set but identifications 
        // may be ambiguous
    } catch FaceTemplateRegistryError.faceDoesNotMatchExisting(let template, let maxScore) {
        // The face doesn't match the existing faces registered as the user ID
        // You can still add the template to your faces set but you may wish to check
        // that you have the correct user identifier.
    } catch {
        // Something else went wrong
    }
}
```

### Face identification

The registry identifies the face templates in its set that best match a submitted face. The identification call returns an array of tagged face templates that meet a configured threshold. The results are oreder by the best matching face template and are uniqued by the user identifier.

```swift
// Input is face and image captured using FaceCapture (see above)
Task(priority: .utility) {
    let registry: FaceTemplateRegistry // see face template registry initialization
    do {
        let identifications = try await registry.identifyFace(face, image: image)
        if let identifiedUserId = identifications.first?.taggedFaceTemplate.identifier {
            if identifications.count > 1 {
                // Identified multiple users, you may wish to confirm the user's identity
            } else {
                // Identified a user with id identifiedUserId
            }
        } else {
            // None of the registered face templates matched the submitted face template
        }
    } catch {
    }
}
```

### Face authentication

The registry compares the submitted face to the face templates registered under the submitted user identifier. The result contains the comparison result, the challenge face template extracted from the submitted face and image, the registered face template that best matched the challenge template and the score with which the templates matched.

If the registry doesn't have any face templates registered under the user identifier the call throws an error.

```swift
// Input is face and image captured using FaceCapture (see above) and an identifier
// of the user to be authenticated
Task(priority: .utility) {
    let registry: FaceTemplateRegistry // see face template registry initialization
    do {
        let result = try await registry.authenticateFace(face, image: image, identifier: userId)
        // Check the result
        if result.authenticated {
            // The authentication succeeded
        } else {
            // The authentication failed
        }
    } catch FaceTemplateRegistryError.identifierNotRegistered(_) {
        // The registry doesn't have any face templates tagged with the submitted identifier
    } catch {
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
let faceRecognition1: FaceRecognitionArcFace
    
// Tagged face templates to populate the first registry
let faceTemplates1: [TaggedFaceTemplate<V24, [Float]>>]

// Create first registry instance
let registry1 = FaceTemplateRegistry(
    faceRecognition: faceRecognition1, 
    faceTemplates: faceTemplates1
)

// Second face recognition instance
let faceRecognition2: FaceRecognition3D
    
// Tagged face templates to populate the second registry
let faceTemplates2: [TaggedFaceTemplate<FaceTemplateVersion3D1, [Float]>]

// Create second registry instance
let registry2 = FaceTemplateRegistry(
    faceRecognition: faceRecognition2, 
    faceTemplates: faceTemplates2
)

// Create multi registry
let multiRegistry = try await FaceTemplateMultiRegistry(
    registries: registry1.eraseToAnyFaceTemplateRegistry(),
    registry2.eraseToAnyFaceTemplateRegistry()
)
```

### Face registration

The face registration call has an identical signature to the `FaceTemplateRegistry.registerFace` call except it returns an array of face templates instead of one template.

The call will register the face in all the dependent registries.

### Face identification

The face identification call has 2 extra parameters that are not present in the `FaceTemplateRegistry.identifyFace` function:

- `safe` – boolean value that indicates whether to check that the face templates are all compared using one face recognition system. If set to `false`, the multi-registry will use its first dependent registry to identify the face. If that fails, it will proceed to the next registry.<br>If omitted, or set to `true`, the multi-registry ensures that at least one of its dependent registries contains all registered user identifiers. This registry will then be used for the face comparison. If no registry contains a full set of user identifiers the function throws `FaceTemplateRegistryError.incompatibleFaceTemplates`.
- `autoEnrol` – if omitted, or set to `true` the multi-registry will automatically enrol the identified face in its dependent registries that are missing the identified user identifier. This is useful when migrating from one recognition system to another. As users are identified they are added to the registries where they're missing.

### Face authentication

The call returns a type-erased version of the `FaceTemplateRegistry`'s authentication result. The signature of the `authenticateFace` call is nearly identical to the `FaceTemplateRegistry.authenticateFace` call, with the exception of an additional `autoEnrol` flag. 

If the `autoEnrol` argument is omitted, or set to `true` the multi-registry will automatically enrol the authenticated face in its dependent registries that are missing the authenticated user identifier. This is useful when migrating from one recognition system to another. As users are autenticated they are added to the registries where they're missing.
