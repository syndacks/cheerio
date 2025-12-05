# macOS System Audio Permissions Setup

This guide explains how to set up macOS permissions for system audio capture during development and in production builds.

## Overview

Pluely captures system audio (e.g., YouTube videos, Zoom calls) to transcribe all speakers in a conversation. This requires special macOS permissions that differ between development and production builds.

## Permission Types

### 1. Audio Input (Microphone)
- **Purpose**: Captures your voice through the microphone
- **Permission**: Standard microphone access
- **Location**: System Settings > Privacy & Security > Microphone
- **Status**: ✅ Works in dev and production builds

### 2. Screen and System Audio Recording
- **Purpose**: Captures system audio output (speakers/headphones)
- **Permission**: Screen Recording (includes system audio in macOS Ventura+)
- **Location**: System Settings > Privacy & Security > Screen Recording
- **Status**: ⚠️ Requires special setup for dev builds

## Development Build Issues

### The Problem

When running `npm run tauri dev`, the app runs as an **unsigned development binary**. macOS treats unsigned apps differently:

- ❌ The app doesn't appear in "Screen Recording" settings
- ❌ System audio capture (Core Audio TAP) fails silently
- ❌ Permission prompts may not trigger
- ✅ Microphone works (simpler permission)
- ❌ YouTube videos, Zoom calls, etc. don't get transcribed

### Solution Options

#### Option 1: Build and Run Production Binary (Quickest for Testing)

1. **Build the production app**:
   ```bash
   npm run tauri build
   ```

2. **Locate the built app**:
   ```bash
   open src-tauri/target/release/bundle/macos/
   ```

3. **Run the .app bundle**:
   - Double-click `Pluely.app`
   - The signed app will properly register with macOS

4. **Grant permissions**:
   - System Settings > Privacy & Security > Screen Recording
   - Find "Pluely" in the list
   - Toggle it ON
   - Restart the app

5. **Test system audio capture**:
   - Play a YouTube video
   - Click the headphones icon or press Cmd+Shift+M
   - The app should now transcribe the video audio

#### Option 2: Manual Code Signing for Dev Builds

For ongoing development, you can manually sign the dev binary:

1. **Clean the build**:
   ```bash
   cd src-tauri
   cargo clean
   ```

2. **Build with entitlements**:
   The `dev.entitlements` file is now configured in `tauri.conf.json`

3. **Sign the binary after each build**:
   ```bash
   codesign --force --deep --sign - \
     --entitlements dev.entitlements \
     src-tauri/target/debug/pluely
   ```

4. **Run the signed binary**:
   ```bash
   npm run tauri dev
   ```

5. **Grant permissions** (if the app now appears in System Settings)

**Note**: You may need to repeat signing after each rebuild.

#### Option 3: Create a Development Certificate

For frequent development:

1. **Create a development certificate**:
   - Open Keychain Access
   - Keychain Access > Certificate Assistant > Create a Certificate
   - Name: "Pluely Development"
   - Identity Type: Self-Signed Root
   - Certificate Type: Code Signing
   - Click Create

2. **Trust the certificate**:
   - Find the certificate in Keychain Access
   - Right-click > Get Info
   - Trust > Code Signing: Always Trust

3. **Sign with your certificate**:
   ```bash
   codesign --force --deep --sign "Pluely Development" \
     --entitlements dev.entitlements \
     src-tauri/target/debug/pluely
   ```

4. **Add to Tauri build config** (optional):
   Update `tauri.conf.json` to auto-sign during dev builds.

## Verifying Permissions

### Check Current Permissions

```bash
# Check if app has screen recording permission
sqlite3 ~/Library/Application\ Support/com.apple.TCC/TCC.db \
  "SELECT client, allowed FROM access WHERE service='kTCCServiceScreenCapture';"
```

### Check Code Signing

```bash
# View current signing status
codesign -dv --verbose=4 src-tauri/target/debug/pluely

# View entitlements
codesign -d --entitlements - src-tauri/target/debug/pluely
```

### Test Audio Capture

1. **Start the app** (dev or production)
2. **Click headphones icon** or press `Cmd+Shift+M`
3. **Check status**:
   - "Listening..." = ✅ Capture is active
   - "Permission Required" = ❌ Need to grant permission
   - Error message = ❌ Configuration issue

4. **Play audio** (YouTube video, Zoom call, etc.)
5. **Verify transcription** appears in the popup

## Troubleshooting

### App Doesn't Appear in Screen Recording Settings

**Cause**: Unsigned dev build
**Fix**: Use Option 1 (production build) or Option 2/3 (code signing)

### "Permission Required" Message

**Cause**: Permission not granted
**Fix**:
- Click "Grant Permission" button
- Or manually enable in System Settings > Privacy & Security > Screen Recording

### Microphone Works But YouTube Doesn't Get Transcribed

**Cause**: Different permission categories
**Fix**:
- Verify Screen Recording permission is granted (not just Microphone)
- Check audio output device is correctly selected in app settings
- Lower VAD sensitivity in Audio Detection Settings

### Permission Granted But Still Not Working

**Cause**: App needs to be restarted after granting permission
**Fix**:
- Quit the app completely
- Restart the app
- Try capturing again

### "No Speech Provider Selected" Error

**Cause**: STT/AI providers not configured
**Fix**:
- Open Settings
- Configure Speech Provider (e.g., OpenAI Whisper, Pluely API)
- Configure AI Provider (e.g., Claude, ChatGPT)

## Technical Details

### Core Audio TAP

macOS system audio capture uses **Core Audio Aggregate Device with TAP**:

- Creates a virtual audio device
- Taps into the speaker output stream
- Requires Screen Recording permission (includes system audio)
- Implementation: `src-tauri/src/speaker/macos.rs`

### Entitlements

The `dev.entitlements` file includes:
- `com.apple.security.device.audio-input` - Microphone access
- `com.apple.security.cs.disable-library-validation` - Load audio libraries
- `com.apple.security.app-sandbox` = false - Required for TAP access
- Additional dev-only entitlements for unsigned code

### Permission Deep Links

The app uses these deep links to open System Settings:

- **Microphone**: `x-apple.systempreferences:com.apple.preference.security?Privacy_AudioCapture`
- **Screen Recording**: `x-apple.systempreferences:com.apple.preference.security?Privacy_ScreenCapture`

## Production Considerations

For **production releases**:

1. ✅ App is properly code-signed with Apple Developer certificate
2. ✅ App is notarized by Apple
3. ✅ Permissions work automatically
4. ✅ App appears in System Settings correctly
5. ✅ Users can grant permissions normally

For **development**:
- Use production build for testing permissions
- Or manually sign dev builds
- Document limitations for contributors

## References

- [Apple TCC Documentation](https://developer.apple.com/documentation/bundleresources/entitlements)
- [Core Audio Programming Guide](https://developer.apple.com/library/archive/documentation/MusicAudio/Conceptual/CoreAudioOverview/)
- [Tauri Code Signing](https://tauri.app/v1/guides/distribution/sign-macos)

## Support

If you continue to have issues:
1. Check Console.app for error messages from Pluely or coreaudiod
2. Verify macOS version compatibility (requires 10.13+)
3. Try resetting TCC permissions: `tccutil reset ScreenCapture`
4. Open an issue on GitHub with debug logs
