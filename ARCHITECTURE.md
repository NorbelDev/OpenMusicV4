# OpenMusic V4 — Phase 1 Architecture Document

## Project Overview

OpenMusic V4 is a production-grade YouTube Music streaming client built from scratch using the SimpMusic repository as the absolute technical reference. It communicates directly with YouTube's InnerTube API — no third-party proxies.

## Architecture: Clean Architecture + MVVM

```
┌─────────────────────────────────────────────────────────┐
│                     UI LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │ SearchScreen  │  │  MiniPlayer  │  │ FullPlayer    │ │
│  │ (Compose)     │  │  (Compose)   │  │ (Compose)     │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬────────┘ │
│         │                  │                  │          │
│  ┌──────┴──────────────────┴──────────────────┴────────┐│
│  │              SearchViewModel (Hilt)                  ││
│  │         PlaybackController (Singleton)               ││
│  └──────────────────────┬──────────────────────────────┘│
├─────────────────────────┼───────────────────────────────┤
│                  DOMAIN LAYER                            │
│  ┌──────────────────────┴──────────────────────────────┐│
│  │         MusicRepository (Interface)                  ││
│  │    SearchResultItem, NowPlaying, PlayerState         ││
│  └──────────────────────┬──────────────────────────────┘│
├─────────────────────────┼───────────────────────────────┤
│                   DATA LAYER                             │
│  ┌──────────────────────┴──────────────────────────────┐│
│  │         MusicRepositoryImpl                          ││
│  │    InnerTubeApi    StreamExtractor                   ││
│  │    InnerTubeModels    PlayerResponse                 ││
│  └─────────────┬────────────────┬──────────────────────┘│
│                │                │                        │
│         ┌──────┴──────┐  ┌─────┴───────────┐           │
│         │ OkHttp      │  │  PlaybackService │           │
│         │ (Network)   │  │  (Media3)        │           │
│         └─────────────┘  └─────────────────┘           │
└─────────────────────────────────────────────────────────┘
```

## File Structure

```
app/src/main/java/com/openmusic/v4/
├── OpenMusicApp.kt              # Application class (Hilt entry point)
├── MainActivity.kt              # Single Activity, hosts Compose UI
├── di/
│   └── AppModule.kt             # Hilt dependency injection modules
├── data/
│   ├── api/
│   │   ├── InnerTubeApi.kt      # Direct YouTube Music API client
│   │   └── StreamExtractor.kt   # Audio stream URL extraction
│   ├── models/
│   │   ├── InnerTubeModels.kt   # Search response data classes
│   │   └── PlayerResponse.kt    # Player response data classes
│   └── repository/
│       └── MusicRepositoryImpl.kt # Transforms API data → domain models
├── domain/
│   ├── model/
│   │   └── SearchResultItem.kt  # Clean domain models for UI
│   └── repository/
│       └── MusicRepository.kt   # Repository interface
├── service/
│   ├── PlaybackService.kt       # Media3 MediaSessionService
│   └── PlaybackController.kt    # Bridge between UI and service
└── ui/
    ├── theme/
    │   └── Theme.kt             # Material 3 dark theme
    ├── viewmodel/
    │   └── SearchViewModel.kt   # ViewModel with debounced search
    ├── screens/
    │   ├── search/
    │   │   └── SearchScreen.kt  # Search UI with LazyColumn
    │   └── player/
    │       └── FullPlayerScreen.kt # Now Playing screen
    └── components/
        └── MiniPlayer.kt        # Persistent bottom player bar
```

---

## 1. InnerTube API Integration (InnerTubeApi.kt)

### How It Works

YouTube Music's web client at `music.youtube.com` communicates with Google's servers using an internal API called **InnerTube**. Every user action (search, play, browse) sends a POST request to endpoints under `https://music.youtube.com/youtubei/v1/`.

Each request contains a JSON body with a `context` block that identifies the client. YouTube uses this to determine what data/formats to return.

### Client Identities

| Client | Name | Use Case |
|--------|------|----------|
| **WEB_REMIX** | YouTube Music Web | Search, Browse, Next |
| **ANDROID_MUSIC** | YouTube Music Android | Player (direct URLs) |
| **TVHTML5_SIMPLY_EMBEDDED** | TV Embed Player | Fallback (age-gate bypass) |

### Search Request

```
POST https://music.youtube.com/youtubei/v1/search?key=AIzaSyC9XL3ZjWddXya6X74dJoCTL-WEYFDNX30

Headers:
  Content-Type: application/json
  User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...
  Referer: https://music.youtube.com/
  Origin: https://music.youtube.com
  X-Goog-Visitor-Id: CgtsZG1ySnZiQWtSbyiMjuGSBg%3D%3D

Body:
{
  "context": {
    "client": {
      "clientName": "WEB_REMIX",
      "clientVersion": "1.20241111.01.00",
      "hl": "en",
      "gl": "US"
    }
  },
  "query": "Bohemian Rhapsody",
  "params": "Eg-KAQwIARAAGAAgACgAMABqChAEEAMQCRAFEAo%3D"  // Songs filter
}
```

### Player Request

