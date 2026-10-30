# Recording from a Bluetooth microphone on Android

Recording from a Bluetooth headset, LE Audio earbuds or hearing aids on Android is surprisingly
hard, and most apps get it wrong. The obvious code runs without an error and records from the
phone's microphone. When it does reach the headset, the first two or three seconds are missing,
or the whole file is digital silence.

This repository collects what it takes to get it right, learned the hard way while building
[Third Ear](https://github.com/jooray/third-ear), I Hear You and
[Offline Speech to Text](https://zapstore.dev/apps/app.offlinespeechtotext):

- why the Bluetooth microphone is a call microphone, and how to make Android hand it over
  (communication device, `VOICE_COMMUNICATION`, preferred device);
- the 2.5 to 3 second wait before the microphone really delivers sound, and how to tell the user
  when to start talking;
- the silence primer that stops some LE Audio HALs from recording only zeros;
- picking the right device when a car and headphones are both connected;
- a first-sound deadline and a single, audible fallback to the phone microphone;
- recording with the screen off from a foreground service;
- transcribing the result on the phone with Offline Speech to Text;
- how to debug it with logcat and `dumpsys`.

Everything is in [SKILL.md](SKILL.md). It reads fine as an article, and it is also an
[Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview), so a coding
agent picks it up when you work on Android audio. To install it for Claude Code:

```sh
git clone https://github.com/jooray/android-bluetooth-microphone ~/.claude/skills/android-bluetooth-microphone
```

Measured on a Pixel 10 Pro Fold with Android 17, LE Audio hearing aids and a classic SCO headset.
Other phones and headsets will differ in the details; corrections and measurements from other
devices are welcome as issues or pull requests.

Juraj Bednár. MIT license.
