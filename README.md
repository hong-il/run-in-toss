# AppInToss Running Tracker (run-in-toss)

이 프로젝트는 AppInToss SDK와 React, TypeScript, Tailwind CSS를 활용하여 제작한 토스 디자인 시스템(TDS) 스타일의 실시간 러닝 추적 웹앱입니다.

---

## 1. 프로젝트 초기화 및 의존성 설치

터미널에서 아래 명령어를 순서대로 실행하여 프로젝트를 생성하고 필요한 라이브러리를 설치합니다.

```bash
# 프로젝트 생성 (Vite + React + TS)
npm create vite@latest run-in-toss -- --template react-ts
cd run-in-toss

# Tailwind CSS 및 PostCSS 설치
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# 필수 라이브러리 설치
npm install lucide-react react-calendar date-fns react-kakao-maps-sdk @apps-in-toss/web-framework
npm install -D @types/react-calendar
```

---

## 2. 프로젝트 디렉토리 구조

`src` 폴더 내부를 아래와 같이 구성합니다.

```text
src/
├── components/
│   ├── RunningMap.tsx
│   └── ResultShareCard.tsx
├── hooks/
│   └── useRunningTracker.ts
├── lib/
│   └── appInToss/
│       └── index.ts
├── pages/
│   ├── HistoryPage.tsx
│   └── RankingPage.tsx
├── types/
│   ├── running.ts
│   └── global.d.ts
├── utils/
│   ├── calculators.ts
│   ├── storage.ts
│   └── share.ts
├── App.tsx
├── index.css
└── main.tsx
```

---

## 3. 전체 소스코드

### 3.1. 설정 및 진입점

**tailwind.config.js**
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
        toss: {
          blue: '#3182f6', blueHover: '#1b64da', lightBlue: '#e8f3ff',
          gray100: '#f2f4f6', gray200: '#e5e8eb', gray500: '#8b95a1',
          gray800: '#333d4b', gray900: '#191f28',
        }
      },
      borderRadius: { '3xl': '1.5rem' },
      fontFamily: { sans: ['Pretendard', 'sans-serif'] }
    },
  },
  plugins: [],
}
```

**index.html**
```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
    <title>Run In Toss</title>
    <script type="text/javascript" src="//[dapi.kakao.com/v2/maps/sdk.js?appkey=YOUR_KAKAO_JS_KEY&libraries=services](https://dapi.kakao.com/v2/maps/sdk.js?appkey=YOUR_KAKAO_JS_KEY&libraries=services)"></script>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

**src/index.css**
```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  body { @apply bg-toss-gray100 text-toss-gray800 antialiased; }
}

.react-calendar { width: 100%; border: none !important; background: white; }
.react-calendar__tile { padding: 12px 6px; border-radius: 12px; }
.react-calendar__tile--active { background-color: #3182f6 !important; color: white !important; }
```

### 3.2. 타입 및 유틸리티

**src/types/running.ts**
```typescript
export interface Coordinate { lat: number; lng: number; }
export interface RunRecord {
  id: string; date: string; distance: number; time: number; pace: number; path: Coordinate[];
}
```

**src/types/global.d.ts**
```typescript
export {};
declare global {
  interface Window {
    AppInToss?: {
      haptic: (type: 'light' | 'medium' | 'heavy') => void;
      requestPermission: (type: 'LOCATION') => Promise<{ granted: boolean }>;
    };
  }
}
```

