# 🎵 OpenMusic V4

A production-grade YouTube Music streaming client built with Kotlin, Jetpack Compose, and Media3. Communicates directly with YouTube's InnerTube API — no third-party proxies.

## Quick Start

1. Open the project in Android Studio Ladybug (2024.2+)
2. Sync Gradle
3. Build and run on a device/emulator (API 26+)
4. Search for any song and tap to play

## Tech Stack

- **Language:** 100% Kotlin
- **UI:** Jetpack Compose + Material 3
- **Media:** AndroidX Media3 (ExoPlayer) with MediaSessionService
- **Network:** OkHttp + Kotlin Serialization
- **Architecture:** MVVM + Clean Architecture (Data/Domain/UI layers)
- **DI:** Hilt
- **Images:** Coil 3

## Key Features (Phase 1)

- ✅ Direct InnerTube API integration (no Piped/Invidious)
- ✅ Background playback via MediaSessionService
- ✅ Notification controls with artwork
- ✅ Audio focus handling (auto-pause for calls)
- ✅ Real-time debounced search
- ✅ Spotify-inspired dark UI
- ✅ Full-screen Now Playing with seekbar
- ✅ High-res artwork loading via Coil

## License

Educational/personal use only. See ARCHITECTURE.md for full details.
