# 🎧 Ambient Mixer 기술 문서

> Web Audio API를 활용한 프로시저럴 사운드 생성의 모든 것

## 📋 목차

1. [프로젝트 개요](#프로젝트-개요)
2. [Web Audio API 기초](#web-audio-api-기초)
3. [노이즈 생성 알고리즘](#노이즈-생성-알고리즘)
4. [사운드별 구현 상세](#사운드별-구현-상세)
5. [오디오 노드 그래프](#오디오-노드-그래프)
6. [스케줄링과 타이밍](#스케줄링과-타이밍)
7. [상태 관리](#상태-관리)
8. [성능 최적화](#성능-최적화)

---

## 프로젝트 개요

### 왜 프로시저럴 사운드인가?

일반적인 앰비언트 사운드 앱은 미리 녹음된 오디오 파일을 사용합니다. 하지만 이 프로젝트는 다른 접근 방식을 택했습니다:

| 방식 | 장점 | 단점 |
|-----|------|------|
| **오디오 파일** | 높은 품질, 자연스러움 | 파일 용량, 네트워크 필요 |
| **프로시저럴 생성** | 무한 반복 없음, 오프라인 가능, 경량 | 구현 복잡도, 완벽한 자연스러움 달성 어려움 |

### 기술 스택

```
┌─────────────────────────────────────┐
│           Single HTML File           │
├─────────────────────────────────────┤
│  HTML5  │  CSS3  │  Vanilla JS      │
├─────────────────────────────────────┤
│         Web Audio API                │
│  (AudioContext, Oscillator, Filter)  │
└─────────────────────────────────────┘
```

---

## Web Audio API 기초

### AudioContext 초기화

Web Audio API의 모든 것은 `AudioContext`에서 시작됩니다:

```javascript
let audioContext = null;
let masterGainNode = null;

function initAudioContext() {
    if (!audioContext) {
        audioContext = new (window.AudioContext || window.webkitAudioContext)();
        masterGainNode = audioContext.createGain();
        masterGainNode.connect(audioContext.destination);
        masterGainNode.gain.value = 0.7;
    }
    // 브라우저 정책: 사용자 인터랙션 후에만 재생 가능
    if (audioContext.state === 'suspended') {
        audioContext.resume();
    }
}
```

> ⚠️ **중요**: 모던 브라우저는 사용자 인터랙션(클릭 등) 없이는 오디오 재생을 차단합니다. 
> 그래서 재생 버튼을 클릭할 때 `initAudioContext()`를 호출합니다.

### 오디오 노드 체인

Web Audio API는 **노드 기반 아키텍처**를 사용합니다:

```
Source Node → Processing Nodes → Destination
    ↓              ↓                 ↓
 (소리 생성)    (필터, 게인)      (스피커 출력)
```

```javascript
// 기본 체인 예시
const source = audioContext.createBufferSource();
const gainNode = audioContext.createGain();

source.connect(gainNode);
gainNode.connect(audioContext.destination);
```

---

## 노이즈 생성 알고리즘

### 노이즈의 종류

```
주파수 스펙트럼
                화이트 노이즈: 모든 주파수 균등
     ▲          ═══════════════════════════
에   │          
너   │   핑크 노이즈: 옥타브당 -3dB
지   │          ════════════════
     │                   ════════════
     │          브라운 노이즈: 옥타브당 -6dB
     │          ════════
     │               ════════
     │                    ════════════════
     └──────────────────────────────────────▶
           저주파              고주파
```

### 1. 화이트 노이즈 (White Noise)

가장 단순한 형태로, 모든 주파수에 동일한 에너지가 분포합니다:

```javascript
function createWhiteNoise(bufferSize) {
    const buffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
    const output = buffer.getChannelData(0);
    
    for (let i = 0; i < bufferSize; i++) {
        // -1 ~ 1 사이의 랜덤 값
        output[i] = Math.random() * 2 - 1;
    }
    
    return buffer;
}
```

**특성**: 날카롭고 밝은 소리, "쉬이이" 하는 TV 노이즈와 유사

### 2. 브라운 노이즈 (Brown/Brownian Noise)

로버트 브라운의 브라운 운동에서 이름을 딴 노이즈입니다. **적분된 화이트 노이즈**로, 저주파가 강조됩니다:

```javascript
function createBrownNoise(bufferSize) {
    const buffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
    const output = buffer.getChannelData(0);
    
    let lastOut = 0;
    for (let i = 0; i < bufferSize; i++) {
        const white = Math.random() * 2 - 1;
        // 이전 값에 작은 변화만 더함 (적분 효과)
        output[i] = (lastOut + (0.02 * white)) / 1.02;
        lastOut = output[i];
        output[i] *= 3.5; // 볼륨 보정
    }
    
    return buffer;
}
```

**수학적 원리**:
```
B(t) = B(t-1) + ε * W(t)

여기서:
- B(t): 현재 브라운 노이즈 값
- B(t-1): 이전 값
- ε: 작은 상수 (0.02)
- W(t): 화이트 노이즈
```

**특성**: 깊고 부드러운 소리, 폭포나 바람 소리에 적합

### 3. 핑크 노이즈 (Pink Noise)

화이트와 브라운의 중간으로, **옥타브당 -3dB** 감쇠합니다. 자연에서 가장 흔한 노이즈 스펙트럼입니다:

```javascript
function createPinkNoise(bufferSize) {
    const buffer = audioContext.createBuffer(1, bufferSize, audioContext.sampleRate);
    const output = buffer.getChannelData(0);
    
    // Voss-McCartney 알고리즘의 변형
    let b0 = 0, b1 = 0, b2 = 0, b3 = 0, b4 = 0, b5 = 0, b6 = 0;
    
    for (let i = 0; i < bufferSize; i++) {
        const white = Math.random() * 2 - 1;
        
        // 각 계수는 서로 다른 시간 상수를 가진 1차 필터
        b0 = 0.99886 * b0 + white * 0.0555179;
        b1 = 0.99332 * b1 + white * 0.0750759;
        b2 = 0.96900 * b2 + white * 0.1538520;
        b3 = 0.86650 * b3 + white * 0.3104856;
        b4 = 0.55000 * b4 + white * 0.5329522;
        b5 = -0.7616 * b5 - white * 0.0168980;
        
        output[i] = b0 + b1 + b2 + b3 + b4 + b5 + b6 + white * 0.5362;
        output[i] *= 0.11; // 볼륨 정규화
        b6 = white * 0.115926;
    }
    
    return buffer;
}
```

**특성**: 가장 자연스러운 노이즈, 빗소리나 바람의 기본 톤으로 적합

---

## 사운드별 구현 상세

### 🌧️ 비 (Rain)

비 소리는 **필터링된 화이트 노이즈**로 구현합니다:

```javascript
case 'rain':
    nodes.source = audioContext.createBufferSource();
    nodes.source.buffer = createNoiseBuffer('white');
    nodes.source.loop = true;
    
    // 로우패스 필터로 날카로운 고주파 제거
    nodes.filter = audioContext.createBiquadFilter();
    nodes.filter.type = 'lowpass';
    nodes.filter.frequency.value = 3000; // 3kHz 이상 차단
    
    nodes.source.connect(nodes.filter);
    nodes.filter.connect(nodes.gainNode);
    nodes.source.start();
    break;
```

**원리**: 화이트 노이즈에서 고주파를 제거하면 빗방울이 바닥에 부딪히는 느낌이 납니다.

### ⛈️ 천둥 (Thunder)

천둥은 **저주파 브라운 노이즈 + 볼륨 변조**입니다:

```javascript
case 'thunder':
    nodes.source = audioContext.createBufferSource();
    nodes.source.buffer = createNoiseBuffer('brown');
    nodes.source.loop = true;
    
    nodes.filter = audioContext.createBiquadFilter();
    nodes.filter.type = 'lowpass';
    nodes.filter.frequency.value = 200; // 매우 낮은 주파수만
    
    // ... 연결 ...
    scheduleThunder(nodes); // 주기적 볼륨 변조
    break;

function scheduleThunder(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    const volume = nodes.gainNode.gain.value;
    
    if (volume > 0) {
        // 천둥 효과: 갑자기 커졌다가 서서히 줄어듦
        nodes.gainNode.gain.setValueAtTime(volume * 0.5, now);
        nodes.gainNode.gain.linearRampToValueAtTime(volume * 2, now + 0.5);
        nodes.gainNode.gain.linearRampToValueAtTime(volume, now + 3);
    }
    
    // 8~23초 후 다음 천둥
    nodes.thunderTimeout = setTimeout(
        () => scheduleThunder(nodes), 
        8000 + Math.random() * 15000
    );
}
```

### 🔥 모닥불 (Fire)

모닥불은 두 가지 레이어로 구성됩니다:

1. **기본 톤**: 밴드패스 필터링된 브라운 노이즈
2. **타닥거림**: 짧은 고주파 임펄스

```javascript
// 기본 불 소리
nodes.filter.type = 'bandpass';
nodes.filter.frequency.value = 500;
nodes.filter.Q.value = 2; // 좁은 대역폭

// 타닥거리는 소리 스케줄링
function scheduleCrackles(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    
    const crackleGain = audioContext.createGain();
    const noise = audioContext.createBufferSource();
    noise.buffer = createNoiseBuffer('white');
    
    const filter = audioContext.createBiquadFilter();
    filter.type = 'highpass';
    filter.frequency.value = 1000;
    
    noise.connect(filter);
    filter.connect(crackleGain);
    crackleGain.connect(nodes.gainNode);
    
    // 짧은 임펄스
    crackleGain.gain.setValueAtTime(0.15, now);
    crackleGain.gain.setTargetAtTime(0, now, 0.01); // 빠른 감쇠
    
    noise.start(now);
    noise.stop(now + 0.05);
    
    // 100~500ms 간격으로 반복
    nodes.crackleTimeout = setTimeout(
        () => scheduleCrackles(nodes), 
        100 + Math.random() * 400
    );
}
```

### 🌊 파도 (Waves)

파도는 **주기적인 볼륨 + 필터 변조**로 밀려오고 빠지는 느낌을 표현합니다:

```javascript
function scheduleWaves(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    const vol = nodes.targetVolume;
    
    // 파도 주기: 조용 → 밀려옴 → 부서짐 → 빠져나감
    nodes.gainNode.gain.cancelScheduledValues(now);
    nodes.gainNode.gain.setValueAtTime(vol * 0.1, now);        // 시작: 조용
    nodes.gainNode.gain.linearRampToValueAtTime(vol * 0.4, now + 2);   // 밀려옴
    nodes.gainNode.gain.linearRampToValueAtTime(vol * 1.0, now + 3.5); // 최고점
    nodes.gainNode.gain.linearRampToValueAtTime(vol * 0.6, now + 4.5); // 빠지기 시작
    nodes.gainNode.gain.linearRampToValueAtTime(vol * 0.1, now + 7);   // 빠져나감
    
    // 필터도 함께 변조 (부서질 때 고주파 증가)
    nodes.filter.frequency.cancelScheduledValues(now);
    nodes.filter.frequency.setValueAtTime(200, now);
    nodes.filter.frequency.linearRampToValueAtTime(400, now + 3);
    nodes.filter.frequency.linearRampToValueAtTime(600, now + 3.5); // 부서지는 순간
    nodes.filter.frequency.linearRampToValueAtTime(200, now + 6);
    
    // 7~9초 후 다음 파도
    nodes.waveTimeout = setTimeout(
        () => scheduleWaves(nodes), 
        7000 + Math.random() * 2000
    );
}
```

**시각화**:
```
볼륨 │      ╭──╮
     │     ╱    ╲
     │    ╱      ╲
     │   ╱        ╲
     │──╱          ╲──────
     └─────────────────────▶ 시간
        0  2  3.5 4.5  7초
```

### 🐦 새소리 (Birds)

새소리는 **Oscillator**를 사용한 주파수 변조로 구현합니다:

```javascript
function scheduleBirdChirps(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    
    // 2~5개의 음표로 구성된 지저귐
    const numNotes = 2 + Math.floor(Math.random() * 4);
    let time = now;
    
    for (let n = 0; n < numNotes; n++) {
        const osc = audioContext.createOscillator();
        const chirpGain = audioContext.createGain();
        osc.type = 'sine';
        
        // 새 지저귐의 특징: 높은 주파수, 빠른 변화
        const baseFreq = 2500 + Math.random() * 2500;
        osc.frequency.setValueAtTime(baseFreq, time);
        osc.frequency.linearRampToValueAtTime(
            baseFreq + (Math.random() - 0.5) * 1000, 
            time + 0.08
        );
        osc.frequency.linearRampToValueAtTime(baseFreq - 500, time + 0.15);
        
        // ADSR 엔벨로프 (간단한 버전)
        chirpGain.gain.setValueAtTime(0, time);
        chirpGain.gain.linearRampToValueAtTime(0.25, time + 0.02);  // Attack
        chirpGain.gain.setValueAtTime(0.25, time + 0.08);           // Sustain
        chirpGain.gain.linearRampToValueAtTime(0, time + 0.15);     // Release
        
        osc.connect(chirpGain);
        chirpGain.connect(nodes.gainNode);
        osc.start(time);
        osc.stop(time + 0.2);
        
        time += 0.1 + Math.random() * 0.1; // 음표 간 간격
    }
    
    // 0.8~3.8초 후 다음 새
    nodes.birdTimeout = setTimeout(
        () => scheduleBirdChirps(nodes), 
        800 + Math.random() * 3000
    );
}
```

### ☕ 카페 (Cafe)

카페 소음은 **인간 목소리 시뮬레이션**으로 구현합니다:

```javascript
function scheduleCafeChatter(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    
    const voiceGain = audioContext.createGain();
    const osc1 = audioContext.createOscillator();
    const osc2 = audioContext.createOscillator();
    const filter = audioContext.createBiquadFilter();
    
    // 인간 목소리의 기본 주파수 범위
    // 남성: ~100Hz, 여성: ~200Hz
    const baseFreq = 100 + Math.random() * 150;
    
    // 톱니파는 풍부한 하모닉을 가짐 (목소리와 유사)
    osc1.type = 'sawtooth';
    osc1.frequency.value = baseFreq;
    osc2.type = 'sawtooth';
    osc2.frequency.value = baseFreq * 1.01; // 미세한 디튠으로 두께감 추가
    
    // 밴드패스 필터로 특정 주파수 대역만 통과
    filter.type = 'bandpass';
    filter.frequency.value = 500 + Math.random() * 1500;
    filter.Q.value = 2;
    
    // 연결
    osc1.connect(filter);
    osc2.connect(filter);
    filter.connect(voiceGain);
    voiceGain.connect(nodes.gainNode);
    
    // 짧은 "말하는" 패턴
    const duration = 0.1 + Math.random() * 0.3;
    voiceGain.gain.setValueAtTime(0, now);
    voiceGain.gain.linearRampToValueAtTime(0.06, now + 0.02);
    voiceGain.gain.setValueAtTime(0.06, now + duration - 0.02);
    voiceGain.gain.linearRampToValueAtTime(0, now + duration);
    
    osc1.start(now);
    osc1.stop(now + duration + 0.1);
    osc2.start(now);
    osc2.stop(now + duration + 0.1);
    
    nodes.cafeTimeout = setTimeout(
        () => scheduleCafeChatter(nodes), 
        100 + Math.random() * 400
    );
}
```

### ⌨️ 키보드 (Keyboard)

기계식 키보드의 "탁탁" 소리를 시뮬레이션합니다:

```javascript
function scheduleKeyboardClicks(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    
    const clickGain = audioContext.createGain();
    const noise = audioContext.createBufferSource();
    noise.buffer = createNoiseBuffer('white');
    
    // 키보드 클릭은 높은 주파수의 짧은 임펄스
    const hipass = audioContext.createBiquadFilter();
    hipass.type = 'highpass';
    hipass.frequency.value = 3000;
    
    const lopass = audioContext.createBiquadFilter();
    lopass.type = 'lowpass';
    lopass.frequency.value = 8000;
    
    noise.connect(hipass);
    hipass.connect(lopass);
    lopass.connect(clickGain);
    clickGain.connect(nodes.gainNode);
    
    // 매우 짧고 날카로운 엔벨로프
    clickGain.gain.setValueAtTime(0.3, now);           // 즉각적인 Attack
    clickGain.gain.setTargetAtTime(0.05, now, 0.005);  // 빠른 1차 감쇠
    clickGain.gain.setTargetAtTime(0, now + 0.02, 0.01); // 완전 감쇠
    
    noise.start(now);
    noise.stop(now + 0.05);
    
    // 타이핑 속도 시뮬레이션 (가끔 멈춤)
    const delay = Math.random() < 0.1 
        ? 300 + Math.random() * 500  // 10% 확률로 긴 멈춤
        : 60 + Math.random() * 140;   // 일반 타이핑 속도
    
    nodes.keyTimeout = setTimeout(
        () => scheduleKeyboardClicks(nodes), 
        delay
    );
}
```

### 🦗 귀뚜라미 (Crickets)

귀뚜라미는 **고주파 사인파의 빠른 펄스**로 구현합니다:

```javascript
function scheduleCrickets(nodes) {
    if (!nodes.active) return;
    const now = audioContext.currentTime;
    
    // 1~2마리의 귀뚜라미
    const numChirps = 1 + Math.floor(Math.random() * 2);
    
    for (let c = 0; c < numChirps; c++) {
        const osc = audioContext.createOscillator();
        const cricketGain = audioContext.createGain();
        
        osc.type = 'sine';
        const freq = 3500 + Math.random() * 1500; // 3.5~5kHz
        osc.frequency.value = freq;
        
        osc.connect(cricketGain);
        cricketGain.connect(nodes.gainNode);
        
        // 빠른 펄스 패턴 (4~6회)
        const pulses = 4 + Math.floor(Math.random() * 3);
        const startTime = now + c * 0.3;
        
        cricketGain.gain.setValueAtTime(0, startTime);
        for (let i = 0; i < pulses; i++) {
            const t = startTime + i * 0.05;
            cricketGain.gain.setValueAtTime(0.15, t);      // ON
            cricketGain.gain.setValueAtTime(0, t + 0.025); // OFF
        }
        
        osc.start(startTime);
        osc.stop(startTime + pulses * 0.05 + 0.1);
    }
    
    nodes.cricketTimeout = setTimeout(
        () => scheduleCrickets(nodes), 
        300 + Math.random() * 1200
    );
}
```

**펄스 패턴 시각화**:
```
볼륨 │ █ █ █ █ █     █ █ █ █
     │ █ █ █ █ █     █ █ █ █
     └─────────────────────────▶ 시간
       ├─50ms─┤
```

---

## 오디오 노드 그래프

### 전체 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                        AudioContext                              │
│                                                                  │
│  ┌──────────────┐                                               │
│  │ Rain Source  │──┐                                            │
│  └──────────────┘  │    ┌────────────┐                          │
│                    ├───▶│ Rain Gain  │──┐                       │
│  ┌──────────────┐  │    └────────────┘  │                       │
│  │Rain Filter   │──┘                    │                       │
│  └──────────────┘                       │                       │
│                                         │    ┌──────────────┐   │
│  ┌──────────────┐    ┌────────────┐    ├───▶│ Master Gain  │──▶ 🔊
│  │ Fire Source  │───▶│ Fire Gain  │────┤    └──────────────┘   │
│  └──────────────┘    └────────────┘    │                       │
│        ↓                               │                       │
│  ┌──────────────┐                      │                       │
│  │ Crackle Osc  │──────────────────────┤                       │
│  └──────────────┘                      │                       │
│                                         │                       │
│  ┌──────────────┐    ┌────────────┐    │                       │
│  │ Bird Osc     │───▶│ Bird Gain  │────┘                       │
│  └──────────────┘    └────────────┘                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 노드 저장 구조

```javascript
const soundNodes = {
    rain: {
        active: true,
        source: BufferSourceNode,
        filter: BiquadFilterNode,
        gainNode: GainNode,
        // ... 타임아웃 ID들
    },
    fire: {
        active: true,
        source: BufferSourceNode,
        filter: BiquadFilterNode,
        gainNode: GainNode,
        crackleTimeout: Number,
    },
    // ...
};
```

---

## 스케줄링과 타이밍

### Web Audio의 정밀한 타이밍

Web Audio API는 `audioContext.currentTime`을 사용하여 **샘플 정확도**의 타이밍을 제공합니다:

```javascript
const now = audioContext.currentTime; // 초 단위 (float)

// 정확한 시간에 값 설정
gainNode.gain.setValueAtTime(1.0, now);

// 선형 보간 (Linear Ramp)
gainNode.gain.linearRampToValueAtTime(0.0, now + 2);

// 지수 감쇠 (Exponential)
gainNode.gain.setTargetAtTime(0.0, now, 0.3); // 시간 상수 0.3초
```

### setTimeout vs Web Audio 스케줄링

```javascript
// ❌ 부정확: JavaScript 이벤트 루프에 의존
setTimeout(() => {
    oscillator.start();
}, 1000);

// ✅ 정확: 오디오 스레드에서 처리
oscillator.start(audioContext.currentTime + 1);
```

하지만 **반복적인 효과**(새소리, 귀뚜라미 등)는 `setTimeout`을 사용합니다. 왜냐하면:
1. 정확한 타이밍보다 자연스러운 무작위성이 더 중요
2. 메모리 관리가 용이
3. 취소/수정이 쉬움

---

## 상태 관리

### 사운드 상태

```javascript
let soundVolumes = {};  // 각 사운드의 볼륨 (0-100)
let soundNodes = {};    // 실행 중인 오디오 노드들
let isPlaying = false;  // 전체 재생 상태
```

### 사운드 시작

```javascript
function startSound(id, type, volume) {
    if (!audioContext) initAudioContext();
    
    // 기존 사운드 정리
    if (soundNodes[id]) {
        stopSound(id);
    }
    
    // 새 사운드 생성
    const nodes = createSound(type);
    soundNodes[id] = nodes;
    
    // 파도는 특별 처리 (자체 볼륨 변조)
    if (type === 'waves') {
        nodes.targetVolume = volume / 100;
        scheduleWaves(nodes);
    } else {
        nodes.gainNode.gain.setTargetAtTime(volume / 100, audioContext.currentTime, 0.1);
    }
}
```

### 사운드 정지

```javascript
function stopSound(id) {
    const nodes = soundNodes[id];
    if (nodes) {
        nodes.active = false;
        
        // 모든 스케줄된 타임아웃 취소
        clearTimeout(nodes.thunderTimeout);
        clearTimeout(nodes.crackleTimeout);
        // ... 기타 타임아웃들
        
        // 부드러운 페이드아웃
        nodes.gainNode.gain.cancelScheduledValues(audioContext.currentTime);
        nodes.gainNode.gain.setTargetAtTime(0, audioContext.currentTime, 0.3);
        
        // 소스 정지
        if (nodes.source) {
            try { 
                nodes.source.stop(audioContext.currentTime + 0.5); 
            } catch(e) {}
        }
        
        // 클린업
        setTimeout(() => {
            if (nodes.gainNode) nodes.gainNode.disconnect();
            delete soundNodes[id];
        }, 600);
    }
}
```

### 프리셋 시스템

```javascript
const DEFAULT_PRESETS = [
    { 
        id: 'focus', 
        name: '집중 모드', 
        emoji: '📚', 
        sounds: { rain: 40, brownnoise: 30 } 
    },
    // ...
];

function applyPreset(presetId) {
    const preset = presets.find(p => p.id === presetId);
    
    SOUNDS.forEach(sound => {
        const volume = preset.sounds[sound.id] || 0;
        soundVolumes[sound.id] = volume;
        
        // UI 업데이트
        document.getElementById(`range-${sound.id}`).value = volume;
        
        // 재생 중이면 사운드 시작/정지
        if (isPlaying) {
            if (volume > 0) {
                startSound(sound.id, sound.type, volume);
            } else {
                stopSound(sound.id);
            }
        }
    });
}
```

---

## 성능 최적화

### 1. 노이즈 버퍼 재사용

```javascript
// 🔴 비효율: 매번 새 버퍼 생성
function scheduleCrackles(nodes) {
    const noise = audioContext.createBufferSource();
    noise.buffer = createNoiseBuffer('white'); // 매번 생성!
    // ...
}

// 🟢 개선: 버퍼 캐싱 (구현 가능)
const noiseBuffers = {};
function getNoiseBuffer(type) {
    if (!noiseBuffers[type]) {
        noiseBuffers[type] = createNoiseBuffer(type);
    }
    return noiseBuffers[type];
}
```

### 2. 노드 연결 해제

사용이 끝난 노드는 반드시 `disconnect()`를 호출하여 가비지 컬렉션이 되도록 합니다:

```javascript
setTimeout(() => {
    if (nodes.gainNode) nodes.gainNode.disconnect();
    delete soundNodes[id];
}, 600);
```

### 3. 스케줄링 최적화

`setTimeout`을 사용할 때 `active` 플래그를 확인하여 불필요한 연산을 방지:

```javascript
function scheduleBirdChirps(nodes) {
    if (!nodes.active) return; // 조기 종료
    // ...
}
```

### 4. 메모리 누수 방지

모든 타임아웃 ID를 노드 객체에 저장하고, 정지 시 모두 취소:

```javascript
// 저장
nodes.birdTimeout = setTimeout(() => scheduleBirdChirps(nodes), delay);

// 정지 시 취소
clearTimeout(nodes.birdTimeout);
```

---

## 마치며

이 프로젝트는 Web Audio API의 강력함을 보여주는 좋은 예시입니다. 단일 HTML 파일로:

- ✅ 12가지 다양한 환경음 생성
- ✅ 완전한 오프라인 동작
- ✅ 외부 오디오 파일 불필요
- ✅ 무한 재생 (루프 끊김 없음)

### 더 발전시킬 수 있는 부분

1. **Convolution Reverb**: 공간감 추가
2. **Binaural Beats**: 집중/수면 효과
3. **Granular Synthesis**: 더 자연스러운 비/바람 소리
4. **Service Worker**: PWA로 완전한 오프라인 지원

### 참고 자료

- [MDN Web Audio API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Audio_API)
- [Web Audio API 명세](https://www.w3.org/TR/webaudio/)
- [Noise 생성 알고리즘](https://noisehack.com/generate-noise-web-audio-api/)

---

*작성일: 2026년 1월*