**src/utils/calculators.ts**
```typescript
export const calculateDistance = (lat1: number, lon1: number, lat2: number, lon2: number): number => {
  const R = 6371e3;
  const dLat = (lat2 - lat1) * Math.PI / 180;
  const dLon = (lon2 - lon1) * Math.PI / 180;
  const a = Math.sin(dLat/2)**2 + Math.cos(lat1*Math.PI/180)*Math.cos(lat2*Math.PI/180)*Math.sin(dLon/2)**2;
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
};

export const formatTime = (s: number) => `${Math.floor(s/60).toString().padStart(2,'0')}:${(s%60).toString().padStart(2,'0')}`;
export const formatDistance = (m: number) => (m / 1000).toFixed(2);
export const formatPace = (sPerKm: number) => {
  if (!sPerKm || !isFinite(sPerKm)) return "-'--\"";
  const m = Math.floor(sPerKm / 60);
  return `${m}'${Math.floor(sPerKm % 60).toString().padStart(2,'0')}"`;
};
```

**src/utils/storage.ts**
```typescript
import { RunRecord } from '../types/running';
const KEY = 'toss_run_data';
export const getRecords = (): RunRecord[] => JSON.parse(localStorage.getItem(KEY) || '[]');
export const saveRecord = (r: RunRecord) => localStorage.setItem(KEY, JSON.stringify([...getRecords(), r]));
```

### 3.3. SDK 및 커스텀 훅

**src/lib/appInToss/index.ts**
```typescript
export const AppInToss = {
  init: () => window.AppInToss ? console.log('SDK Ready') : console.warn('Web Mode'),
  haptic: (t: 'light' | 'medium' | 'heavy') => window.AppInToss?.haptic(t),
  requestLocation: async () => (await window.AppInToss?.requestPermission('LOCATION'))?.granted ?? true,
};
```

**src/hooks/useRunningTracker.ts**
```typescript
import { useState, useRef, useCallback } from 'react';
import { calculateDistance } from '../utils/calculators';
import { Coordinate } from '../types/running';

export const useRunningTracker = () => {
  const [isRunning, setIsRunning] = useState(false);
  const [elapsed, setElapsed] = useState(0);
  const [distance, setDistance] = useState(0);
  const [path, setPath] = useState<Coordinate[]>([]);
  const watchId = useRef<number | null>(null);
  const timerId = useRef<number | null>(null);
  const lastLoc = useRef<Coordinate | null>(null);

  const start = () => {
    setElapsed(0); setDistance(0); setPath([]); setIsRunning(true);
    timerId.current = window.setInterval(() => setElapsed(e => e + 1), 1000);
    watchId.current = navigator.geolocation.watchPosition((p) => {
      const { latitude: lat, longitude: lng, accuracy } = p.coords;
      if (accuracy > 30) return;
      if (lastLoc.current) {
        const d = calculateDistance(lastLoc.current.lat, lastLoc.current.lng, lat, lng);
        if (d > 2) {
          setDistance(prev => prev + d);
          setPath(prev => [...prev, { lat, lng }]);
        }
      } else { setPath([{ lat, lng }]); }
      lastLoc.current = { lat, lng };
    }, undefined, { enableHighAccuracy: true });
  };

  const stop = () => {
    if (watchId.current) navigator.geolocation.clearWatch(watchId.current);
    if (timerId.current) clearInterval(timerId.current);
    setIsRunning(false);
  };

  return { isRunning, elapsed, distance, path, start, stop };
};
```

### 3.4. 컴포넌트 및 메인 뷰

**src/components/RunningMap.tsx**
```tsx
import { Map, Polyline, MapMarker } from "react-kakao-maps-sdk";
import { Coordinate } from "../types/running";

export default function RunningMap({ path }: { path: Coordinate[] }) {
  if (!path.length) return <div className="h-full bg-toss-gray200 flex items-center justify-center">데이터 없음</div>;
  return (
    <Map center={path[0]} style={{ width: "100%", height: "100%" }} level={3}>
      <Polyline path={[path]} strokeWeight={5} strokeColor="#3182f6" />
      <MapMarker position={path[0]} /><MapMarker position={path[path.length - 1]} />
    </Map>
  );
}
```

**src/App.tsx**
```tsx
import { useState, useEffect } from 'react';
import { Play, Square, MapPin, Calendar, Trophy } from 'lucide-react';
import { useRunningTracker } from './hooks/useRunningTracker';
import { formatTime, formatDistance, formatPace } from './utils/calculators';
import { saveRecord } from './utils/storage';
import { AppInToss } from './lib/appInToss';
import HistoryPage from './pages/HistoryPage';
import RankingPage from './pages/RankingPage';

