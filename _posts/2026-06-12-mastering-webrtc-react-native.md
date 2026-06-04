---
layout: post
title: "Mastering WebRTC in React Native: Lessons from a Large-Scale App"
date: 2026-06-12 12:00:00 +0600
published: false
tags:
  - react-native
  - webrtc
  - mobile
  - video-streaming
description: "Implementing WebRTC in React Native presents unique challenges. Learn how to manage RTCPeerConnection, signaling, and background states effectively."
---

Integrating live video into a mobile application using WebRTC is a notoriously complex endeavor. While the web platform handles much of the complexity natively in the browser, mobile platforms introduce hardware encoding variations, aggressive battery management, and complex application lifecycles.

During the development of a recent large-scale real-time ecosystem, I learned several critical lessons about taming WebRTC within React Native. Here is how we managed connections, signaling, and stream handling.

## The Core Concept: Signaling

WebRTC is fundamentally peer-to-peer, but peers cannot connect without first knowing how to find each other over the public internet. This process is called **signaling**. 

In our architecture, we used a NestJS backend with Socket.io to act as the signaling server. The React Native client is responsible for creating an `RTCPeerConnection` and negotiating the connection through the socket.

### Encapsulating the Logic

One of the biggest mistakes you can make in React Native WebRTC is tangling the `RTCPeerConnection` logic with your UI components. If a component unmounts, your call drops. If it re-renders unnecessarily, you might accidentally renegotiate the stream.

To solve this, we abstracted the WebRTC logic into a dedicated service layer and exposed it via custom React hooks.

```typescript
// A simplified abstraction of our custom hook
import { RTCPeerConnection } from 'react-native-webrtc';

export function useWebRTC() {
  const peerConnectionRef = useRef<RTCPeerConnection | null>(null);

  const initializeConnection = useCallback((config) => {
    const pc = new RTCPeerConnection(config);
    
    // Handle ICE candidates
    pc.onicecandidate = (event) => {
      if (event.candidate) {
        signalingService.sendIceCandidate(event.candidate);
      }
    };

    // Handle remote streams
    pc.ontrack = (event) => {
      if (event.streams && event.streams[0]) {
        setRemoteStream(event.streams[0]);
      }
    };

    peerConnectionRef.current = pc;
    return pc;
  }, []);

  // ... other methods for createOffer, createAnswer, etc.
}
```

By keeping the `RTCPeerConnection` in a `useRef` (or a singleton service), it survives component re-renders, allowing the UI to react to connection state changes without interfering with the underlying media pipeline.

## The ICE Candidate Dance

Network Address Translation (NAT) and firewalls block direct peer-to-peer connections. WebRTC uses ICE (Interactive Connectivity Establishment) candidates to find the best route between peers.

In React Native, managing this asynchronous exchange is tricky because you might receive ICE candidates *before* the remote session description is fully set.

**The Solution:** Queueing.
If your React Native app receives a remote ICE candidate via the signaling WebSocket, but the `RTCPeerConnection` hasn't finished processing the `setRemoteDescription`, you must queue the candidate in memory. Once the remote description is successfully set, you drain the queue and add the candidates.

## Handling Mobile Network Volatility

Mobile devices jump between WiFi, 4G, and 5G constantly. They also enter tunnels and lose signal. The `RTCPeerConnection` state must be monitored aggressively.

```typescript
pc.onconnectionstatechange = () => {
  if (pc.connectionState === 'disconnected' || pc.connectionState === 'failed') {
    // Trigger UI updates, attempt ICE restart, or fallback to audio-only
    handleConnectionDrop();
  }
};
```

When a connection fails, relying solely on WebRTC's internal reconnection logic isn't always enough on mobile. We implemented a signaling-layer heartbeat over Socket.io. If the WebSocket heartbeat fails, the app knows it has lost total connectivity and can gracefully freeze the UI rather than leaving the user in an ambiguous state.

## App Backgrounding

When a user backgrounds a React Native app on iOS, the OS aggressively suspends execution. If you are in a WebRTC call, the camera feed will usually be paused by the OS for privacy reasons, but the audio can continue if the proper background modes are configured in Xcode.

We ensured that:
1. **Camera state is handled:** When `AppState.currentState` goes to `background`, we temporarily disable the local video track so the remote peer sees a black screen rather than a frozen frame.
2. **Audio persists:** Proper audio session configuration ensures the microphone and speaker continue to function.

## Conclusion

WebRTC in React Native bridges the gap between web APIs and native performance. By strictly separating connection logic from the UI, properly queuing ICE candidates, and aggressively managing the mobile app lifecycle, you can build incredibly robust real-time video experiences.
