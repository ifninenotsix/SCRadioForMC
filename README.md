# SoundCloud Radio

[Русская версия](README_RU.md)

A Minecraft Forge mod that adds a **Radio** item. Right-click it to open the
in-game player, paste a link to a SoundCloud track or playlist, and everyone
nearby hears the music. Sound fades with distance and pans left/right depending
on where the radio holder is standing.

No external music player is required: AAC and MP3 decoding is bundled into the
mod jar.

## Features

- Play public SoundCloud tracks and playlists directly from the game.
- Scrollable track list with search by title or track number; click a row to
  switch tracks. The current track is highlighted.
- Large playlists are supported. A playlist with **482 tracks** was tested;
  the mod fetches the full list instead of stopping at the short preview
  SoundCloud sometimes returns.
- Positional audio: distance attenuation and stereo panning around the player
  holding the radio.
- Volume and bass boost are broadcast by the radio holder, so nearby players
  hear the same mix. Each listener keeps their own values in config.
- Three-band equalizer (low / mid / high) is applied locally by each listener.
- Pure-Java DSP: bass boost and EQ are processed in PCM before output, so no
  native sound-card controls are required.
- The server never downloads audio and never receives signed CDN stream URLs.
  Each client resolves and decodes its own stream.

## Requirements

- Minecraft **1.19.2**
- Forge **43.x** (`forge_version_range = [43,)` in this project)
- Java **17** to build and run
- Internet access on the client for SoundCloud requests

The Gradle wrapper downloads Gradle automatically; you do not need a local
Gradle installation.

## Installation

1. Install Forge 43.x for Minecraft 1.19.2.
2. Download the release jar, or build it yourself (see below).
3. Put **`soundcloudradio-1.19.2-1.0.0-all.jar`** into the `mods` folder.
   Use the `-all.jar`: the plain jar does not include the AAC/MP3 decoders.
4. For multiplayer, install the same jar on the server too. The server uses the
   mod to relay playback state and to verify that the player still holds a
   radio. In single-player the integrated server loads it automatically.

> Clients and server must use the same jar version. The network protocol
> version is currently `2`.

## Building

Windows:

```powershell
.\gradlew.bat build
```

Linux / macOS:

```bash
./gradlew build
```

Output:

```text
build/libs/soundcloudradio-1.19.2-1.0.0-all.jar
```

For development:

```powershell
.\gradlew.bat runClient
.\gradlew.bat runServer
```

The build is checked with JDK 17 and Gradle 8.1.1. Real AAC-HLS playback was
verified at 44.1 kHz / 16-bit / stereo.

## Usage

1. Get the radio:
   - creative tab **Misc**, or
   - `/give @p soundcloudradio:radio`
2. Right-click the radio to open the player.
3. The GUI is pre-filled with the default playlist from config. Replace it with
   any public SoundCloud track or playlist URL and press **Load**.
4. Use the track list:
   - type into **Search tracks** to filter by title or number;
   - click a track to play it;
   - the current track is highlighted.
5. Controls: **Play/Pause**, **Stop**, **Prev**, **Next**. Tracks advance
   automatically when `autoNext` is enabled.
6. Sliders:
   - **Volume** and **Bass boost** affect everyone listening to your radio;
   - **EQ Low / Mid / High** affect only your own output.

All slider values are saved when the GUI closes.

## Configuration

The file `config/soundcloudradio-client.toml` is created after the first run.

| Option | Default | Description |
|---|---|---|
| `playlistUrl` | built-in demo playlist | URL pre-filled in the GUI |
| `clientId` | `""` | Manual SoundCloud `client_id`; empty = scrape it from the website |
| `volume` | `80` | Radio volume, `0..100` (%) |
| `bassBoost` | `0` | Bass boost, `0..100` (`0` = off, `100` ≈ +15 dB at 100 Hz) |
| `eqLow` | `0` | Low band, 250 Hz shelf, `-12..+12` dB |
| `eqMid` | `0` | Mid band, 1 kHz peak, `-12..+12` dB |
| `eqHigh` | `0` | High band, 4 kHz shelf, `-12..+12` dB |
| `radius` | `48.0` | Audible radius in blocks around the radio holder |
| `autoNext` | `true` | Automatically play the next track when the current one ends |

If automatic `client_id` detection breaks after a SoundCloud website change,
paste a current `client_id` into `clientId` manually.

## How it works

- `api-v2.soundcloud.com/resolve` converts a page URL into a track or playlist.
  For playlists the mod additionally requests `/playlists/{id}` to get the full
  track list. Tracks that SoundCloud returns only as `{ id, kind, policy }`
  stubs are enriched through `/tracks/{id}` in a small thread pool. This is why
  playlists with 150+ tracks work. The first load of a very large playlist may
  take a few seconds.
- The `client_id` is scraped from SoundCloud's JS bundles and cached for 12
  hours.
- SoundCloud mostly serves AAC-HLS / fMP4 for public tracks. The mod uses JAAD
  plus a small `moof/trun/mdat` demuxer. Progressive MP3 streams fall back to
  mp3spi/JLayer/Tritonus.
- Audio is converted to `44100 Hz / 16-bit / stereo / little-endian` PCM. Bass
  boost and EQ are RBJ biquad filters applied in Java right before the samples
  are written to the `SourceDataLine`.
- The server receives only the SoundCloud page URL, title, track index,
  position/entity id, owner volume and bass boost — never the CDN audio URL.
- Positional sound is implemented with Java Sound `MASTER_GAIN` and `PAN`. This
  is not Minecraft's native sound engine: there is no wall occlusion or echo.

## Limitations

- Only public SoundCloud tracks can be streamed. Private, geo-blocked, or
  removed tracks will not produce a stream.
- Supported codecs: AAC-HLS and MP3. Opus/FLAC are not supported.
- The mod uses SoundCloud's internal API (`api-v2`), not an official public
  API. Website changes may temporarily break resolution.
- The person playing a track is responsible for the content they play.
- This project is not affiliated with or endorsed by SoundCloud.

## Troubleshooting

| Symptom | What to check |
|---|---|
| No sound at all | Check volume, distance/`radius`, and that your client can reach SoundCloud |
| Only a few tracks load | Use a freshly built `-all.jar`; old jars stopped at the API preview |
| "Could not find a SoundCloud client_id" | Set `clientId` manually in the config |
| Crash mentioning `junit.awtui` or module resolution | You are using an old jar; current builds exclude JUnit/Hamcrest from the shaded jar |
| Multiplayer does not react | The mod must also be installed on the server, with the same protocol version |
| Large playlist takes time to appear | Expected: metadata for id-only track stubs is fetched during Load |

## Contributing

Issues and pull requests are welcome. Please keep Java 17 compatibility, avoid
adding native dependencies, and run `./gradlew build` before opening a PR.

## License

Mod source code is released into the public domain under **CC0 1.0 Universal**.
See [LICENSE](LICENSE).

Bundled third-party libraries keep their own licenses:

- JAAD — public domain
- MP3SPI — LGPL 2.1+ / JavaZoom license
- JLayer — LGPL 2.1+
- Tritonus Share — LGPL 2.1+
