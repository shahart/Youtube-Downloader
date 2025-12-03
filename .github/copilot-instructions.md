# Copilot Instructions for YouTube-Downloader

## Project Architecture

**YouTube-Downloader** is a gRPC-based microservice that wraps YT-DLP command-line tool for downloading YouTube videos and audio. The project uses **Maven** (primary) with Gradle fallback, Java 19+, Protocol Buffers 3.23.1, and gRPC 1.55.1.

### Core Architecture Flow
```
Client (Flutter GUI) 
  ↓ (gRPC + TLS)
DownloaderServer (port 8082)
  ↓
DownloaderService (gRPC service impl)
  ↓
Downloader (command executor)
  ↓
yt-dlp (shell command execution)
  ↓
Downloaded file (MP3/MP4)
```

### Key Components

- **`DownloaderServer`** - Main entry point; sets up gRPC server with TLS/SSL on port 8082 (configurable in `Globals.java`). Uses Netty-based server with mutual auth support.
- **`DownloaderService`** - Implements gRPC service stub; handles `DownloaderRequest` proto messages, calls `Downloader.executeDownloadRequest()`, returns `DownloaderResponse`.
- **`Downloader`** - Core business logic; executes shell commands via `FileUtils.runScript()`. Does NOT handle individual audio/video downloads directly—uses `CommandBuilder` to construct yt-dlp commands.
- **`CommandBuilder`** - Generates yt-dlp shell commands from `DownloaderConfig` proto messages. Handles playlist detection, format selection, resolution, metadata embedding.
- **`AutoChannelClient`** - gRPC client stub for connecting to server; loads TLS credentials from classpath (`resources/cert/`).
- **`FileUtils`** - Windows-specific cmd.exe wrapper using `ProcessBuilder`; executes shell commands, captures output, waits for exit code.

### Protocol Buffers Schema
Defined in `src/main/proto/downloaderConfig.proto`:
- **`DownloaderRequest`** - Wraps `DownloaderConfig`
- **`DownloaderResponse`** - Returns config + error/status fields
- **`DownloaderConfig`** - Contains link, path, downloadType (Audio/Video), outputFormat, resolution, isPlaylist, retries, embedSubtitles, embedThumbnail

## Build & Dependencies

**Build Tools**: Maven (primary), Gradle (secondary)  
**Java Target**: Java 8 (Gradle), Java 19 (Maven) - Maven is canonical.  
**Build Command**: `mvn clean package` generates fat JAR with main class `org.base.DownloaderServer`

**Critical Dependencies**:
- `protobuf-java:3.23.1` - Proto runtime
- `grpc-all:1.55.1` - gRPC server/client + Netty transport
- `log4j-core:2.14.1` - Logging
- `jackson-annotations:2.13.0` - JSON support

**Proto Compilation**: Maven plugin `protobuf-maven-plugin` auto-generates Java stubs in `target/generated-sources/protobuf/`. gRPC plugin generates service stubs via `compile-custom` goal.

## Developer Workflows

### Local Development
1. **Install yt-dlp**: Add to Windows PATH. Project expects `yt-dlp` command available system-wide.
2. **Generate Proto Stubs**: `mvn clean compile` (auto-runs protobuf plugin)
3. **Build Fat JAR**: `mvn clean package` → `target/youtubeDownload-1.0-SNAPSHOT-jar-with-dependencies.jar`
4. **Run Server**: `java -cp target/youtubeDownload-1.0-SNAPSHOT-jar-with-dependencies.jar org.base.DownloaderServer` (loads certs from classpath)

### TLS Certificate Setup

