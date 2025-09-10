# 📦 `resumable-background-downloader`

> *A React-first npm package for resumable, parallel, chunked file downloads — with experimental background persistence using Service Workers (Chrome-only), and fallback to Web Workers + IndexedDB for all browsers.*

[![npm version](https://img.shields.io/npm/v/resumable-background-downloader.svg)](https://www.npmjs.com/package/resumable-background-downloader)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## 🎯 Goal

Enable large file downloads in React apps with:

✅ **Chunked downloads** (split file into byte ranges)  
✅ **Parallel downloads** (multiple chunks simultaneously via Web Workers)  
✅ **Retry on failure** (exponential backoff)  
✅ **Resumable** (save progress to IndexedDB, resume later)  
✅ **Survives tab refresh**  
✅ **Experimental: Survives tab close** — via Chrome’s `Background Fetch API`  
✅ **React Hooks API** — simple, declarative usage  
✅ **Auto browser detection** — uses best available strategy  
🚧 **Future**: Tauri/Electron bridge for true native background

---

## ⚠️ Critical Limitations

> ❗ **No browser can reliably continue downloading after tab close — except Chrome/Edge using experimental Background Fetch API.**

- **Web Workers die** when tab closes.
- **Service Workers** are short-lived and event-driven — not meant for long-running chunked downloads.
- **Background Fetch API** (Chrome-only):
  - ✅ Survives tab close
  - ✅ Auto-retry built-in
  - ❌ Cannot control chunking or headers (`Range`, etc.)
  - ❌ Cannot run multiple parallel requests

> 🔄 Your package will **auto-fallback**:
> - Chrome → `BackgroundFetchStrategy` (whole file, survives close)
> - Others → `ChunkedWorkerStrategy` (parallel chunks, dies on close, but resumable)

---

## 🧱 Architecture

```
src/
├── index.ts                          # Public React hook + main API
├── strategies/
│   ├── BackgroundFetchStrategy.ts    # Chrome-only: uses Background Fetch API
│   └── ChunkedWorkerStrategy.ts      # All browsers: Web Workers + Range requests
├── utils/
│   ├── chunker.ts                    # Splits file into byte ranges
│   ├── storage.ts                    # Saves/resumes chunks via IndexedDB
│   ├── retry.ts                      # Retry with exponential backoff
│   └── browserDetect.ts              # Detects Background Fetch support
├── worker/
│   └── downloader.worker.ts          # Web Worker: downloads one chunk
├── sw/
│   └── sw.ts                         # Service Worker: handles Background Fetch events
├── types.ts                          # TypeScript interfaces
└── constants.ts                      # Configurable defaults (chunkSize, retries, etc.)
```

---

## 🚀 Public API (React Hook)

```tsx
import { useBackgroundDownloader } from 'resumable-background-downloader';

function App() {
  const { startDownload, resumeDownload, getProgress, pauseDownload } = useBackgroundDownloader();

  const handleStart = async () => {
    const downloadId = await startDownload(
      'https://example.com/largefile.zip',
      {
        fileName: 'largefile.zip',
        chunkSize: 1024 * 1024, // 1MB chunks
        parallel: 4,            // 4 chunks in parallel
        maxRetries: 3,
      }
    );
    console.log('Download started with ID:', downloadId);
  };

  const progress = getProgress(downloadId); // { loaded, total, percent }

  return (
    <div>
      <button onClick={handleStart}>Start Download</button>
      {progress && <div>{progress.percent}%</div>}
    </div>
  );
}
```

---

## 🔄 Strategies

### 1. `BackgroundFetchStrategy` (Chrome/Edge only)

- Uses `navigator.serviceWorker.ready` + `backgroundFetch.fetch()`
- Downloads whole file — no chunking or parallelism
- Survives tab close ✅
- Auto-retry ✅
- Progress via notification or SW events
- Stores final blob in IndexedDB on success

### 2. `ChunkedWorkerStrategy` (All browsers)

- Splits file into ranges using `Content-Range`
- Spawns 1 Web Worker per chunk (configurable parallelism)
- Each worker downloads its range with `fetch + Range header`
- Saves each chunk to IndexedDB
- On failure → retry with backoff
- On resume → load existing chunks, skip completed ranges
- Dies if tab closes — but user can resume later

---

## 💾 Storage: IndexedDB Schema

```ts
interface DownloadChunk {
  downloadId: string;
  chunkIndex: number;
  startByte: number;
  endByte: number;
  buffer: ArrayBuffer;
  status: 'pending' | 'success' | 'failed';
}

interface DownloadMeta {
  id: string;
  url: string;
  fileName: string;
  totalSize: number;
  chunkSize: number;
  createdAt: Date;
  updatedAt: Date;
  status: 'active' | 'paused' | 'completed' | 'failed';
}
```

> ✅ On app restart → check `DownloadMeta` → resume if `status === 'active'`

---

## ♻️ Retry Logic

```ts
// utils/retry.ts
export const withRetry = async <T>(
  fn: () => Promise<T>,
  retries: number = 3,
  baseDelay: number = 1000
): Promise<T> => {
  for (let i = 0; i < retries; i++) {
    try {
      return await fn();
    } catch (err) {
      if (i === retries - 1) throw err;
      await new Promise(resolve => setTimeout(resolve, baseDelay * Math.pow(2, i)));
    }
  }
};
```

Applied per chunk in `ChunkedWorkerStrategy`.

---

## 🛠️ Service Worker (Background Fetch Handlers)

```ts
// sw/sw.ts

self.addEventListener('backgroundfetchsuccess', (event: any) => {
  event.waitUntil(handleBackgroundFetchSuccess(event));
});

async function handleBackgroundFetchSuccess(event: any) {
  const bgFetch = event.backgroundFetch;
  const records = await bgFetch.matchAll();
  const response = records[0].response;
  const blob = await response.blob();

  // Save to IndexedDB
  await saveFileToIDB(event.id, blob);

  // Send success notification
  self.registration.showNotification('Download Complete', {
    body: `File saved successfully.`,
  });
}

self.addEventListener('backgroundfetchfail', (event: any) => {
  // Built-in retry — no action needed unless you want custom logic
  console.warn('Background fetch failed:', event.id);
});
```

> ⚠️ Must be registered in main thread:

```ts
// index.ts (during init)
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/sw.js');
}
```

---

## 👷 Web Worker (Chunk Downloader)

```ts
// worker/downloader.worker.ts

self.addEventListener('message', async (e: MessageEvent) => {
  const { url, start, end, chunkId, downloadId } = e.data;

  try {
    const response = await fetch(url, {
      headers: {
        Range: `bytes=${start}-${end}`,
      },
    });

    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    const buffer = await response.arrayBuffer();

    self.postMessage({
      type: 'chunk-success',
      chunkId,
      downloadId,
      buffer,
      start,
      end,
    });
  } catch (error) {
    self.postMessage({
      type: 'chunk-error',
      chunkId,
      downloadId,
      error: error.message,
    });
  }
});
```

---

## 📊 Progress Tracking

```ts
// getProgress(downloadId: string) → { loaded: number, total: number, percent: number }

// Calculated by:
// loaded = sum of all downloaded chunk sizes
// total = file size (from HEAD request or Content-Range)
```

---

## 🧪 Browser Support Detection

```ts
// utils/browserDetect.ts

export const isBackgroundFetchSupported = (): boolean => {
  return (
    typeof navigator !== 'undefined' &&
    'serviceWorker' in navigator &&
    'backgroundFetch' in Registration.prototype
  );
};
```

---

## 🚧 Future Roadmap

| Feature                        | Status      |
|-------------------------------|-------------|
| Tauri/Electron Bridge         | Planned     |
| Pause/Resume UI Component     | Planned     |
| Download Queue Management     | Planned     |
| Bandwidth Throttling Option   | Planned     |
| Support Range in Background Fetch | Experimental / Not Possible Yet |
| Safari/Firefox Background     | ❌ Not Possible |

---

## 📦 Installation

```bash
npm install resumable-background-downloader
```

---

## 🏗️ Development Setup

```bash
git clone https://github.com/yourname/resumable-background-downloader
cd resumable-background-downloader
npm install
npm run dev
```

> Includes Vite + React dev server with SW and Worker bundling.

---

## 🧪 Testing

Use Chrome for Background Fetch testing. Use Firefox/Safari to test fallback ChunkedWorkerStrategy.

```bash
npm run test
npm run build
```

---

## 🤝 Contributing

Contributions welcome! Especially:

- Better IndexedDB abstraction
- UI progress component
- Tauri bridge
- Documentation & demos

---

## 📄 License

MIT

---