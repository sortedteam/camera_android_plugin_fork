# Local patch of camera_android 0.10.10+6

Only file changed: `android/src/main/java/io/flutter/plugins/camera/Camera.java`
(see `camera_android_0.10.10+6.patch` for the exact diff).

## Problem
The app can die with an uncatchable native crash:

    NullPointerException: Attempt to invoke virtual method
    'void CameraCaptureSession.close()' on a null object reference
      at io.flutter.plugins.camera.Camera.closeCaptureSession
      at io.flutter.plugins.camera.Camera$1.onClosed

Seen on a Samsung Galaxy M21 (Exynos). Cause: `closeCaptureSession()` checks
`captureSession != null` and then dereferences it again, while `stopAndReleaseCamera()`
(main thread, from `dispose()`) sets it to null in between.

Upstream (no released fix at the time of writing):
- https://github.com/flutter/flutter/issues/114012  (open, same crash)
- https://github.com/flutter/flutter/issues/185265  (same crash, proposed the local-copy fix)
- https://github.com/flutter/packages/pull/12224    (closed unmerged; reviewer asked for a
  single lock object around captureSession)

## Change
1. `private final Object sessionLock` guards the read-and-clear of
   `captureSession` / `cameraDevice`.
2. `closeCaptureSession()` and `stopAndReleaseCamera()` take the reference and null the field
   inside `synchronized (sessionLock)`, then do the (slow) close OUTSIDE the lock.
3. `refreshPreviewCaptureSession`, `runPrecaptureSequence`, `takePictureAfterPrecapture`,
   `lockAutoFocus`, `unlockAutoFocus`, `pausePreview` read the session once into a local
   variable and handle `IllegalStateException` (camera already closed) instead of crashing.

No behaviour change when nothing is racing.

## Remove when
`camera_android` ships an equivalent fix: delete `third_party/camera_android` and the
`dependency_overrides` entry in `pubspec.yaml`.
