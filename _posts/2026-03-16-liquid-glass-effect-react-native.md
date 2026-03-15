---
layout: post
title: "Apple's Liquid Glass Effect for React Native — on Both Platforms"
date: 2026-03-16 12:00:00 +0600
published: true
tags:
  - react-native
  - liquid-glass
  - glassmorphism
  - webview
  - svg-filters
description: "Bring Apple's liquid glass aesthetic to Android and older iOS with a WebView-based SVG filter pipeline — no native modules required."
---

Apple introduced liquid glass in iOS 26, but it's locked to the latest SDK and iOS only. If your app targets Android or supports older iOS versions, you're out of luck. Building a convincing glass effect from scratch means wrangling SVG displacement maps, chromatic aberration, specular highlights, and backdrop blur — then figuring out how to render all of it inside React Native.

[`expo-rn-liquid-glass-view`](https://github.com/arfuhad/expo-rn-liquid-glass-view) runs the entire glass effect inside a transparent WebView. No native module linking. Works on Android API 21+ and iOS 13+. Drop it into any Expo or bare React Native project.

---

## What You Get

| Capability | Standard RN Views | This Package |
| --- | --- | --- |
| Backdrop blur | Via native blur libs | Yes (CSS `backdrop-filter`) |
| **Displacement refraction** | No | Yes (SVG `feDisplacementMap`) |
| **Chromatic aberration** | No | Yes (per-channel displacement) |
| **Specular border highlights** | No | Yes (screen/overlay blend) |
| **Background content refraction** | No | Yes (view-shot capture) |
| **Adaptive performance** | No | Yes (FPS-based quality tiers) |
| **Works on Android** | N/A | Yes |

---

## Quick Setup

```bash
npm install expo-rn-liquid-glass-view react-native-webview
```

Optionally, for background refraction:

```bash
npm install react-native-view-shot
```

No native linking, no config plugins. The WebView handles everything.

---

## How the Rendering Pipeline Works

The effect runs entirely inside a transparent WebView overlaid on your React Native content. Here's what happens inside:

```
LiquidGlassView (React Native)
    |
    +-- measures layout via onLayout
    +-- manages background capture (react-native-view-shot)
    +-- renders WebViewRenderer
          |
          +-- generates HTML document with:
          |     +-- SVG filter pipeline
          |     +-- CSS backdrop-filter (blur + saturate)
          |     +-- Specular border highlights
          |     +-- JS bridge for prop updates
          |
          +-- renders <WebView source={{ html }} />
          +-- communicates via injectJavaScript / onMessage
```

### The SVG Filter Chain

This is the core of the visual effect. Five stages run in sequence inside a single SVG `<filter>`:

**1. Displacement map loading** — An `feImage` loads a pre-baked radial gradient (or a runtime-generated SDF canvas for `shader` mode). This map defines *how* the backdrop warps.

**2. Per-channel displacement** — Three separate `feDisplacementMap` operations warp the R, G, and B channels at slightly different scales. The green channel displaces 5% less than red, blue 10% less. This creates chromatic aberration — the color fringing you see at glass edges.

```xml
<!-- Red channel -->
<feDisplacementMap in="SourceGraphic" in2="DISPLACEMENT_MAP"
  scale="70" xChannelSelector="R" yChannelSelector="B" />

<!-- Green channel (slightly less displacement) -->
<feDisplacementMap in="SourceGraphic" in2="DISPLACEMENT_MAP"
  scale="66.5" xChannelSelector="R" yChannelSelector="B" />

<!-- Blue channel (even less) -->
<feDisplacementMap in="SourceGraphic" in2="DISPLACEMENT_MAP"
  scale="63" xChannelSelector="R" yChannelSelector="B" />
```

**3. Edge masking** — A radial gradient mask ensures aberration only appears at the edges. The center stays clean. Without this, the entire glass surface would show color fringing, which looks wrong.

**4. CSS `backdrop-filter`** — `blur()` + `saturate()` applied to the warp layer. This is what makes the background content behind the glass look frosted.

**5. Specular borders** — Dual-layer borders with `mix-blend-mode: screen` and `overlay` create the characteristic light refraction along glass edges. In `overLight` mode, additional shadow layers appear for use against bright backgrounds.

### Displacement Modes

Four algorithms for generating the warp map:

| Mode | How It Works | Best For |
| --- | --- | --- |
| `standard` | Pre-baked radial gradient SVG | General use, fast |
| `polar` | Polar coordinate variant | Different edge character |
| `prominent` | Stronger edge displacement | Dramatic, pronounced warp |
| `shader` | Runtime SDF via `<canvas>` | Shape-accurate warp |

---

## Basic Usage

{% raw %}
```tsx
import { LiquidGlassView } from 'expo-rn-liquid-glass-view';

function GlassCard() {
  return (
    <LiquidGlassView
      style={{ width: 300, height: 200 }}
      cornerRadius={20}
      blurAmount={0.0625}
      displacementScale={70}
      aberrationIntensity={2}
      onReady={() => console.log('Glass ready!')}
    >
      <Text style={{ color: '#fff', fontSize: 24 }}>Hello Glass</Text>
    </LiquidGlassView>
  );
}
```
{% endraw %}

The glass is invisible until the WebView finishes rendering the SVG pipeline. Use `onReady` to fade it in:

{% raw %}
```tsx
const [ready, setReady] = useState(false);

<LiquidGlassView
  style={{ opacity: ready ? 1 : 0 }}
  onReady={() => setReady(true)}
/>
```
{% endraw %}

---

## Background Refraction

The WebView's `backdrop-filter` only blurs content inside its own DOM. To refract *native* React Native content — your actual app UI — you need background capture.

Point a `backgroundRef` at the view behind the glass, and set a capture mode:

{% raw %}
```tsx
import { useRef } from 'react';
import { View, Image } from 'react-native';
import { LiquidGlassView } from 'expo-rn-liquid-glass-view';

function RefractionDemo() {
  const backgroundRef = useRef<View>(null);

  return (
    <View style={{ flex: 1 }}>
      {/* Content to refract */}
      <View ref={backgroundRef} collapsable={false} style={{ flex: 1 }}>
        <Image source={require('./bg.jpg')} style={{ flex: 1 }} />
      </View>

      {/* Glass overlay */}
      <LiquidGlassView
        style={{ position: 'absolute', top: 100, left: 30, width: 300, height: 200 }}
        backgroundRef={backgroundRef}
        captureMode="static"
        cornerRadius={20}
      >
        <Text style={{ color: '#fff' }}>Refracted content</Text>
      </LiquidGlassView>
    </View>
  );
}
```
{% endraw %}

Under the hood, `react-native-view-shot` captures the background view as a base64 image. That image is injected into the WebView's `<body>` background via `injectJavaScript`. Each glass instance uses `measureInWindow` to compute its offset, then sets `background-position` so the WebView shows the correct region.

### Capture Modes

| Mode | Behavior | Use Case |
| --- | --- | --- |
| `none` | No capture, uses fallback gradient | Decorative glass, no refraction needed |
| `static` | Capture once on mount + layout changes | Static backgrounds |
| `periodic` | Capture every N ms (`captureInterval`) | Slowly changing content |
| `realtime` | Self-chaining capture loop | Animations, scrolling content |
| `manual` | Only via `ref.current.capture()` | User-triggered updates |

### Shared Capture Deduplication

When multiple glass views share the same `backgroundRef`, only one capture loop runs. Results are shared via a module-level `WeakMap` registry keyed by the ref. This prevents concurrent `captureRef` calls that would cause "No view found with reactTag" errors.

{% raw %}
```tsx
const backgroundRef = useRef<View>(null);

{/* Both share one capture loop */}
<LiquidGlassView backgroundRef={backgroundRef} captureMode="periodic" ... />
<LiquidGlassView backgroundRef={backgroundRef} captureMode="periodic" ... />
```
{% endraw %}

---

## Tint Overlays

Apply solid colors or CSS gradients over the glass surface. The tint sits above the blurred backdrop but below specular highlights:

{% raw %}
```tsx
{/* Solid color tint */}
<LiquidGlassView tintColor="rgba(255, 100, 50, 0.2)" ... />

{/* Gradient tint */}
<LiquidGlassView
  tintGradient="linear-gradient(180deg, rgba(253,151,61,0.1), rgba(255,115,0,0.9))"
  ...
/>
```
{% endraw %}

`tintGradient` takes priority over `tintColor` when both are set.

---

## Performance Monitoring

The WebView reports FPS via `onMessage`. The `usePerformanceMonitor` hook consumes this data and manages quality tiers:

{% raw %}
```tsx
import { LiquidGlassView, usePerformanceMonitor } from 'expo-rn-liquid-glass-view';

function AdaptiveGlass() {
  const { adjustedProps, handlePerformanceReport } = usePerformanceMonitor({
    mode: 'auto',
    downgradeThreshold: 24,
    upgradeThreshold: 50,
  });

  return (
    <LiquidGlassView
      style={{ width: 300, height: 200 }}
      displacementScale={70}
      aberrationIntensity={2}
      onPerformanceReport={handlePerformanceReport}
      {...adjustedProps}
    />
  );
}
```
{% endraw %}

### Quality Tiers

In `auto` mode, the hook automatically steps down (or back up) based on sustained FPS:

| Tier | What Changes |
| --- | --- |
| `high` | No overrides — full quality |
| `medium` | Removes chromatic aberration (biggest GPU lever) |
| `low` | Reduced displacement + blur, no aberration |
| `minimal` | Near-zero effects, force `standard` mode |

Three consecutive bad samples (~6s) trigger a downgrade. Five consecutive good samples (~10s) trigger an upgrade. These thresholds are configurable.

For manual control:

```tsx
const { setTier } = usePerformanceMonitor({ mode: 'manual' });

<Button title="Low Quality" onPress={() => setTier('low')} />
```

---

## Props Reference

| Prop | Type | Default | Description |
| --- | --- | --- | --- |
| `displacementScale` | `number` | `70` | Refraction intensity (0–200) |
| `blurAmount` | `number` | `0.0625` | Backdrop blur strength (0–1) |
| `saturation` | `number` | `140` | Backdrop color saturation (%) |
| `aberrationIntensity` | `number` | `2` | Chromatic aberration at edges (0–10) |
| `cornerRadius` | `number` | `20` | Glass shape corner radius (px) |
| `overLight` | `boolean` | `false` | Light background mode |
| `tintColor` | `string` | — | Solid CSS color tint |
| `tintGradient` | `string` | — | CSS gradient tint (overrides tintColor) |
| `mode` | `DisplacementMode` | `'standard'` | Displacement map algorithm |
| `backgroundRef` | `RefObject<View>` | — | View to capture for refraction |
| `captureMode` | `CaptureMode` | `'none'` | Background capture strategy |
| `captureInterval` | `number` | `500` | Capture interval in ms |
| `onReady` | `() => void` | — | Fires when glass is rendered |
| `onPerformanceReport` | `(fps) => void` | — | FPS data callback |

---

## Troubleshooting

**Android shows a white/blank background** — Set `collapsable={false}` on the `backgroundRef` view. Without this, React Native optimizes away the native backing view and `react-native-view-shot` fails silently.

**Background refraction shows the wrong region** — The `backgroundRef` view must cover the full area behind all glass views. Each instance computes its offset via `measureInWindow` — if the background doesn't start at the expected position, the math breaks.

**WebView flickers on mount** — Expected behavior. The SVG pipeline needs a moment to render. Use `onReady` to fade in.

**High battery drain** — Switch `captureMode` from `realtime` to `periodic` or `static`. Use `usePerformanceMonitor` in `auto` mode to reduce effect quality on slower devices.

---

## Compatibility

| Requirement | Version |
| --- | --- |
| React Native | >= 0.72.0 |
| Expo | SDK 49+ |
| iOS | 13.0+ |
| Android | API 21+ |
| react-native-webview | >= 13.0.0 |
| react-native-view-shot | >= 3.0.0 (optional) |

No native modules. No config plugins. Works with both bare React Native and Expo managed workflows.

---

**Repository:** [github.com/arfuhad/expo-rn-liquid-glass-view](https://github.com/arfuhad/expo-rn-liquid-glass-view)

**Install:** `npm install expo-rn-liquid-glass-view`
