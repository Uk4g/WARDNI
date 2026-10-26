# Architecture — detailed graphs

## System graph

```mermaid
flowchart TB
    subgraph Room["In the room — simulated for demo"]
        CAM[Low-light / IR camera]
        MIC[Intercom microphone]
        VITALS[Mock vitals<br/>HR / SpO2 / profile]
    end

    subgraph Models["Meta models — live, via backend proxy"]
        SAM[SAM 3.1<br/>segmentation + pose]
        VOICE[Muse Voice Transcribe<br/>speech to text + intent]
        SPARK[Muse Spark 1.3<br/>triage + handoff]
    end

    subgraph Product["Dashboard — React + Vite, static build"]
        DASH[Ward grid<br/>6 rooms, colours, timeline]
        WF[Wireframe view<br/>NO raw video]
        LOG[Event log<br/>single source of truth]
        HANDOFF[Morning handoff<br/>generated from log]
    end

    CAM --> SAM
    SAM --> WF
    MIC --> VOICE
    VOICE --> DASH
    VITALS --> SPARK
    SAM --> SPARK
    DASH --> SPARK
    SPARK --> DASH
    DASH --> LOG
    LOG --> HANDOFF
    DASH --> NURSE[Nurse<br/>acknowledge / turn done / decide]
```

## Adapter interfaces — same shape, simulated and live

Every adapter has `MODE = 'simulated' | 'live'`. Same input, same output, so the
dashboard works with no key, and falls back to simulated if a live call fails.

```js
// src/ai/vision.js
detectVision(room, frame) // -> { pose: 'lying'|'legs-over-rail'|'sitting', line: 'secure'|'tugged', event: null | {type, priority} }

// src/ai/voice.js
transcribe(audio) // -> { text: string, intent: 'pain'|'comfort'|'routine'|'other' }

// src/ai/reasoning.js
triage(request, profile, vitals) // -> { priority: 'RED'|'YELLOW'|'GREEN', reason: string, nextAction: string }
makeHandoff(log) // -> string
```

## Night flow — state graph

```mermaid
stateDiagram-v2
    [*] --> Stable: night starts, all green
    Stable --> TurnDue: turn timer >= 120 min
    Stable --> BedExitRisk: SAM: sitting up, legs over rail
    Stable --> VoiceRequest: intercom transcript
    VoiceRequest --> Red: pain + post-op / urgent sign
    VoiceRequest --> Yellow: comfort request
    VoiceRequest --> AutoAnswered: routine question
    Stable --> LineTug: SAM: line flagged tugged
    TurnDue --> TurnDone: nurse marks turn done
    BedExitRisk --> Acknowledged: nurse acknowledges
    Red --> Acknowledged: nurse acknowledges, walks first
    Acknowledged --> Stable: resolved, logged
    TurnDone --> Stable: timer reset, logged
```

## Privacy graph

```mermaid
flowchart LR
    RAW[Raw video<br/>in room only] -->|segmented, not streamed| WIRE[Wireframe<br/>nurse view]
    WIRE -->|no faces, no names| DASH[Dashboard tile<br/>Room number + pose + line]
    DASH -->|fictional only| LOG[Event log<br/>Room + time + event]
```

No raw video, no real names, no real patient data leave the simulated room.
