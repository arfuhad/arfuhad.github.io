---
layout: post
title: "Real-Time Face Detection in React Native WebRTC"
date: 2026-02-23 12:00:00 +0600
published: true
tags:
  - react-native
  - webrtc
  - face-detection
  - ml-kit
description: "Add ML Kit face tracking, blink detection, and head pose estimation directly into your React Native WebRTC video pipeline."
---

Standard `react-native-webrtc` handles video calls, but it can't tell you *what's in the frame*. No face detection, no eye tracking, no blink events. If you need liveness checks for KYC, gaze monitoring for proctoring, or attention tracking for telehealth -- you're stuck extracting frames manually, wiring up ML Kit on both platforms, managing threading, and bridging results back to JS.

[`react-native-webrtc-face-detection`](https://github.com/arfuhad/react-native-webrtc-face-detection) is a drop-in replacement that plugs Google ML Kit face detection directly into the WebRTC video pipeline. Everything from the original library works identically -- it just adds perception on top.

---

## What Standard WebRTC Is Missing

| Capability                      | Standard WebRTC | This Package                 |
| ------------------------------- | --------------- | ---------------------------- |
| Video calls, data channels      | Yes             | Yes                          |
| **Face detection**              | No              | Yes (ML Kit, both platforms) |
| **Eye open/closed state**       | No              | Yes (per-eye probability)    |
| **Blink counting**              | No              | Yes (state machine)          |
| **Head pose estimation**        | No              | Yes (yaw / pitch / roll)     |
| **Frame capture on blink**      | No              | Yes (base64 JPEG)            |
| **Face bounding box overlay**   | No              | Yes (animated component)     |
| **React hooks for ML features** | No              | Yes (auto-cleanup)           |

---

## Quick Setup

```bash
npm install react-native-webrtc-face-detection
```

For iOS/Android platform setup (Podfile, permissions, etc.), see the [README](https://github.com/arfuhad/react-native-webrtc-face-detection#installation).

The one critical step: enable face detection at app startup, **before** any hooks are used:

```typescript
import { configureWebRTC } from 'react-native-webrtc-face-detection';

configureWebRTC({ enableFaceDetection: true });
```

Face detection is opt-in. It won't activate unless you call this -- the flag defaults to `false` for performance reasons.

---

## How the Pipeline Works

```
Camera Sensor
     |
     v
RTCVideoSource (captures frames at 30fps)
     |
     v
VideoEffectProcessor
     |
     +---> Frame passes through to RTCView (immediately, no delay)
     |
     +---> FaceDetectionProcessor (every Nth frame, async)
                |
                v
            ML Kit FaceDetector
                |
                v
            faceDetected event --> useFaceDetection hook
            blinkDetected event --> useBlinkDetection hook
```

**Key design decisions:**

- **Non-blocking** -- The video frame passes to the renderer immediately. ML Kit runs on a separate thread, so detection never adds latency to the display. Results lag 1-2 frames behind, which is imperceptible.
- **Frame skipping** -- Only every Nth frame goes to ML Kit (`frameSkipCount`, default: 3). At 30fps camera, that's ~10 detections/sec -- plenty for blink and gaze tracking while keeping CPU reasonable.
- **Atomic guard** -- If ML Kit is still processing when the next frame arrives, that frame is dropped. No queue buildup on slower devices.

### Blink State Machine

```
Eye OPEN (probability >= 0.3)
     |
     |  probability drops below 0.3
     v
Eye CLOSED --> store frame (for captureOnBlink)
     |
     |  probability rises above 0.3
     v
Eye OPEN --> emit blinkDetected event
```

One open-close-open cycle = one blink. Tracked per-face, per-eye using ML Kit's tracking ID. The threshold (0.3) is configurable.

---

## Complete Example

A single-screen app with face overlay, blink tracking, and face capture on blink:

{% raw %}
```tsx
import React, { useState, useEffect } from 'react';
import {
    View,
    Text,
    Image,
    StyleSheet,
    SafeAreaView,
    TouchableOpacity,
} from 'react-native';
import {
    configureWebRTC,
    mediaDevices,
    MediaStream,
    RTCView,
    FaceDetectionOverlay,
    useFaceDetection,
    useBlinkDetection,
} from 'react-native-webrtc-face-detection';

// Enable once at module level
configureWebRTC({ enableFaceDetection: true });

export default function FaceDetectionScreen() {
    const [stream, setStream] = useState<MediaStream | null>(null);
    const [lastFaceImage, setLastFaceImage] = useState<string | null>(null);

    // Get front camera stream
    useEffect(() => {
        let mounted = true;
        mediaDevices.getUserMedia({
            video: { facingMode: 'user', frameRate: 30 },
            audio: false,
        }).then(s => {
            if (mounted) setStream(s);
        }).catch(console.error);

        return () => { mounted = false; };
    }, []);

    // Cleanup on unmount
    useEffect(() => {
        return () => { stream?.getTracks().forEach(t => t.stop()); };
    }, [stream]);

    const videoTrack = stream?.getVideoTracks()[0] ?? null;

    // Face detection hook
    const {
        detectionResult, isEnabled, enable, error,
    } = useFaceDetection(videoTrack, { frameSkipCount: 3 });

    // Blink detection with image capture
    const {
        blinkCount, recentBlinks, getBlinkRate,
        enable: enableBlink, resetCount,
    } = useBlinkDetection(videoTrack, {
        captureOnBlink: true,
        cropToFace: true,
        imageQuality: 0.7,
        maxImageWidth: 480,
    });

    // Enable when track is ready
    useEffect(() => {
        if (videoTrack) { enable(); enableBlink(); }
    }, [videoTrack]);

    // Store captured face image on blink
    useEffect(() => {
        if (recentBlinks.length === 0) return;
        const lastBlink = recentBlinks[recentBlinks.length - 1];
        if (lastBlink.faceImage) setLastFaceImage(lastBlink.faceImage);
    }, [recentBlinks]);

    const faceCount = detectionResult?.faces.length ?? 0;
    const firstFace = detectionResult?.faces[0];

    return (
        <SafeAreaView style={styles.container}>
            <Text style={styles.title}>Face Detection Demo</Text>

            {/* Video + Overlay (mirror and objectFit MUST match) */}
            <View style={styles.videoContainer}>
                {stream && (
                    <RTCView
                        streamURL={stream.toURL()}
                        style={styles.video}
                        mirror={true}
                        objectFit="cover"
                    />
                )}
                <FaceDetectionOverlay
                    detectionResult={detectionResult}
                    mirror={true}
                    objectFit="cover"
                    config={{
                        showFaceBox: true,
                        showEyeBoxes: true,
                        showMouthBox: true,
                        showHeadPose: true,
                        showEyeStatus: true,
                    }}
                    style={StyleSheet.absoluteFill}
                />
            </View>

            {/* Stats */}
            <View style={styles.statsPanel}>
                <Text style={styles.statText}>
                    Faces: {faceCount} | Detection: {isEnabled ? 'ON' : 'OFF'}
                </Text>
                {firstFace && (
                    <>
                        <Text style={styles.statText}>
                            L: {firstFace.landmarks.leftEye.isOpen ? 'Open' : 'Closed'}
                            {' '}({(firstFace.landmarks.leftEye.openProbability * 100).toFixed(0)}%)
                            {'  '}
                            R: {firstFace.landmarks.rightEye.isOpen ? 'Open' : 'Closed'}
                            {' '}({(firstFace.landmarks.rightEye.openProbability * 100).toFixed(0)}%)
                        </Text>
                        {firstFace.headPose && (
                            <Text style={styles.statText}>
                                Yaw: {firstFace.headPose.yaw.toFixed(1)}°
                                {'  '}Pitch: {firstFace.headPose.pitch.toFixed(1)}°
                                {'  '}Roll: {firstFace.headPose.roll.toFixed(1)}°
                            </Text>
                        )}
                    </>
                )}
                <Text style={styles.statText}>
                    Blinks: {blinkCount} | Rate: {getBlinkRate().toFixed(1)}/min
                </Text>
                <TouchableOpacity onPress={resetCount} style={styles.resetButton}>
                    <Text style={styles.resetText}>Reset Count</Text>
                </TouchableOpacity>
            </View>

            {/* Last captured face */}
            {lastFaceImage && (
                <View style={styles.captureContainer}>
                    <Text style={styles.captureLabel}>Last Blink Capture:</Text>
                    <Image
                        source={{ uri: `data:image/jpeg;base64,${lastFaceImage}` }}
                        style={styles.capturedFace}
                    />
                </View>
            )}

            {error && <Text style={styles.errorText}>Error: {error.message}</Text>}
        </SafeAreaView>
    );
}

const styles = StyleSheet.create({
    container: { flex: 1, backgroundColor: '#000' },
    title: { color: '#fff', fontSize: 18, fontWeight: '600', textAlign: 'center', paddingVertical: 8 },
    videoContainer: { width: '100%', aspectRatio: 3 / 4, backgroundColor: '#111', overflow: 'hidden' },
    video: { flex: 1 },
    statsPanel: { padding: 12, backgroundColor: '#1a1a1a' },
    statText: { color: '#ccc', fontSize: 13, fontFamily: 'monospace', marginBottom: 4 },
    resetButton: { marginTop: 8, backgroundColor: '#333', paddingHorizontal: 12, paddingVertical: 6, borderRadius: 4, alignSelf: 'flex-start' },
    resetText: { color: '#fff', fontSize: 13 },
    captureContainer: { padding: 12, alignItems: 'center' },
    captureLabel: { color: '#888', fontSize: 12, marginBottom: 8 },
    capturedFace: { width: 120, height: 120, borderRadius: 8, borderWidth: 1, borderColor: '#333' },
    errorText: { color: '#ff4444', fontSize: 13, padding: 12 },
});
```
{% endraw %}

The overlay draws animated green boxes around faces, blue boxes on open eyes (red when closed), and shows head rotation angles -- all updating at ~10fps.

> **Important:** The `mirror` and `objectFit` props on `FaceDetectionOverlay` must match `RTCView`. The overlay maps ML Kit's frame coordinates to screen pixels using these values. If they don't match, the boxes will be offset.

---

## Detection Data Structure

Each processed frame produces a `FaceDetectionResult`:

```
FaceDetectionResult
  .faces[]
    Face
      .bounds             // { x, y, width, height } in frame pixels
      .landmarks
        .leftEye          // { position, isOpen, openProbability, blinkCount }
        .rightEye         // same shape
        .mouth?           // { position, width, height }
        .nose?            // { position }
      .confidence         // 0.0 to 1.0
      .trackingId?        // stable integer across frames for the same face
      .headPose?          // { yaw, pitch, roll } in degrees
  .timestamp              // ms since epoch
  .frameWidth / .frameHeight
```

`trackingId` lets you maintain per-face state in JS. Note: if a face leaves and re-enters the frame, it may get a new ID.

---

## Use Case Patterns

### Liveness Detection (KYC)

A photo can't blink. Verify the user blinks at least twice within a time window:

```typescript
const sessionStart = useRef(Date.now());
const isLive = blinkCount >= 2;
const elapsed = (Date.now() - sessionStart.current) / 1000;

if (elapsed > 10 && !isLive) {
    setStatus('liveness_failed');
}
if (isLive) {
    submitForVerification(lastFaceImage);
}
```

For stronger liveness: combine with head pose challenges ("turn your head left").

### Exam Proctoring -- Gaze Detection

Alert if the student looks away for more than 3 seconds:

```typescript
const [lookAwayStart, setLookAwayStart] = useState<number | null>(null);

useEffect(() => {
    const face = detectionResult?.faces[0];
    const lookingAway = face?.headPose
        ? Math.abs(face.headPose.yaw) > 20
        : false;

    if (lookingAway && !lookAwayStart) {
        setLookAwayStart(Date.now());
    } else if (!lookingAway) {
        setLookAwayStart(null);
    }

    if (lookAwayStart && Date.now() - lookAwayStart > 3000) {
        flagProctorAlert('student_looking_away');
    }
}, [detectionResult]);
```

### Fatigue / Attention Monitoring

Normal blink rate is ~12-20/min. Abnormal rates can signal issues:

```typescript
const rate = getBlinkRate();
const status = rate < 8 ? 'eye_strain_risk'
             : rate > 25 ? 'fatigue_detected'
             : 'normal';
```

### Blink-to-Interact (Accessibility)

Double-blink (two blinks within 500ms) as an input trigger:

```typescript
useEffect(() => {
    if (recentBlinks.length < 2) return;
    const last = recentBlinks[recentBlinks.length - 1];
    const prev = recentBlinks[recentBlinks.length - 2];

    if (last.timestamp - prev.timestamp < 500) {
        onDoubleBlink();
    }
}, [recentBlinks]);
```

### Telehealth Session Quality

Track whether the patient is present and close enough to the camera:

```typescript
const faceCount = detectionResult?.faces.length ?? 0;
const face = detectionResult?.faces[0];

if (faceCount === 0) {
    setSessionStatus('patient_absent');
}

if (face && detectionResult) {
    const faceArea = face.bounds.width * face.bounds.height;
    const frameArea = detectionResult.frameWidth * detectionResult.frameHeight;
    if (faceArea / frameArea < 0.10) {
        setSessionStatus('patient_too_far');
    }
}
```

---

## Performance Tuning

`frameSkipCount` is your primary dial:

| frameSkipCount | Detection Rate (30fps) | CPU     | Best For                  |
| -------------- | ---------------------- | ------- | ------------------------- |
| 1              | 30/sec                 | High    | Research / debugging      |
| 3 (default)    | 10/sec                 | Medium  | General use               |
| 5              | 6/sec                  | Low     | Blink detection, liveness |
| 10             | 3/sec                  | Minimal | Periodic presence checks  |

For blink detection, 6-10fps is plenty. The state machine doesn't need 30fps to work reliably.

**Pause when backgrounded** to save battery:

```typescript
useEffect(() => {
    const sub = AppState.addEventListener('change', state => {
        if (state === 'background') disable();
        if (state === 'active' && videoTrack) enable();
    });
    return () => sub.remove();
}, [videoTrack, disable, enable]);
```

When using `captureOnBlink`, keep `imageQuality` at 0.5-0.7 and `maxImageWidth` at 320-480 for faster encoding. If you don't need image capture, leave it off (the default).

---

## Troubleshooting

**"Face detection is disabled"** -- `configureWebRTC({ enableFaceDetection: true })` wasn't called before `enable()`. Call it at app startup.

**"Not supported for remote tracks"** -- Face detection runs in the local capture pipeline. It only works on tracks from `getUserMedia()`, not remote peer tracks.

**Blink count stays at zero** -- Lower `blinkThreshold` (try 0.2). Make sure you're using `useBlinkDetection`, not just `useFaceDetection`. Ensure the face is fully visible.

**Overlay boxes are offset** -- The `mirror` and `objectFit` props on `FaceDetectionOverlay` must match `RTCView` exactly.

**High battery drain** -- Increase `frameSkipCount` to 5+. Disable `captureOnBlink` if unused. Add `AppState` listeners to pause detection when backgrounded.

---

**Repository:** [github.com/arfuhad/react-native-webrtc-face-detection](https://github.com/arfuhad/react-native-webrtc-face-detection)

**Install:** `npm install react-native-webrtc-face-detection`
