# Black_Jack

This repository now includes a Unity MVP script set for the **Resonance** endless arcade game under `Assets/Scripts`:

- `GameManager`
- `RingController`
- `PulseSpawner`
- `Pulse`
- `ScoreManager`
- `InputManager`
- `UIManager`

These scripts implement the core loop, difficulty scaling, score/combo logic, and start/game-over state flow for an Android portrait setup.

## Android CI release workflow

`/.github/workflows/android-release.yml` builds Android APK + AAB on every push to `master` and publishes them in a GitHub Release tagged `v1.0.<run_number>`.

Required repository secrets:

- `UNITY_LICENSE`
- `UNITY_EMAIL`
- `UNITY_PASSWORD`


;
 = addon("eks-pod-identity-agent", cluster, null, 