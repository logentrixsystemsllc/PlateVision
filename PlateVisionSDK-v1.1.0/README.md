# PlateVision SDK for iOS

PlateVision provides on-device licence-plate recognition for iOS. This repository contains the compiled SDK distribution; it does not contain the SDK source code.

## Requirements

- iOS 16.0 or later
- Swift 5 compatible application target
- Camera permission when using the SDK-managed capture session
- PlateVision evaluation or production credentials

## Repository contents

```text
PlateVisionSDK.xcframework/   Compiled device and simulator framework
README.md                     Integration guide
LICENSE.md                    Evaluation and redistribution terms
```

The XCFramework includes:

- iOS device: `arm64`
- iOS Simulator: `arm64` and `x86_64`

## Installation

1. Download or clone this repository.
2. Drag `PlateVisionSDK.xcframework` into the Xcode project.
3. Enable **Copy items if needed**.
4. In the application target, open **General → Frameworks, Libraries, and Embedded Content**.
5. Set `PlateVisionSDK.xcframework` to **Embed & Sign**.
6. Add `import PlateVisionSDK` where the SDK is used.

Do not add the framework to the app target more than once.

## Activation

Set the licence-validation endpoint supplied with the evaluation credentials, then activate the SDK:

```swift
import PlateVisionSDK

let scanner = PlateVisionScanner.shared
scanner.licenceValidationURL = URL(
    string: "https://YOUR-LICENSING-HOST/api/validate-licence.php"
)!

scanner.activate(
    apiKey: "YOUR_API_KEY",
    secretKey: "YOUR_SECRET_KEY"
) { result in
    switch result {
    case .success(let licence) where licence.isValid:
        print("PlateVision activated")
    case .success:
        print("Licence is not valid")
    case .failure(let error):
        print("Activation failed: \(error.localizedDescription)")
    }
}
```

Never commit live API or secret keys to a public repository.

## Application-owned camera pipeline

Use the frame-input APIs when the application already owns its `AVCaptureSession`. PlateVision does not create or control the capture session in this workflow.

The SDK accepts:

- `CMSampleBuffer`
- `CVPixelBuffer`
- `CGImage`
- `UIImage`

### Live-frame example

```swift
import AVFoundation
import ImageIO
import PlateVisionSDK

final class FrameProcessor: NSObject, AVCaptureVideoDataOutputSampleBufferDelegate {
    private let frameQueue = DispatchQueue(
        label: "com.platevision.integration.frames",
        qos: .userInitiated
    )
    private var isProcessingFrame = false

    func attach(to output: AVCaptureVideoDataOutput) {
        output.alwaysDiscardsLateVideoFrames = true
        output.setSampleBufferDelegate(self, queue: frameQueue)
    }

    func captureOutput(
        _ output: AVCaptureOutput,
        didOutput sampleBuffer: CMSampleBuffer,
        from connection: AVCaptureConnection
    ) {
        guard !isProcessingFrame else { return }
        isProcessingFrame = true

        PlateVisionScanner.shared.process(
            sampleBuffer: sampleBuffer,
            orientation: .right
        ) { [weak self] result in
            switch result {
            case .success(let detections):
                for detection in detections {
                    print(detection.plateText, detection.boundingBox)
                }
            case .failure(let error):
                print(error.localizedDescription)
            }

            self?.frameQueue.async {
                self?.isProcessingFrame = false
            }
        }
    }
}
```

Use backpressure as shown above. Do not queue every camera frame while recognition is still processing the previous frame.

### Still-image example

```swift
PlateVisionScanner.shared.process(image: image) { result in
    switch result {
    case .success(let detections):
        print(detections)
    case .failure(let error):
        print(error.localizedDescription)
    }
}
```

Frame-input completion handlers run on the main queue. Detection bounding boxes use normalized Vision coordinates for the complete supplied frame, with the origin at the lower-left. `scanROI` applies to the SDK-managed camera workflow; crop a supplied frame in the application if external-frame processing should cover a smaller region.

The SDK does not store or modify externally supplied frames. The application retains responsibility for preview, tracking, redaction, and persistence. This supports applications that must anonymise people or unrelated plates before writing evidence images, but integration behavior and legal compliance remain the application developer's responsibility.

## SDK-managed camera

For applications that want PlateVision to manage camera capture:

```swift
final class ScanHandler: PlateVisionDelegate {
    func start() {
        let scanner = PlateVisionScanner.shared
        scanner.delegate = self
        scanner.startScanning()
    }

    func plateVision(
        _ sdk: PlateVisionScanner,
        didDetect result: PVDetectionResult
    ) {
        print(result.plateText, result.confidencePercent)
    }
}
```

Add `NSCameraUsageDescription` to the application `Info.plist`. Use `PVCameraPreview(sdk: .shared)` in SwiftUI or `PlateVisionScanner.shared.previewLayer` in UIKit.

## Public frame-input API

```swift
process(sampleBuffer:orientation:completion:)
process(pixelBuffer:orientation:completion:)
process(cgImage:orientation:completion:)
process(image:completion:)
```

## Privacy and data handling

- Plate recognition runs on the device.
- Externally supplied frames remain under application control.
- The SDK does not persist externally supplied frames.
- Licence activation communicates with the configured validation endpoint.
- The host application is responsible for consent, retention, redaction, and compliance requirements.

## Troubleshooting

### Unable to find module dependency `PlateVisionSDK`

Confirm that:

1. `PlateVisionSDK.xcframework` is present inside the project directory.
2. It is added to the correct application target.
3. Its embed setting is **Embed & Sign**.
4. The selected XCFramework contains the required device or simulator architecture.
5. Xcode has been closed and reopened after removing stale framework references.

### Camera preview is unavailable

Confirm that `NSCameraUsageDescription` exists and camera permission has been granted. The iOS Simulator does not provide the same camera behavior as a physical device.

## Version

This repository contains PlateVision SDK `1.1.0`.

## Support

Use the support channel supplied with the evaluation or commercial agreement. Do not post API keys, secret keys, captured images, or personal data in a public issue.

