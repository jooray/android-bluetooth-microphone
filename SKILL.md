---
name: android-bluetooth-microphone
description: Record audio on Android from a Bluetooth microphone (classic SCO headset, LE Audio headset, hearing aids) instead of the phone mic. Use when AudioRecord, MediaRecorder or AAudio silently records from the phone mic, returns all zeros or digital silence from a Bluetooth or BLE headset, when setCommunicationDevice, availableCommunicationDevices, TYPE_BLE_HEADSET, TYPE_BLUETOOTH_SCO, VOICE_COMMUNICATION, routedDevice or setPreferredDevice come up, when the first seconds of a Bluetooth recording are missing, when logcat shows "skip kBleRestart due to no ble edge connected", when an app picks the car's hands-free microphone instead of the headphones (Android Auto connected), when a recording service crashes with "Neither user … nor current process has android.permission.WAKE_LOCK", when the user does not know when to start talking, or when building voice notes, dictation or headset-button recording on Android 12 to 17. Also covers transcribing the finished recording on the device with Offline Speech to Text (OSTT): SpeechRecognizer with a file, EXTRA_AUDIO_SOURCE, ERROR_RECOGNIZER_BUSY.
---

# Recording from a Bluetooth microphone on Android

Selecting the Bluetooth device as the audio input does not work. The Bluetooth mic is a call microphone, and Android only gives it to apps that behave like a call. This skill is the recipe, the traps, how to debug them, and how to transcribe the result on the phone. The findings come from three real apps (OSTT, I Hear You, Third Ear); device-specific ones are marked in the last section.