**Certificate Files Required** (placed in `src/main/resources/cert/`):
- `server-cert.pem` - Server certificate (X.509)
- `server-key.pem` - Server private key (PKCS#8)
- `ca-cert.pem` - Certificate Authority certificate (used by client to verify server)
- `client-cert.pem` - Client certificate for mutual TLS
- `client-key.pem` - Client private key for mutual TLS

**Server-Side Loading** (`DownloaderServer.loadTLSCredentials()`):
1. Reads cert files from classpath as `InputStream` (not filesystem directly)
2. Converts each stream to temp file via `streamToFile()` (writes to JVM temp directory)
3. Builds `SslContext` via `SslContextBuilder.forServer(certFile, keyFile)` with `ClientAuth.NONE`
4. Temp files auto-deleted on JVM exit via `deleteOnExit()`
5. Config passed to `NettyServerBuilder.sslContext()` during server initialization

**Client-Side Loading** (`AutoChannelClient.loadTLScredentials()`):
1. Similar classpath stream → temp file pattern
2. Uses `GrpcSslContexts.forClient()` for client-side context
3. Sets key manager (client cert + key) and trust manager (CA cert)
4. Passed to `NettyChannelBuilder.sslContext()` when creating managed channel
5. Note: Method name has typo "credentialS" (should be "credentials")

**Important**: Do NOT use file-based cert paths directly (commented-out code shows old approach). Always load from classpath streams to ensure certs work in JAR deployments and Docker containers.

**Debugging Certificate Issues**:
- Cert files must be in classpath; verify at runtime: `AutoChannelClient.loadTLScredentials()` logs file existence checks
- Mismatch between server/client cert chains causes `SSLException` on first message
- Temp files created in system temp dir; check via `System.getProperty("java.io.tmpdir")`
- If running fat JAR, ensure `pom.xml` `assembly` plugin includes `resources/` directory

### Testing
- Unit tests in `src/test/java/org/base/`
- `FileUtilsTest.java` - Windows ProcessBuilder tests
- Run with: `mvn test`

## Project-Specific Conventions

### Command Building Pattern
**Never** hard-code yt-dlp commands directly. Always use `CommandBuilder` to construct commands:
```java
// WRONG:
String cmd = "yt-dlp " + link + " -f mp4";

// CORRECT:
String cmd = new CommandBuilder(request.getConfig()).build();
```

### Error Handling
- `Downloader` methods return `null` on failure (not exceptions)
- `FileUtils.runScript()` returns `true`/`false` for exit code == 0
- Logger calls use `LOGGER.severe()` for errors, `LOGGER.info()` for success
- No checked exception propagation; all errors logged but not re-thrown

### File Path Handling
- Use `File` objects; paths are Windows-native (backslashes)
- Output paths passed via proto; default output: `System.getProperty("user.home")`
- `checkDownloaded()` verifies file exists before returning

### Proto Message Conventions
- `downloadType` values: exactly "Audio" or "Video" (case-sensitive)
- `outputFormat`: e.g., "mp3" (audio), "best" or "best[ext=mp4]" (video)
- `resolution`: e.g., "1080" (yt-dlp format string); empty/null = no filter
- `isPlaylist`: boolean flag; affects `--no-playlist` presence

**Proto Message Construction Examples**:

```java
// Single audio file download (high quality MP3)
DownloaderConfig audioConfig = DownloaderConfigOuterClass.DownloaderConfig.newBuilder()
    .setLink("https://www.youtube.com/watch?v=...")
    .setPath(System.getProperty("user.home") + "/Downloads")
    .setDownloadType("Audio")
    .setOutputFormat("mp3")
    .setIsPlaylist(false)
    .setRetries(3)
    .setEmbedThumbnail(true)
    .build();

DownloaderRequest request = DownloaderConfigOuterClass.DownloaderRequest.newBuilder()
    .setConfig(audioConfig)
    .build();
```

```java
// Playlist video download with resolution limit
DownloaderConfig videoConfig = DownloaderConfigOuterClass.DownloaderConfig.newBuilder()
    .setLink("https://www.youtube.com/playlist?list=...")
    .setPath(System.getProperty("user.home") + "/Videos")
    .setDownloadType("Video")
    .setOutputFormat("best[ext=mp4]")
    .setResolution("720")                    // Max 720p
    .setIsPlaylist(true)
    .setRetries(5)
    .setEmbedSubtitles(true)
    .setEmbedThumbnail(true)
    .build();

DownloaderRequest request = DownloaderConfigOuterClass.DownloaderRequest.newBuilder()
    .setConfig(videoConfig)
    .build();
```

```java
// Parsing response from server
DownloaderResponse response = client.blockingStub.executeCommand(request);
String downloadedFilePath = response.getConfig().getPath();
String originalLink = response.getConfig().getLink();
String errorMsg = response.getError();
int status = response.getStatus();

if (response.hasError() && !response.getError().isEmpty()) {
    LOGGER.severe("Download failed: " + errorMsg);
} else {
    LOGGER.info("Downloaded to: " + downloadedFilePath);
}
```

## Integration Points

### External: yt-dlp
- Invoked via Windows `cmd /c` → yt-dlp subprocess
- Must be in PATH; no fallback bundled version
- Output captured to stdout/stderr and logged

### gRPC Service Boundary
- Single RPC: `DownloaderManagerService.ExecuteCommand(DownloaderRequest) → DownloaderResponse`
- Blocking stub used for now (async stubs available but unused)
- 100 MiB inbound message limit; 1000 concurrent calls per connection

### File System
- Reads proto files, certs from classpath
- Writes downloads to user-specified paths
- Temp cert files auto-cleanup on JVM exit

## Notes on Dead Code
- `Main.java` marked "not used anymore" - legacy CLI interface
- `ConfigManager.java` - empty stub
- Gradle `build.gradle` present but Maven `pom.xml` is canonical
- Commented-out async stubs in `AutoChannelClient` and plaintext channel builders

## Common Pitfalls
1. **Forgetting proto compilation**: Proto changes require `mvn clean compile` before using generated stubs
2. **yt-dlp not in PATH**: `runScript()` will fail silently with exit code 1
3. **Certificate loading**: Certs must be in `resources/cert/` and on classpath at runtime
4. **Hardcoded paths**: Use `System.getProperty("user.home")` or proto-provided paths, not absolute strings
5. **Varargs abuse**: `isPlaylist` and `onlyLastLine` use optional varargs (e.g., `boolean...`) - avoid extending this pattern
