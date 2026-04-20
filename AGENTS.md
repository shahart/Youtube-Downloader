# Repository Guidelines

## Project Structure & Module Organization
The repository root contains documentation and one Java module: `youtubeDownload/`. Main application code lives in `youtubeDownload/src/main/java/org/base`, tests live in `youtubeDownload/src/test/java/org/base`, and protobuf definitions live in `youtubeDownload/src/main/proto`. Build metadata exists for both Gradle (`build.gradle`, `gradlew`, `gradlew.bat`) and Maven (`pom.xml`); keep changes aligned if you touch dependency or Java version settings.

## Build, Test, and Development Commands
Run commands from `youtubeDownload/`.

- `./gradlew test` or `gradlew.bat test`: run the JUnit test suite.
- `./gradlew build`: compile the module and assemble the Gradle artifact.
- `mvn test`: run tests through Maven, including protobuf code generation hooks.
- `mvn package`: build the runnable jar configured for `org.base.DownloaderServer`.

The project depends on `yt-dlp` being available on `PATH` when exercising real download flows.

## Coding Style & Naming Conventions
Follow the existing Java style: 4-space indentation, braces on the same line, and one top-level class per file. Use `UpperCamelCase` for classes, `lowerCamelCase` for methods and fields, and keep package names lowercase under `org.base`. Favor small, focused methods and avoid introducing new package roots unless the module is being reorganized deliberately. No formatter or linter is checked in, so match surrounding code closely.

## Testing Guidelines
Tests use JUnit 4 (`org.junit.Test`). Name test classes `*Test.java` and keep them beside the production package they cover. Prefer deterministic unit tests around utility and command-building logic; avoid network or `yt-dlp` integration in default test runs unless explicitly isolated. Run `gradlew.bat test` before opening a PR.

## Commit & Pull Request Guidelines
Recent history uses short, imperative commit subjects such as `fixed video and audio output extension.` and `Advanced options.` Keep subjects concise and action-oriented; group related changes into a single commit when practical. Pull requests should include a clear summary, note any build or runtime impact, link the relevant issue if one exists, and attach screenshots or logs when changing CLI output, installer flow, or server behavior.

## Configuration & Runtime Notes
TLS assets are loaded from classpath resources by the server code, and protobuf/gRPC generation is part of the Maven build. If you change certificates, proto contracts, or startup entry points, document the operational impact in the PR.