Reference implementation: [Third Ear](https://github.com/jooray/third-ear) (GPLv3), in `app/src/main/kotlin/app/thirdear/audio/` (`AndroidSources.kt`: routing, primer, capture; `SourcePolicy.kt`: first-sound deadline and fallback, unit-tested) and `pipeline/Transcriber.kt` (OSTT).

## TL;DR checklist

1. Make the headset the communication device: choose it from `AudioManager.availableCommunicationDevices` (first the device media plays to, then LE Audio `TYPE_BLE_HEADSET` or Android 17's hearing-aid type 33, then `TYPE_BLUETOOTH_SCO`; see "Choosing between several devices") and call `setCommunicationDevice()`.
2. Wait for `OnCommunicationDeviceChangedListener` to report it (timeout about 3 s), then sleep about 500 ms more. The callback fires before the link has been reconfigured.
3. Record with `MediaRecorder.AudioSource.VOICE_COMMUNICATION`. `MIC` and `VOICE_RECOGNITION` silently record from the phone, and so does AAudio's default VoiceRecognition preset.
4. Call `setPreferredDevice()` with a Bluetooth input of the same type. Do not require its address to match the output's.
5. After `startRecording()`, expect the Bluetooth microphone to take **2.5 to 3 s** to come up (we measured 1.7 to 3.5 s on LE Audio). Frames are absent or exact zeros until then, and anything said meanwhile is lost. Show "waiting for the Bluetooth microphone", signal the moment it really records (see "Telling the user when to speak"), and do not reopen the stream.
6. Start a silence primer on an `AudioTrack` (`USAGE_VOICE_COMMUNICATION`) right after `startRecording()`. Without playback to the headset, some LE Audio HALs deliver only zeros for the whole recording.
7. Verify with `routedDevice`, a routing listener and `AudioRecordingCallback`. If the input is not Bluetooth, you are on the phone mic; tell the user.
8. Exact zeros mean no real audio. A quiet room still produces non-zero samples.
9. Give the first sound a generous deadline (8 s worked). Then fall back to the phone mic once, with an audible cue, and stay there for that recording.
10. On finish: stop, release, `clearCommunicationDevice()`, resume media later.
11. Optional: hand the file to Offline Speech to Text for an on-device transcript (see the last recipe section).

## Why it is counterintuitive

Android's audio input list shows a Bluetooth device, so it looks like you can pick it. You can't. Bluetooth audio has two modes. In media mode (A2DP, or the LE Audio media context) the headset is output only. A microphone exists only in the conversational mode (SCO, or the LE Audio CONV context), which the system enters for calls and for apps that declare a communication use case.

Three consequences surprise people:

- A normal recording preset records from the phone and reports no error. Your code runs, the file has audio, and it is the wrong microphone.
- Setting the communication device does not make the mic stream appear. On LE Audio the headset's microphone stream is set up when a recording starts, and that reconfiguration takes seconds.
- During that setup the recording is already "running" but delivers nothing real. Reading it and giving up, or reopening it, is the most common way to break it.

## Recipe

### 1. Permissions

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT"/>
<uses-permission android:name="android.permission.MODIFY_AUDIO_SETTINGS"/>
```

Request `RECORD_AUDIO` and `BLUETOOTH_CONNECT` at runtime before anything below.

### 2. Routing helper

```kotlin
class BtRoute(private val am: AudioManager, private val exec: Executor) {
    var device: AudioDeviceInfo? = null
        private set

    private fun isLe(t: Int) = t == AudioDeviceInfo.TYPE_BLE_HEADSET || t == 33 // 33: LE hearing aid, Android 17

    private fun pick(): AudioDeviceInfo? {
        val devs = am.availableCommunicationDevices
            .filter { isLe(it.type) || it.type == AudioDeviceInfo.TYPE_BLUETOOTH_SCO }
        // where media plays now (API 33+): the headset the user is listening on
        val media = if (Build.VERSION.SDK_INT >= 33) am.getAudioDevicesForAttributes(
            AudioAttributes.Builder().setUsage(AudioAttributes.USAGE_MEDIA).build()
        ).mapNotNull { it.address?.uppercase() }.toSet() else emptySet()
        return devs.firstOrNull { it.address.uppercase() in media }
            ?: devs.firstOrNull { isLe(it.type) }
            ?: devs.firstOrNull()
    }

    /** Returns true when the headset is the communication device and has had time to settle. */
    suspend fun acquire(timeoutMs: Long = 3000, settleMs: Long = 500): Boolean {
        val target = pick() ?: return false
        val reported = CompletableDeferred<Unit>()
        val l = AudioManager.OnCommunicationDeviceChangedListener { d ->
            if (d?.id == target.id) reported.complete(Unit)
        }
        am.addOnCommunicationDeviceChangedListener(exec, l)
        try {
            if (am.communicationDevice?.id != target.id) {
                if (!am.setCommunicationDevice(target)) return false
                withTimeoutOrNull(timeoutMs) { reported.await() } ?: return false
            }
            delay(settleMs) // the device is reported before the LE Audio link is reconfigured
            device = target
            return true
        } finally {
            am.removeOnCommunicationDeviceChangedListener(l)
        }
    }

    fun release() {
        am.clearCommunicationDevice()
        device = null
    }
}
```

SCO routing took about 550 ms in our tests; LE Audio needs the settle delay on top of the callback.

### 3. Preferred input device

```kotlin
fun findBtInput(am: AudioManager, route: AudioDeviceInfo): AudioDeviceInfo? {
    val inputs = am.getDevices(AudioManager.GET_DEVICES_INPUTS)
    // LE Audio: compare the family, not the exact type (hearing aids can list output and input
    // under different LE types) and accept another address (a binaural set exposes both sides)
    fun le(t: Int) = t == AudioDeviceInfo.TYPE_BLE_HEADSET || t == 33
    fun family(d: AudioDeviceInfo) = if (le(route.type)) le(d.type) else d.type == route.type
    return inputs.firstOrNull { family(it) && it.address.equals(route.address, ignoreCase = true) }
        ?: inputs.firstOrNull { family(it) }
}
```

### Choosing between several devices

`availableCommunicationDevices` lists every hands-free link in no useful order. With a car connected
(Android Auto, or just the car's HFP) and headphones on, a plain "first LE, else first SCO" can pick
the car: Android Auto with audio left on the phone (GrapheneOS lets you turn off "reroute audio to
Android Auto" for the sandboxed app) keeps media on the headphones, and the user expects their
microphone. Prefer the communication device whose address matches the current media output
(`getAudioDevicesForAttributes(USAGE_MEDIA)`, public since API 33; an A2DP headphone's address is
the same as its SCO address), then LE Audio, then classic SCO. An app built around one device (the
user's hearing aids) can still put that one first. Keep the policy a pure function over
(type, address) pairs and unit-test the car + headphones case.

### 4. The recorder

16 kHz mono PCM16, buffer of at least one second, reader thread at urgent audio priority.

```kotlin
@SuppressLint("MissingPermission")
fun openRecord(btInput: AudioDeviceInfo?): AudioRecord {
    val rate = 16000
    val min = AudioRecord.getMinBufferSize(rate, AudioFormat.CHANNEL_IN_MONO, AudioFormat.ENCODING_PCM_16BIT)
    val buf = maxOf(min * 4, rate * 2) // 1 s of PCM16 = 32000 bytes
    val rec = AudioRecord.Builder()
        .setAudioSource(MediaRecorder.AudioSource.VOICE_COMMUNICATION)
        .setAudioFormat(AudioFormat.Builder()
            .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
            .setSampleRate(rate)
            .setChannelMask(AudioFormat.CHANNEL_IN_MONO).build())
        .setBufferSizeInBytes(buf)
        .build()
    if (btInput != null) rec.setPreferredDevice(btInput)
    return rec
}
```

`VOICE_COMMUNICATION` may turn on echo cancellation and noise suppression. That is acceptable for speech; check `AcousticEchoCanceler.isAvailable()` if you need raw audio.

### 5. The silence primer

```kotlin
fun startPrimer(seconds: Int = 3): AudioTrack {
    val rate = 16000
    val bytes = rate * 2 * seconds
    val t = AudioTrack.Builder()
        .setAudioAttributes(AudioAttributes.Builder()
            .setUsage(AudioAttributes.USAGE_VOICE_COMMUNICATION)
            .setContentType(AudioAttributes.CONTENT_TYPE_SPEECH).build())
        .setAudioFormat(AudioFormat.Builder()
            .setEncoding(AudioFormat.ENCODING_PCM_16BIT)
            .setSampleRate(rate)
            .setChannelMask(AudioFormat.CHANNEL_OUT_MONO).build())
        .setBufferSizeInBytes(bytes)
        .setTransferMode(AudioTrack.MODE_STATIC)
        .build()
    t.write(ByteArray(bytes), 0, bytes) // zeros
    t.play()
    return t
}
```

Start it right after `startRecording()`. It follows the communication device, so the headset gets an active output stream while its input comes up. Stop and release it when the first real audio arrives or after the three seconds. See pitfall 6 for why this matters.

### 6. Capture loop with first-sound watchdog

```kotlin
class BtRecorder(
    private val rec: AudioRecord,
    private val onFirstSound: () -> Unit,
    private val onFailure: (String) -> Unit,
) {
    @Volatile var running = true
    val pcm = ByteArrayOutputStream()

    fun start(startTimeoutMs: Long = 8000) = thread(name = "bt-rec") {
        Process.setThreadPriority(Process.THREAD_PRIORITY_URGENT_AUDIO)
        val buf = ShortArray(1600) // 100 ms
        val t0 = SystemClock.elapsedRealtime()
        var soundSeen = false
        var emptyReads = 0
        rec.startRecording()
        while (running) {
            val n = rec.read(buf, 0, buf.size)
            if (n < 0) { onFailure("read error $n"); break }
            if (n == 0) {
                // before sound: normal while the LE link is being set up. After: failure.
                if (soundSeen && ++emptyReads > 10) { onFailure("stalled"); break }
                if (!soundSeen && SystemClock.elapsedRealtime() - t0 > startTimeoutMs) {
                    onFailure("no sound in ${startTimeoutMs} ms"); break
                }
                Thread.sleep(10); continue
            }
            emptyReads = 0
            if (!soundSeen) {
                if (buf.take(n).any { it.toInt() != 0 }) { soundSeen = true; onFirstSound() }
                else if (SystemClock.elapsedRealtime() - t0 > startTimeoutMs) {
                    onFailure("only zeros for ${startTimeoutMs} ms"); break
                }
                // Keep the zero frames only if you want timing continuity; words said here are lost anyway.
                continue
            }
            val bb = ByteBuffer.allocate(n * 2).order(ByteOrder.LITTLE_ENDIAN)
            for (i in 0 until n) bb.putShort(buf[i])
            pcm.write(bb.array())
        }
    }
}
```

The limits for stalls and zeros apply only after the first non-zero sample. Before it, the only limit is the generous start timeout.

### 7. Orchestration and fallback

```
acquire route (wait + settle)
 -> open AudioRecord(VOICE_COMMUNICATION, preferred BT input)
 -> startRecording
 -> start primer
 -> UI: "waiting for the Bluetooth microphone..."   (play a "go ahead" cue here if you like)
 -> first non-zero audio -> UI: recording
 -> no sound after N seconds -> fallback
```

```kotlin
suspend fun record(): ByteArray {
    val route = BtRoute(am, mainExecutor)
    var onBt = route.acquire()
    var rec = openRecord(if (onBt) findBtInput(am, route.device!!) else null)
    val primer = if (onBt) startPrimer() else null
    // ... start BtRecorder; on failure with onBt == true:
    //   stop + release rec, route.release(), primer?.release()
    //   playCue(R.raw.fallback_phone)          // distinct, audible
    //   onBt = false; rec = openRecord(null); restart the reader
    //   keep the PCM already captured, append the phone audio
    // On any end: rec.stop(); rec.release(); route.release()
}
```

Fall back at most once per recording and then stay on the phone until it ends. Flipping back and forth produces gaps and confusing audio. Keep what was already captured.

### 8. Telling the user when to speak

The wait is the part users notice. Android usually needs 2.5 to 3 seconds between "recording started" and a Bluetooth microphone that really delivers sound, and nothing in the platform tells the user. People start talking when they tap, so the first words of every note go missing and they blame the app (or the headset).

- Do not signal at `startRecording()`, nor when the communication device is reported. Signal on the **first non-zero frame** (the `onFirstSound` callback above). That is the only moment that is true.
- Show a waiting state until then ("Waiting for the Bluetooth microphone…"), with a timer that does not run yet or a distinct lamp.
- Offer an audible cue, because Bluetooth recording usually happens without looking at the screen: a short rising tone, or a spoken "Go ahead". Make it a setting (off, tone, voice).
- On a Bluetooth microphone a spoken cue is fine: it plays into the headset and the user can talk over it. On the **phone's own microphone** use only a short tone, or nothing: the speaker's sound is recorded into the note, and a spoken clip ends up in the transcript.
- Pad the cue with leading silence or play it a moment after the first frame (pitfall 9): the first fraction of a second is clipped while the link changes mode.
- Use the same channel for the fallback: a distinct falling tone when the Bluetooth microphone did not come up and the phone mic took over.

Third Ear does all of this; its "Listening cue" setting is off, tone, or "Go ahead".

### 9. Verify where you actually record

```kotlin
fun logRouting(rec: AudioRecord, am: AudioManager, exec: Executor) {
    Log.i("BtRec", "routed=${rec.routedDevice?.type} ${rec.routedDevice?.productName}")
    rec.addOnRoutingChangedListener({ r ->
        Log.i("BtRec", "routing changed -> ${r.routedDevice?.type}")
    }, Handler(Looper.getMainLooper()))
    am.registerAudioRecordingCallback(exec, object : AudioManager.AudioRecordingCallback() {
        override fun onRecordingConfigChanged(cfgs: List<AudioRecordingConfiguration>) {
            cfgs.forEach { Log.i("BtRec", "src=${it.clientAudioSource} silenced=${it.isClientSilenced} dev=${it.audioDevice?.type}") }
        }
    })
}

fun isBluetoothInput(d: AudioDeviceInfo?) =
    d?.type == AudioDeviceInfo.TYPE_BLE_HEADSET || d?.type == AudioDeviceInfo.TYPE_BLUETOOTH_SCO
```

If `routedDevice` is not Bluetooth, you are recording from the phone. Tell the user in the UI. Do not assume the recording is fine because audio is arriving. (The device name in `productName` is user-facing data; don't log it in shared bug reports.)

### Java / AAudio note

AAudio from native code works, but its default input preset is VoiceRecognition, which records from the phone. Set the preset and the device explicitly:

```c
AAudioStreamBuilder_setInputPreset(b, AAUDIO_INPUT_PRESET_VOICE_COMMUNICATION);
AAudioStreamBuilder_setDeviceId(b, btInputDeviceId); // AudioDeviceInfo.getId() from Java
AAudioStreamBuilder_setDirection(b, AAUDIO_DIRECTION_INPUT);
```

The communication device must still be set from Java first. Until the LE link is up, AAudio delivers zeros rather than no frames, so the first-sound watchdog above applies unchanged. Float capture worked once the HAL issue (pitfall 6) was handled, but `AudioRecord` with PCM16 is the safer default.

### 10. Transcribe on the phone with Offline Speech to Text

[Offline Speech to Text (OSTT)](https://zapstore.dev/apps/app.offlinespeechtotext) is a free Android app that transcribes on the device, offline, and exposes itself as a standard `RecognitionService`. A recorder hands it the finished file and gets text back, headless, with the screen locked. No speech engine in your app, no server, no network.

Two contracts, pick by capability:

- **File API** (OSTT with `app.offlinespeechtotext.action.TRANSCRIBE_AUDIO`): `SpeechRecognizer` with the extra `app.offlinespeechtotext.extra.AUDIO_URI`, a `content:` Uri you granted read access to. OSTT decodes the file itself, any length.
- **Caller audio** (Android 13+, any recognizer): decode the file to 16 kHz mono PCM16 yourself and stream it through `RecognizerIntent.EXTRA_AUDIO_SOURCE` (a pipe); close the writer at the end and the recognizer finalises.

Never send `AUDIO_URI` to a provider that does not advertise the file API: a recognizer that ignores the extra starts listening to the microphone instead.

Manifest (package visibility, Android 11+):

```xml
<queries>
    <intent><action android:name="android.speech.RecognitionService" /></intent>
    <!-- OSTT's file contract ships with this activity; it is the capability probe -->
    <intent>
        <action android:name="app.offlinespeechtotext.action.TRANSCRIBE_AUDIO" />
        <data android:scheme="content" />
    </intent>
    <package android:name="app.offlinespeechtotext" />
</queries>
```

```kotlin
const val OSTT = "app.offlinespeechtotext"
const val EXTRA_AUDIO_URI = "app.offlinespeechtotext.extra.AUDIO_URI"

/** OSTT first, then any other recognition service. */
fun providers(ctx: Context): List<ComponentName> =
    ctx.packageManager.queryIntentServices(Intent(RecognitionService.SERVICE_INTERFACE), 0)
        .filter { it.serviceInfo.exported && it.serviceInfo.enabled }
        .map { ComponentName(it.serviceInfo.packageName, it.serviceInfo.name) }
        .sortedBy { if (it.packageName == OSTT) 0 else 1 }

fun supportsFileApi(ctx: Context, c: ComponentName) = ctx.packageManager.queryIntentActivities(
    Intent("app.offlinespeechtotext.action.TRANSCRIBE_AUDIO").setPackage(c.packageName)
        .setData(Uri.parse("content://probe/audio")), 0,   // matches only with a content: Uri
).isNotEmpty()

/** Call on the main thread; [audio] is a content: Uri from your FileProvider. */
fun transcribe(ctx: Context, c: ComponentName, audio: Uri, done: (Result<String>) -> Unit) {
    val r = SpeechRecognizer.createSpeechRecognizer(ctx, c)
    r.setRecognitionListener(object : RecognitionListener {
        override fun onResults(b: Bundle?) {
            r.destroy()
            val text = b?.getStringArrayList(SpeechRecognizer.RESULTS_RECOGNITION)?.firstOrNull()
            done(if (text.isNullOrBlank()) Result.failure(Exception("no speech")) else Result.success(text))
        }
        override fun onError(e: Int) { r.destroy(); done(Result.failure(Exception("error $e"))) }
        override fun onReadyForSpeech(p: Bundle?) {}; override fun onBeginningOfSpeech() {}
        override fun onRmsChanged(v: Float) {}; override fun onBufferReceived(b: ByteArray?) {}
        override fun onEndOfSpeech() {}; override fun onPartialResults(b: Bundle?) {}
        override fun onEvent(t: Int, p: Bundle?) {}
    })
    ctx.grantUriPermission(c.packageName, audio, Intent.FLAG_GRANT_READ_URI_PERMISSION)
    r.startListening(Intent(RecognizerIntent.ACTION_RECOGNIZE_SPEECH)
        .putExtra(RecognizerIntent.EXTRA_LANGUAGE_MODEL, RecognizerIntent.LANGUAGE_MODEL_FREE_FORM)
        .putExtra(RecognizerIntent.EXTRA_PREFER_OFFLINE, true)
        .putExtra(EXTRA_AUDIO_URI, audio))
}
```

For the caller-audio contract replace `EXTRA_AUDIO_URI` with `EXTRA_AUDIO_SOURCE` (read end of `ParcelFileDescriptor.createPipe()`), `EXTRA_AUDIO_SOURCE_CHANNEL_COUNT = 1`, `EXTRA_AUDIO_SOURCE_ENCODING = ENCODING_PCM_16BIT`, `EXTRA_AUDIO_SOURCE_SAMPLING_RATE = 16000`, and feed the decoded PCM (`MediaExtractor` + `MediaCodec`, resampled to 16 kHz mono) from a thread into the write end.

What made it reliable in Third Ear:

- **One job at a time.** OSTT answers a request it cannot take now with `ERROR_RECOGNIZER_BUSY`, `ERROR_AUDIO`, `ERROR_SERVER` or `ERROR_SERVER_DISCONNECTED` (for example while its keyboard dictates). Treat those as "busy" and retry with backoff for about 10 minutes; keep the recording and the job across process death. `ERROR_NO_MATCH` means it ran and heard no words: do not retry.
- **A timeout that scales.** Inference runs above real time, but give it 3 minutes plus twice the recording's duration. A newer recognition request from another app can retire yours without any callback, so a missing answer counts as busy.
- **Revoke the Uri grant** and `destroy()` the recognizer on every exit path.
- **Run it from a foreground service** (or WorkManager with a foreground notification) so it survives the screen going off; `SpeechRecognizer` must be created and called on the main looper.
- Save the transcript next to the audio (`note.m4a` + `note.txt`): searchable later and recoverable if the delivery step fails.

## Pitfalls

1. Default presets record from the phone. `MIC`, `VOICE_RECOGNITION`, `UNPROCESSED` and AAudio's default preset do not open LE Audio inputs. Only the communication use case does.
2. `setPreferredDevice` alone is not routing. It is a hint inside a route that already exists.
3. The callback lies a little. `OnCommunicationDeviceChangedListener` reports the device before the LE Audio stack has reconfigured. Start recording right away and the setup races with you.
4. The mic stream starts at `startRecording()`. The stack reconfigures to the CONV context with the sequence CODEC_CONFIG, QOS, ENABLE, then ISO data path. We measured 1.7 to 3.5 s from `startRecording()` to the first audio, typically 2.5 to 3 s. Words spoken earlier are gone, so show a waiting state and signal the first real audio (recipe 8).
5. Do not reopen to "fix" the wait. Closing the stream releases the route and restarts the LE setup, so it never finishes. A watchdog that declared the stream dead after 2 to 3 s of zeros and reopened looped until it ended on the phone mic. Give a starting stream generous time; apply stall and zero limits only after sound has arrived.
6. A silent LE input can stay silent. If an LE Audio input starts while nothing is playing to the headset, the audio HAL may log `skip kBleRestart due to no ble edge connected`. The input then delivers exact digital zeros for the whole recording, even though the Bluetooth stack shows both directions streaming (ASEs STREAMING, ISO data paths set up both ways, RESUME_CONFIRMED). It was intermittent. One app that happened to play a spoken cue into the headset right after starting the recorder always worked; another, silent, always failed on the same headset minutes apart. The primer above fixed it: logs then showed `ble_restart end` and real audio within 1 to 2 s. Seen on one HAL; others may differ, but the primer costs nothing.
7. Other sounds delay the mic. Anything playing to the headset (a notification, a sonification cue) holds the LE link in the sound-effects (SFX) context. Mic setup starts only after it ends.
8. Zeros are not a quiet room. Room noise gives non-zero samples. Exact zeros mean no real audio: a privacy toggle, a muted headset, an input not set up, or the HAL issue above. This is also a cheap self-test that needs nobody to speak.
9. Cues get clipped. Sonification cues (`USAGE_ASSISTANCE_SONIFICATION`) reach the headset during recording and mix over the call. The first fraction of a second of a cue played as the link switches modes is cut off, so pad cue files with leading silence (we used 2 s) so the stream is open while the link settles. A spoken "go ahead" while already recording is fine, because the user can talk over it.
10. Media sounds flat in call mode. While the headset is in call mode, music to it uses the call codec. Pause media at the start through MediaSession transport controls (for example a NotificationListener plus `MediaSessionManager`), not by requesting audio focus. Resume about 1.5 s after the recording stopped and the route was cleared, and only if nothing else started playing.
11. Short reads are failures only after sound. More than 10 consecutive empty reads or a negative read after the first sound: stop and save what you have.
12. Do not forget `clearCommunicationDevice()`. If you leave it set, the headset stays in call mode and other apps' audio stays flat.
13. Declare `WAKE_LOCK` if you take one. A microphone foreground service that keeps recording with the screen off often takes a partial wake lock; without `<uses-permission android:name="android.permission.WAKE_LOCK"/>` the `acquire()` throws a SecurityException ("Neither user … nor current process has android.permission.WAKE_LOCK") and kills the service at start, so every recording crashes, which looks like a Bluetooth bug when the user happened to use a headset. Declare it and wrap `acquire()` so a missing lock never ends a recording. (The audio HAL holds its own `AudioIn` wake lock for an active record client, so the read loop keeps running; your lock is for your encoder, UI updates and the work after the stop.)
14. A stuck state exists. We once saw a headset advertise only the CONV context and the stack answer every RESUME_REQ with REQUEST_CANCELED for over 20 s. We never explained it. Fall back to the phone mic with an audible cue in that case.

## Background recording (headset button, screen off)

- Use a foreground service with `foregroundServiceType="microphone"` and the `FOREGROUND_SERVICE_MICROPHONE` permission.
- On Android 14 and later that type can only be started while the app is visible (a while-in-use permission). After a reboot or an app update the service can run but cannot record until the user opens the app once.
- Restart your service on `ACTION_MY_PACKAGE_REPLACED` as well as `BOOT_COMPLETED`. An update kills the process, and without that the service stays dead.
- Tell the user clearly when the mic is not available ("open the app once"); a silent failure looks like a Bluetooth bug.
- From the background you may write to the clipboard. You may not start activities.
- Verify screen-off continuity, do not assume it: record, `adb shell input keyevent 26` (or let the phone doze), wait 30 s, stop, then compare the file's duration with wall time and look for runs of exact zeros, e.g. `ffmpeg -i f.m4a -af silencedetect=n=-120dB:d=0.5 -f null -` (a room never reaches -120 dB). Nothing in activity or fragment lifecycle callbacks may stop the recorder; a source-scanning unit test can enforce that.

## Debugging recipe

Reproduce, then dump immediately. The LE Audio ring buffer holds only about a minute.

```bash
# your tag plus the audio HAL's Bluetooth and router lines
adb logcat -v time | grep -E "BtRec|AHal::BT|AHal::Router"
```

Lines worth finding:

- `skip kBleRestart` means the HAL bug from pitfall 6.
- `ble_restart` and `ble_restart end` mean the restart that fixes it ran.
- `BLE_CONVERSATION` means the conversational context was selected.
- `bluetooth-low-energy-headset-microphones -> voip-...-capture` means the policy routed the right input.

```bash
adb shell dumpsys bluetooth_manager    # LE Audio state machine log
adb shell dumpsys media.audio_policy   # active record clients, source, flags, silenced, effects
adb shell dumpsys audio                # communication device, recording configs
```

In the LE Audio log look for:

- `RESUME_REQ: LocalSink` is the start of a resume.
- `METADATA_UPDATE: Reconfigure SFX->CONV` is the switch to the call context.
- `ENABLE ... meta: 03020200` is the CONV context (the stream was enabled for conversation).
- `REQUEST_CANCELED` means the stack refused. See pitfall 14.
- `Group sink/source available contexts` shows what the headset advertises.

Compare a working app against a failing one side by side on the same headset. Policy-level dumps can look identical while the HAL behaves differently; that is how pitfall 6 was found.

## Device-specific notes

- Android 17 on the Pixel 10 Pro Fold: LE Audio hearing aids showed up as `TYPE_BLE_HEADSET` (26) in `availableCommunicationDevices` and as the routed input in our Third Ear test. OSTT's policy also treats type 33 as Android 17's LE hearing-aid type (not yet a public constant in SDK 36); accepting both costs nothing.
- A car's microphone under Android Auto is classic HFP/SCO; with the car and LE hearing aids both connected, LE wins unless media plays to the car.

Everything above was measured on a Pixel 10 Pro Fold, Android 17, with LE Audio hearing aids, plus one classic-SCO headset (Sony WH-1000XM6). Treat the numbers as ballparks, not guarantees.

- SCO routing took about 550 ms. LE Audio needed the 500 ms settle delay plus 1.7 to 3.5 s from `startRecording()` to the first real audio.
- The `skip kBleRestart due to no ble edge connected` behavior was seen on this Pixel's audio HAL. Other vendors may start the input without any playback, or break in other ways. Keep the primer, since it is harmless.
- The hearing aids exposed a binaural set whose input address could differ from the output address. Matching by type, not address, worked.
- The SCO headset uses the same flow (communication device, `VOICE_COMMUNICATION`).
- AAudio from Rust initially recorded from the phone mic because of the default VoiceRecognition preset. After switching to the communication preset it worked, with zeros during link setup.
- Android 14 and later restricts starting the microphone foreground service from the background; this is platform behavior, not device-specific.
