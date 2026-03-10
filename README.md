# HearTune

HearTune는 다음 기능을 하나의 Android 앱으로 묶은 오디오 프로젝트입니다.

- 청력 테스트 기반 프로파일 생성
- 실시간 DSP 재생
- 프로파일 기반 EQ 보정
- `MediaCodec`와 FFmpeg fallback을 함께 쓰는 포맷 대응 재생

이 저장소는 Kotlin/Android 멀티모듈 프로젝트이며, 현재 기준으로 API 26+, Java 21, Kotlin 2.0.21, AGP 8.7.3 환경을 사용합니다.

## 현재 상태

2026-03-10 기준으로 저장소에는 다음 영역이 구현되어 있습니다.

- `Player`, `Hearing Test`, `Profile`, `Settings`, `Tuning` 탭을 가진 Compose 앱 셸
- underrun 및 처리 시간 진단을 포함한 `AudioTrack` 기반 실시간 재생
- `:core-dsp`의 JVM DSP 파이프라인
- quick screening, advanced/training, pause/resume, Proto DataStore 저장을 포함한 청력 테스트 흐름
- 일반 포맷은 `MediaCodec` 우선, DSD/APE는 FFmpeg 우선 경로를 사용하는 하이브리드 디코더 라우팅
- `:ndk-decoder`의 arm64 FFmpeg JNI 브리지
- `:ndk-aaudio`의 debug 전용 AAudio spike 모듈
- 재생 및 청력 테스트 검증용 unit/instrumentation 테스트 자산과 스크립트

## 저장소 규모 요약

현재 체크인된 소스 및 테스트 구성은 다음과 같습니다.

- 모듈
  - `app`
  - `core-audio`
  - `core-dsp`
  - `feature-audiometry`
  - `ndk-aaudio`
  - `ndk-decoder`
- 소스 파일 수
  - `app/src/main`: 26
  - `core-audio/src/main`: 13
  - `core-dsp/src/main`: 27
  - `feature-audiometry/src/main`: 31
  - `ndk-aaudio/src/main`: 4
  - `ndk-decoder/src/main`: 11
- 테스트 파일 수
  - `app/src/test`: 3
  - `app/src/androidTest`: 30
  - `core-audio/src/test`: 7
  - `core-dsp/src/test`: 16
  - `feature-audiometry/src/test`: 12
  - `feature-audiometry/src/androidTest`: 7

현재 복잡도가 집중된 큰 파일은 다음과 같습니다.

- `PlayerSession.kt`: 766줄
- `PlayerScreen.kt`: 815줄
- `AudiometryScreen.kt`: 1084줄
- `AudioTrackEngine.kt`: 362줄

## README 코드 레벨 점검 메모

이전 `README.md`는 코드 기준으로 여러 문제가 있었습니다.

- mojibake가 보여서 한글 섹션 일부를 읽기 어려웠습니다.
- 현재의 `:ndk-decoder` 모듈과 하이브리드 디코더 구조를 반영하지 못했습니다.
- 튜닝 상태 범위를 과소 설명하고 있었습니다. 실제 앱은 multiband, binaural, dynamic EQ, de-esser, frequency lowering, AGC, noise gate, DRC, limiter, profile application까지 저장하고 노출합니다.
- 구현 완료 상태와 향후 과제를 구분하지 못했습니다.
- 현재 검증 스크립트와 체크인된 실행 결과를 가리키지 못했습니다.

이 문서는 현재 코드 기준으로 다시 정리한 버전입니다.

## 현재 남아 있는 코드 레벨 이슈

README는 정리됐지만, 코드 자체에는 아직 문서화가 필요한 이슈가 남아 있습니다.

- 일부 UI/메시지 문자열이 코드 안에서 아직 mojibake 상태입니다.
  - `AppRoot`의 하단 탭 라벨
  - `DspImpactAnalyzer`의 사용자 메시지 일부
- `PlayerSession`, `PlayerScreen`, `AudiometryScreen`은 책임이 많이 몰려 있어 구조 파악과 유지보수 난도가 높습니다.
- 튜닝 상태 저장은 정상 동작하지만, `:app`이 `:feature-audiometry`의 저장 타입에 기대고 있어서 모듈 경계가 완전히 깔끔하진 않습니다.

## 모듈 구성

### `:app`

- Compose UI 진입점과 탭 셸
- 재생 세션 오케스트레이션
- 디코더 라우팅 및 소스 probe
- 튜닝 상태 연결과 persistence bridge