export default function App() {
  const [tab, setTab] = useState<'run' | 'history' | 'rank'>('run');
  const { isRunning, elapsed, distance, path, start, stop } = useRunningTracker();

  useEffect(() => { AppInToss.init(); AppInToss.requestLocation(); }, []);

  const handleStop = () => {
    stop();
    if (distance > 10) {
      saveRecord({ id: Date.now().toString(), date: new Date().toISOString(), distance, time: elapsed, pace: (elapsed/distance)*1000, path });
    }
  };

  return (
    <div className="min-h-screen bg-toss-gray100 font-sans pb-24 text-toss-gray800">
      {tab === 'run' && (
        <main className="p-6 pt-12 flex flex-col items-center">
          <div className="w-full max-w-md bg-white rounded-3xl p-8 shadow-sm">
            <p className="text-center text-toss-gray500 text-sm mb-2 font-medium">달린 거리 (km)</p>
            <p className="text-center text-6xl font-black mb-10 tabular-nums">{formatDistance(distance)}</p>
            <div className="grid grid-cols-2 gap-4 pt-6 border-t border-toss-gray100">
              <div className="text-center"><p className="text-xs text-toss-gray500 mb-1">페이스</p><p className="text-2xl font-bold text-toss-blue">{formatPace((elapsed/distance)*1000)}</p></div>
              <div className="text-center border-l border-toss-gray100"><p className="text-xs text-toss-gray500 mb-1">시간</p><p className="text-2xl font-bold">{formatTime(elapsed)}</p></div>
            </div>
          </div>
          <div className="mt-10 w-full max-w-md">
            {!isRunning ? (
              <button onClick={() => { AppInToss.haptic('medium'); start(); }} className="w-full bg-toss-blue text-white py-5 rounded-2xl font-bold text-xl flex justify-center gap-3 active:scale-95 transition-transform"><Play fill="white"/> 러닝 시작</button>
            ) : (
              <button onClick={() => { AppInToss.haptic('heavy'); handleStop(); }} className="w-full bg-toss-gray900 text-white py-5 rounded-2xl font-bold text-xl flex justify-center gap-3 active:scale-95 transition-transform"><Square fill="white"/> 운동 종료</button>
            )}
          </div>
        </main>
      )}
      {tab === 'history' && <HistoryPage />}
      {tab === 'rank' && <RankingPage userDistance={distance} />}

      <nav className="fixed bottom-0 w-full bg-white/80 backdrop-blur-md border-t flex justify-around py-3 px-2">
        {[ {id:'history', i:Calendar, l:'기록'}, {id:'run', i:MapPin, l:'러닝'}, {id:'rank', i:Trophy, l:'랭킹'} ].map(t => (
          <button key={t.id} onClick={() => setTab(t.id as any)} className={`flex flex-col items-center gap-1 ${tab === t.id ? 'text-toss-blue' : 'text-toss-gray500'}`}><t.i size={24}/><span className="text-[10px] font-bold">{t.l}</span></button>
        ))}
      </nav>
    </div>
  );
}
```

---

## 4. 실행 및 배포 가이드

1. **API 키 설정**: `index.html` 내 `YOUR_KAKAO_JS_KEY`를 실제 카카오 JavaScript 키로 교체합니다.
2. **로컬 테스트**: `npm run dev` 명령어를 통해 실행 후 위치 권한을 승인합니다.
3. **실기기 배포**: AppInToss SDK의 네이티브 기능(햅틱, 공유 등)을 확인하려면 HTTPS 환경으로 배포한 뒤 토스 파트너 센터에 등록한 웹뷰 URL을 통해 접속하면 됩니다.