```
POST https://music.youtube.com/youtubei/v1/player?key=AIzaSyC9XL3ZjWddXya6X74dJoCTL-WEYFDNX30

Headers:
  Content-Type: application/json
  User-Agent: com.google.android.apps.youtube.music/7.27.52 ...

Body:
{
  "context": {
    "client": {
      "clientName": "ANDROID_MUSIC",
      "clientVersion": "7.27.52",
      "androidSdkVersion": 34,
      "osVersion": "14",
      "osName": "Android"
    }
  },
  "videoId": "fJ9rUzIMcZQ",
  "contentCheckOk": true,
  "racyCheckOk": true
}
```

---

## 2. Stream Extraction Logic (StreamExtractor.kt)

### The Problem

Getting a videoId is easy. Getting a *playable audio URL* from that videoId is the hard part, because YouTube employs multiple anti-bot layers:

### Layer 1: Signature Cipher

Some formats return `signatureCipher` instead of a direct `url`. This is an encrypted URL that requires:
1. Fetching YouTube's `player.js` (changes frequently)
2. Extracting the cipher function chain
3. Applying JavaScript string transformations to the signature
4. Appending the decoded signature to the base URL

**Our solution:** Use the `ANDROID_MUSIC` client identity, which usually returns direct `url` fields without cipher. This is the same approach SimpMusic uses as their primary strategy.

### Layer 2: N-Parameter Throttling

Even with a valid URL, YouTube adds an `n` query parameter. Without properly transforming it, download speeds are throttled to ~50KB/s.

The transform function is obfuscated in `player.js` and changes every few weeks.

**Our solution:** The ANDROID_MUSIC client has different throttling behavior. For Phase 1, this is sufficient. Phase 2 will integrate NewPipe's extractor for full `n`-parameter handling (matching SimpMusic's `newPipePlayer()` method).

### Layer 3: PoToken / BotGuard

YouTube's newest protection requires a "proof of origin" token for certain requests.

**Our solution:** The ANDROID_MUSIC client identity doesn't require PoToken for most tracks. SimpMusic handles this via their `ghostRequest()` method for edge cases.

### Audio Format Selection

```
Priority Order:
1. itag 140 — M4A AAC 128kbps  ← Best ExoPlayer compatibility
2. itag 251 — WebM Opus 160kbps ← Higher quality
3. itag 250 — WebM Opus 70kbps  ← Medium fallback
4. itag 249 — WebM Opus 50kbps  ← Last resort
5. Any audio format, sorted by bitrate descending
```

---

## 3. Media3 Background Service (PlaybackService.kt)

### Why MediaSessionService?

Android aggressively kills background processes. A plain `Service` with `startForeground()` is not enough — you need `MediaSessionService` because:

1. **Automatic notification management** — Media3 creates and updates the notification with play/pause/skip controls and artwork
2. **Audio focus handling** — Automatically pauses for phone calls, ducks for navigation
3. **Media button support** — Headphone play/pause/skip buttons work automatically
4. **Android Auto integration** — Works with car head units
5. **System media controls** — Shows up in Android 11+ media controls panel
6. **Process priority** — Android gives foreground media services higher priority than other background processes

### Configuration Details

```kotlin
ExoPlayer.Builder(this)
    .setAudioAttributes(
        AudioAttributes.Builder()
            .setContentType(C.AUDIO_CONTENT_TYPE_MUSIC)  // Tells Android this is music
            .setUsage(C.USAGE_MEDIA)                      // Media playback usage
            .build(),
        handleAudioFocus = true  // ExoPlayer handles focus automatically
    )
    .setHandleAudioBecomingNoisy(true)  // Pause when headphones unplugged
    .setWakeMode(C.WAKE_MODE_NETWORK)  // Keep CPU + WiFi during playback
    .build()
```

### AndroidManifest Requirements

```xml
<!-- Permission -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />

<!-- Service Declaration -->
<service
    android:name=".service.PlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="true">
    <intent-filter>
        <action android:name="androidx.media3.session.MediaSessionService" />
    </intent-filter>
</service>
```

---

## 4. Dependency Versions (May 2026)

| Library | Version | Purpose |
|---------|---------|---------|
| Kotlin | 2.1.0 | Language |
| Compose BOM | 2024.12.01 | UI Framework |
| Material3 | (via BOM) | Design System |
| Media3 | 1.5.1 | ExoPlayer + MediaSession |
| OkHttp | 4.12.0 | HTTP Client |
| Kotlinx Serialization | 1.7.3 | JSON Parsing |
| Coil 3 | 3.0.4 | Image Loading |
| Hilt | 2.53.1 | Dependency Injection |
| Navigation Compose | 2.8.5 | Navigation |

---

## 5. Phase 2 Roadmap

- [ ] Full signature cipher decoding (integrate NewPipe extractor)
- [ ] N-parameter transform function
- [ ] PoToken / BotGuard handling
- [ ] Room database for offline caching
- [ ] Queue management (add to queue, reorder)
- [ ] Album / Artist / Playlist browsing
- [ ] Lyrics support (LRCLIB integration)
- [ ] Download for offline playback
- [ ] Android Auto support
- [ ] Equalizer
- [ ] Sleep timer