핵심 파일:

- [app/src/main/java/com/heartune/app/AppRoot.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/AppRoot.kt)
- [app/src/main/java/com/heartune/app/PlayerScreen.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/PlayerScreen.kt)
- [app/src/main/java/com/heartune/app/player/PlayerSession.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/player/PlayerSession.kt)
- [app/src/main/java/com/heartune/app/player/decoder/HybridDecoderFactory.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/player/decoder/HybridDecoderFactory.kt)

### `:core-audio`

- `AudioTrack` 출력 엔진
- PCM 양자화
- 출력 포맷 협상 및 reconcile
- ring buffer와 간단한 stream reader

### `:core-dsp`

- canonical DSP 구현
- 18-band EQ 지원
- compressor, limiter, resampler, smoothing
- 앱이 쓰는 각종 tuning processor

### `:feature-audiometry`

- 청력 테스트 UI와 로직
- 출력 경로 감시
- resume/pause persistence
- profile 및 tuning Proto DataStore 저장

### `:ndk-decoder`

- FFmpeg JNI 브리지
- native decode/open/read/close 경로
- prebuilt arm64 shared library

### `:ndk-aaudio`

- debug 전용 native AAudio sine 출력 실험

## 구현된 재생 기능

- Sine 재생
- WAV 재생
- 48 kHz raw PCM16LE stereo 재생
- `MediaCodec` 기반 encoded 재생
- FFmpeg fallback/open 경로
  - DSD (`.dsf`, `.dff`)
  - APE
  - WMA/ASF probe 지원
  - ALAC/FLAC 검증 자산 기반 테스트
- 파일 소스 loop 재생
- source와 negotiated output rate가 다를 때 runtime resampling
- 96 kHz 초과 고샘플레이트 경로에서 overload 감지 후 restart/fallback
- underrun, underflow, CPU time, render time, write time 진단

## 구현된 청력 테스트 기능

- 헤드폰 체크 흐름
- quick screening 모드
- advanced/training 경로와 session restore
- 출력 경로 변경 감지 및 auto-pause
- 테스트 볼륨 lock/restore
- Proto DataStore 프로파일 저장
- resume state persistence

핵심 파일:

- [feature-audiometry/src/main/java/com/heartune/audiometry/AudiometryScreen.kt](/d:/project_new/kotlin_audio_heartune/feature-audiometry/src/main/java/com/heartune/audiometry/AudiometryScreen.kt)
- [feature-audiometry/src/main/java/com/heartune/audiometry/logic/QuickScreeningSession.kt](/d:/project_new/kotlin_audio_heartune/feature-audiometry/src/main/java/com/heartune/audiometry/logic/QuickScreeningSession.kt)
- [feature-audiometry/src/main/java/com/heartune/audiometry/logic/AudiometrySession.kt](/d:/project_new/kotlin_audio_heartune/feature-audiometry/src/main/java/com/heartune/audiometry/logic/AudiometrySession.kt)

## 구현된 튜닝 상태

현재 persistence되는 `TuningState`에는 다음이 포함됩니다.

- DSP A/B wet-dry 모드
- profile apply 토글
- 보정 프리셋: `SOFT`, `NORMAL`, `STRONG`
- manual low/mid/high EQ 조정
- EQ enable
- DRC enable 및 DRC 파라미터
- multiband compressor
- binaural balance/crossfeed
- dynamic EQ
- de-esser
- frequency lowering
- limiter
- AGC
- noise gate

핵심 파일:

- [app/src/main/java/com/heartune/app/tuning/TuningState.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/tuning/TuningState.kt)
- [app/src/main/java/com/heartune/app/TuningScreen.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/TuningScreen.kt)
- [app/src/main/java/com/heartune/app/tuning/TuningStateMapper.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/tuning/TuningStateMapper.kt)

## 디코더 정책

현재 디코더 라우팅은 의도적으로 혼합 구조입니다.

- 일반적으로 지원되는 포맷: `MediaCodec` 우선, FFmpeg fallback
- DSD: FFmpeg 우선, open target rate `88_200 Hz`
- APE: FFmpeg 우선
- WMA/ASF: probe hint와 FFmpeg fallback 지원

관련 파일:

- [app/src/main/java/com/heartune/app/player/decoder/DecoderRoutingPolicy.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/player/decoder/DecoderRoutingPolicy.kt)
- [app/src/main/java/com/heartune/app/player/decoder/DecoderFormatHints.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/player/decoder/DecoderFormatHints.kt)
- [app/src/main/java/com/heartune/app/player/decoder/FfmpegAudioReader.kt](/d:/project_new/kotlin_audio_heartune/app/src/main/java/com/heartune/app/player/decoder/FfmpegAudioReader.kt)

## 빌드

### 요구 사항

- Android SDK platform 35
- JDK 21
- Android NDK `26.3.11579264`
- CMake `3.22.1`
- instrumentation을 돌릴 수 있는 연결 디바이스 1대 이상

### Debug APK 설치

```powershell
.\gradlew.bat :app:installDebug
```

### 실행 smoke

```powershell
adb shell am start -W -n com.heartune.app/.MainActivity
```

### Unit 테스트

```powershell
.\gradlew.bat :app:testDebugUnitTest :core-audio:test :core-dsp:test :feature-audiometry:test
```

### Instrumentation 스위트

```powershell
.\scripts\run_instrumentation_suite.ps1 -Serial <device-serial>
```

### Hybrid Decoder 검증

```powershell
.\scripts\run_hybrid_decoder_validation.ps1 -Serial <device-serial>
```

### ADB Self-Test Hook

`MainActivity`에는 UI 조작 없이 adb로 오디오 경로를 검증할 수 있는 self-test 진입점이 있습니다.

사용 가능한 intent extra:

- `self_test_audio=true`
- `induce_underrun=true|false`
- `induce_underflow=true|false`
- `self_test_phase1_ms`
- `self_test_phase2_ms`

이 경로는 자동 오디오 sanity check 용도입니다.

## 최근 기록된 검증 결과

현재 체크인된 summary 기준으로 다음 결과가 있습니다.

- `2026-03-05` instrumentation summary, device `0123456789ABCDEF`
  - overall pass: `True`
  - FLAC / ALAC / DSD workflow 및 PCM validation 클래스 통과
- `2026-03-04` hybrid decoder validation, 같은 디바이스
  - overall pass: `False`
  - 기록된 실패 클래스: `EncodedDsdLoopPlaybackTest`
- `2026-03-10` 수동 install/launch smoke, device `SP4000T` (`0123456789ABCDEF`)
  - `:app:installDebug` 성공
  - `com.heartune.app/.MainActivity` 실행 성공
  - 관측된 첫 cold launch: 약 `1979 ms`

리포트 파일 위치:

- [reports/instrumentation](/d:/project_new/kotlin_audio_heartune/reports/instrumentation)
- [reports/hybrid-validation](/d:/project_new/kotlin_audio_heartune/reports/hybrid-validation)

## 알려진 제약

- FFmpeg native artifact는 현재 `arm64-v8a` 기준으로 포함되어 있습니다.
- `PlayerScreen`, `PlayerSession`, `AudiometryScreen`은 기능이 많아 유지보수 난도가 높습니다.
- tuning persistence는 `:feature-audiometry`에 있고, `:app`이 이를 공유 인프라처럼 사용합니다.
- 일부 generated NDK `.cxx` artifact가 저장소 이력에 존재하므로 취급에 주의가 필요합니다.
- 일부 user-facing Kotlin source 문자열은 아직 깨진 상태라 코드 자체 정리가 필요합니다.

## 문서 맵

- 문서 인덱스: [docs/README.md](/d:/project_new/kotlin_audio_heartune/docs/README.md)
- 제품/범위: [docs/PRD.md](/d:/project_new/kotlin_audio_heartune/docs/PRD.md)
- 기술 기준: [docs/TRD.md](/d:/project_new/kotlin_audio_heartune/docs/TRD.md)
- 작업 상태/로드맵: [docs/TASKS.md](/d:/project_new/kotlin_audio_heartune/docs/TASKS.md)
- 검증 계획/측정 결과: [docs/VERIFICATION_PLAN.md](/d:/project_new/kotlin_audio_heartune/docs/VERIFICATION_PLAN.md)
- Native 디코더/FFmpeg 가이드: [docs/NATIVE_DECODER.md](/d:/project_new/kotlin_audio_heartune/docs/NATIVE_DECODER.md)

## 다음 권장 작업

- `PlayerSession`을 source opening, output path setup, producer loop, diagnostics 컴포넌트로 분리
- `AudiometryScreen`을 stage별 composable과 state holder로 분리
- `TuningStateRepository`를 `:feature-audiometry` 밖으로 옮길지 결정
- local/generated artifact와 ignore 규칙 정리
