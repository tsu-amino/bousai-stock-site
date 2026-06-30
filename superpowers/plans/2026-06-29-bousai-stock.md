# 防災ストック管理アプリ Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a fully offline disaster-preparedness stock management app for iOS/Android using Expo + TypeScript that tracks expiry dates and fires local push notifications.

**Architecture:** expo-router handles file-based navigation with a tab bar root; expo-sqlite stores all data locally in a single `bousai.db`; Zustand holds in-memory mirrors of DB state for reactive UI updates; services/notifications.ts and services/photos.ts encapsulate side-effect logic.

**Tech Stack:** Expo SDK (latest), expo-router, expo-sqlite, expo-notifications, expo-image-picker, expo-file-system, Zustand, dayjs, TypeScript

---

## File Map

```
bousai-stock/
├── app/
│   ├── _layout.tsx                    # Root Stack (expo-router entry)
│   ├── (tabs)/
│   │   ├── _layout.tsx                # Bottom tab bar
│   │   ├── index.tsx                  # Home: category summary
│   │   ├── list.tsx                   # All items: search + filter
│   │   ├── notifications.tsx          # Notification list + toggle
│   │   └── settings.tsx              # Storage location CRUD
│   ├── item/
│   │   ├── new.tsx                    # New item form
│   │   └── [id]/
│   │       ├── index.tsx              # Item detail
│   │       └── edit.tsx              # Item edit form
│   └── category/
│       └── [id].tsx                   # Category item list
├── src/
│   ├── db/
│   │   ├── database.ts                # openDatabaseSync + initDb()
│   │   ├── schema.ts                  # SQL CREATE TABLE strings
│   │   ├── seed.ts                    # categories + storage_locations seed
│   │   └── queries/
│   │       ├── items.ts               # Item CRUD
│   │       ├── categories.ts          # getCategories()
│   │       └── locations.ts           # StorageLocation CRUD
│   ├── domain/
│   │   ├── types.ts                   # TypeScript types
│   │   └── expiry.ts                  # Pure expiry helpers (tested)
│   ├── store/
│   │   └── useAppStore.ts             # Zustand store
│   ├── services/
│   │   ├── notifications.ts           # schedule/cancel/sync notifications
│   │   └── photos.ts                  # savePhoto / deletePhoto
│   ├── theme/
│   │   └── tokens.ts                  # Colors, Radius, Spacing, FontSize
│   └── components/
│       ├── ExpiryBadge.tsx            # Colored pill: expired/soon/ok
│       ├── MetricCard.tsx             # Summary count card
│       ├── AlertBanner.tsx            # Urgent single-item warning
│       ├── CategoryCard.tsx           # Home category row card
│       ├── ItemCard.tsx               # List item card with thumbnail
│       └── ItemForm.tsx               # Shared add/edit form (used by new + edit screens)
├── app.json
├── package.json
├── tsconfig.json
└── jest.config.js
```

---

## Task 1: Project Scaffold

**Files:**
- Create: `app.json`
- Create: `package.json` (via create-expo-app)
- Create: `tsconfig.json`
- Create: `jest.config.js`

- [ ] **Step 1: Create Expo project**

```bash
cd /Users/a/develop/kraynote
npx create-expo-app bousai-stock --template blank-typescript
cd bousai-stock
```

- [ ] **Step 2: Install required packages**

```bash
npx expo install expo-router expo-sqlite expo-notifications expo-image-picker expo-file-system react-native-safe-area-context react-native-screens
npm install zustand dayjs
npm install --save-dev ts-jest @types/jest
```

- [ ] **Step 3: Configure expo-router in app.json**

Replace contents of `app.json`:

```json
{
  "expo": {
    "name": "防災ストック",
    "slug": "bousai-stock",
    "version": "1.0.0",
    "orientation": "portrait",
    "userInterfaceStyle": "light",
    "scheme": "bousai-stock",
    "icon": "./assets/icon.png",
    "ios": {
      "supportsTablet": false,
      "infoPlist": {
        "NSCameraUsageDescription": "アイテムの写真を撮影するために使用します",
        "NSPhotoLibraryUsageDescription": "ライブラリから写真を選択するために使用します"
      }
    },
    "android": {
      "adaptiveIcon": {
        "foregroundImage": "./assets/adaptive-icon.png",
        "backgroundColor": "#6B7C47"
      }
    },
    "plugins": [
      "expo-router",
      "expo-sqlite",
      [
        "expo-notifications",
        {
          "icon": "./assets/notification-icon.png",
          "color": "#6B7C47"
        }
      ],
      [
        "expo-image-picker",
        {
          "photosPermission": "ライブラリから写真を選択するために使用します",
          "cameraPermission": "アイテムの写真を撮影するために使用します"
        }
      ]
    ]
  }
}
```

- [ ] **Step 4: Configure tsconfig.json**

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "types": ["jest"],
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

- [ ] **Step 5: Configure jest.config.js**

```js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  moduleFileExtensions: ['ts', 'tsx', 'js'],
  testMatch: ['**/*.test.ts', '**/*.test.tsx'],
  transform: {
    '^.+\\.(ts|tsx)$': ['ts-jest', { tsconfig: { jsx: 'react' } }],
  },
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
  },
};
```

- [ ] **Step 6: Verify expo starts**

```bash
npx expo start --no-dev
```

Expected: Metro bundler starts without errors. Press `q` to quit.

- [ ] **Step 7: Create placeholder assets**

```bash
mkdir -p assets
# Copy icon.png placeholder (use any 1024x1024 PNG, or generate one)
# For now copy from vaccine_tracker if available:
cp /Users/a/develop/kraynote/vaccine_tracker/app/assets/icon.png assets/icon.png 2>/dev/null || \
  curl -s "https://via.placeholder.com/1024" -o assets/icon.png 2>/dev/null || \
  echo "Add assets/icon.png manually"
```

- [ ] **Step 8: Commit**

```bash
git init
git add .
git commit -m "feat: scaffold Expo project with expo-router and deps"
```

---

## Task 2: Domain Types

**Files:**
- Create: `src/domain/types.ts`

- [ ] **Step 1: Create types.ts**

```typescript
// src/domain/types.ts

export type CategoryId = 'food' | 'water' | 'battery' | 'medical' | 'gear';

export type Category = {
  id: CategoryId;
  name: string;
  icon: string;   // icon name (e.g. from @expo/vector-icons or similar)
  colorKey: string;
  sort_order: number;
};

export type StorageLocation = {
  id: string;
  name: string;
  is_default: boolean;
  sort_order: number;
};

export type ExpiryType = 'expiry' | 'inspection';

export type Item = {
  id: string;
  category_id: CategoryId;
  name: string;
  quantity: number;
  unit: string | null;
  expiry_date: string;          // YYYY-MM-DD
  expiry_type: ExpiryType;
  storage_location_id: string | null;
  storage_note: string | null;
  photo_path: string | null;
  memo: string | null;
  notify_days_before: number;
  notification_id: string | null;
  created_at: string;           // ISO 8601
  updated_at: string;           // ISO 8601
};

export type ExpiryStatus = 'expired' | 'soon' | 'ok';
```

- [ ] **Step 2: Commit**

```bash
git add src/domain/types.ts
git commit -m "feat: add domain types"
```

---

## Task 3: Expiry Helper (with tests)

**Files:**
- Create: `src/domain/expiry.ts`
- Create: `src/domain/expiry.test.ts`

- [ ] **Step 1: Write failing tests**

```typescript
// src/domain/expiry.test.ts
import { getDaysUntil, getExpiryStatus, formatExpiryDate, isExpired } from './expiry';

describe('getDaysUntil', () => {
  it('returns 0 for today', () => {
    const today = '2026-06-29';
    expect(getDaysUntil('2026-06-29', today)).toBe(0);
  });

  it('returns positive for future dates', () => {
    expect(getDaysUntil('2026-07-29', '2026-06-29')).toBe(30);
  });

  it('returns negative for past dates', () => {
    expect(getDaysUntil('2026-06-28', '2026-06-29')).toBe(-1);
  });
});

describe('getExpiryStatus', () => {
  it('returns expired for past dates', () => {
    expect(getExpiryStatus('2026-06-28', '2026-06-29')).toBe('expired');
  });

  it('returns expired for today (day 0)', () => {
    expect(getExpiryStatus('2026-06-29', '2026-06-29')).toBe('expired');
  });

  it('returns soon for 1-30 days ahead', () => {
    expect(getExpiryStatus('2026-07-15', '2026-06-29')).toBe('soon');
    expect(getExpiryStatus('2026-07-29', '2026-06-29')).toBe('soon');
  });

  it('returns ok for more than 30 days ahead', () => {
    expect(getExpiryStatus('2026-07-30', '2026-06-29')).toBe('ok');
  });
});

describe('isExpired', () => {
  it('returns true for past date', () => {
    expect(isExpired('2026-06-01', '2026-06-29')).toBe(true);
  });

  it('returns false for future date', () => {
    expect(isExpired('2026-12-31', '2026-06-29')).toBe(false);
  });
});

describe('formatExpiryDate', () => {
  it('formats to Japanese locale string', () => {
    expect(formatExpiryDate('2026-12-31')).toBe('2026年12月31日');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
npx jest src/domain/expiry.test.ts
```

Expected: FAIL — "Cannot find module './expiry'"

- [ ] **Step 3: Implement expiry.ts**

```typescript
// src/domain/expiry.ts
import dayjs from 'dayjs';
import type { ExpiryStatus } from './types';

export function getDaysUntil(expiryDate: string, today: string): number {
  return dayjs(expiryDate).diff(dayjs(today), 'day');
}

export function getExpiryStatus(expiryDate: string, today: string): ExpiryStatus {
  const days = getDaysUntil(expiryDate, today);
  if (days <= 0) return 'expired';
  if (days <= 30) return 'soon';
  return 'ok';
}

export function isExpired(expiryDate: string, today: string): boolean {
  return getDaysUntil(expiryDate, today) <= 0;
}

export function formatExpiryDate(expiryDate: string): string {
  return dayjs(expiryDate).format('YYYY年M月D日');
}

export function formatDaysUntil(days: number): string {
  if (days < 0) return `${Math.abs(days)}日前に切れました`;
  if (days === 0) return '本日期限';
  return `あと${days}日`;
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
npx jest src/domain/expiry.test.ts
```

Expected: PASS — all 7 tests green

- [ ] **Step 5: Commit**

```bash
git add src/domain/expiry.ts src/domain/expiry.test.ts
git commit -m "feat: add expiry helper with tests"
```

---

## Task 4: Theme Tokens

**Files:**
- Create: `src/theme/tokens.ts`

- [ ] **Step 1: Create tokens.ts**

```typescript
// src/theme/tokens.ts

export const Colors = {
  // Backgrounds
  appBg: '#F5F3EE',
  cardBg: '#FFFFFF',
  formBg: '#F0EDE6',

  // Text
  textMain: '#2C3520',
  textSub: '#6B6B5A',
  textMuted: '#9A9880',
  textPlaceholder: '#B8B59A',

  // Dividers
  divider: '#E8E4DA',

  // Brand: olive green
  primary: '#6B7C47',
  primaryDeep: '#4A5A2A',
  primaryTint: '#EEF1E6',
  primaryRing: '#B0BC88',

  // Expiry status
  expired: {
    bg: '#FAEBE7',
    text: '#B5604F',
    bar: '#D08A7C',
    border: '#F4C4BA',
  },
  soon: {
    bg: '#FBF3E0',
    text: '#A8771F',
    bar: '#E0B04A',
    border: '#F0D898',
  },
  ok: {
    bg: '#F0EDE6',
    text: '#6B6B5A',
    bar: '#B0BC88',
    border: '#E8E4DA',
  },

  // Category icon backgrounds
  category: {
    food:    { bg: '#FAECE8', icon: '#D08060' },
    water:   { bg: '#E8F0FA', icon: '#5080C0' },
    battery: { bg: '#FBF3E0', icon: '#D0A030' },
    medical: { bg: '#FAE8F0', icon: '#C05880' },
    gear:    { bg: '#EEF1E6', icon: '#5A7840' },
  },
} as const;

export const Radius = {
  card: 12,
  chip: 999,
  thumbnail: 8,
} as const;

export const Spacing = {
  screenH: 16,
  screenV: 16,
  cardPad: 16,
  sectionGap: 20,
  rowGap: 12,
} as const;

export const FontSize = {
  title: 20,
  heading: 17,
  body: 15,
  sub: 13,
  caption: 12,
} as const;

export type CategoryId = 'food' | 'water' | 'battery' | 'medical' | 'gear';
```

- [ ] **Step 2: Commit**

```bash
git add src/theme/tokens.ts
git commit -m "feat: add theme tokens (olive green palette)"
```

---

## Task 5: Database — Schema, Seed, Init

**Files:**
- Create: `src/db/schema.ts`
- Create: `src/db/seed.ts`
- Create: `src/db/database.ts`

- [ ] **Step 1: Create schema.ts**

```typescript
// src/db/schema.ts

export const CREATE_CATEGORIES = `
  CREATE TABLE IF NOT EXISTS categories (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    icon TEXT NOT NULL,
    color_key TEXT NOT NULL,
    sort_order INTEGER NOT NULL
  );
`;

export const CREATE_STORAGE_LOCATIONS = `
  CREATE TABLE IF NOT EXISTS storage_locations (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    is_default INTEGER NOT NULL DEFAULT 0,
    sort_order INTEGER NOT NULL
  );
`;

export const CREATE_ITEMS = `
  CREATE TABLE IF NOT EXISTS items (
    id TEXT PRIMARY KEY,
    category_id TEXT NOT NULL,
    name TEXT NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1,
    unit TEXT,
    expiry_date TEXT NOT NULL,
    expiry_type TEXT NOT NULL DEFAULT 'expiry',
    storage_location_id TEXT,
    storage_note TEXT,
    photo_path TEXT,
    memo TEXT,
    notify_days_before INTEGER NOT NULL DEFAULT 14,
    notification_id TEXT,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    FOREIGN KEY (category_id) REFERENCES categories(id),
    FOREIGN KEY (storage_location_id) REFERENCES storage_locations(id)
  );
`;

export const CREATE_IDX_CATEGORY = `
  CREATE INDEX IF NOT EXISTS idx_items_category ON items(category_id);
`;

export const CREATE_IDX_EXPIRY = `
  CREATE INDEX IF NOT EXISTS idx_items_expiry ON items(expiry_date);
`;
```

- [ ] **Step 2: Create seed.ts**

```typescript
// src/db/seed.ts
import type { SQLiteDatabase } from 'expo-sqlite';

const CATEGORIES = [
  { id: 'food',    name: '非常食',       icon: 'package',       color_key: 'food',    sort_order: 1 },
  { id: 'water',   name: '飲料水',       icon: 'droplet',       color_key: 'water',   sort_order: 2 },
  { id: 'battery', name: '電池・電源',   icon: 'battery-2',     color_key: 'battery', sort_order: 3 },
  { id: 'medical', name: '医薬品・救急', icon: 'first-aid-kit', color_key: 'medical', sort_order: 4 },
  { id: 'gear',    name: '防災用品・点検', icon: 'helmet',      color_key: 'gear',    sort_order: 5 },
];

const STORAGE_LOCATIONS = [
  { id: 'loc-1', name: '玄関収納',     is_default: 1, sort_order: 1 },
  { id: 'loc-2', name: 'リビング',     is_default: 1, sort_order: 2 },
  { id: 'loc-3', name: '寝室',         is_default: 1, sort_order: 3 },
  { id: 'loc-4', name: 'キッチン',     is_default: 1, sort_order: 4 },
  { id: 'loc-5', name: '物置',         is_default: 1, sort_order: 5 },
  { id: 'loc-6', name: '車のトランク', is_default: 1, sort_order: 6 },
  { id: 'loc-7', name: 'その他',       is_default: 1, sort_order: 7 },
];

export function seedDatabase(db: SQLiteDatabase): void {
  const catCount = db.getFirstSync<{ count: number }>('SELECT COUNT(*) as count FROM categories')?.count ?? 0;
  if (catCount > 0) return; // already seeded

  db.withTransactionSync(() => {
    for (const cat of CATEGORIES) {
      db.runSync(
        'INSERT INTO categories (id, name, icon, color_key, sort_order) VALUES (?, ?, ?, ?, ?)',
        [cat.id, cat.name, cat.icon, cat.color_key, cat.sort_order]
      );
    }
    for (const loc of STORAGE_LOCATIONS) {
      db.runSync(
        'INSERT INTO storage_locations (id, name, is_default, sort_order) VALUES (?, ?, ?, ?)',
        [loc.id, loc.name, loc.is_default, loc.sort_order]
      );
    }
  });
}
```

- [ ] **Step 3: Create database.ts**

```typescript
// src/db/database.ts
import * as SQLite from 'expo-sqlite';
import {
  CREATE_CATEGORIES,
  CREATE_STORAGE_LOCATIONS,
  CREATE_ITEMS,
  CREATE_IDX_CATEGORY,
  CREATE_IDX_EXPIRY,
} from './schema';
import { seedDatabase } from './seed';

let _db: SQLite.SQLiteDatabase | null = null;

export function getDb(): SQLite.SQLiteDatabase {
  if (!_db) {
    _db = SQLite.openDatabaseSync('bousai.db');
  }
  return _db;
}

export function initDb(): void {
  const db = getDb();
  db.withTransactionSync(() => {
    db.execSync(CREATE_CATEGORIES);
    db.execSync(CREATE_STORAGE_LOCATIONS);
    db.execSync(CREATE_ITEMS);
    db.execSync(CREATE_IDX_CATEGORY);
    db.execSync(CREATE_IDX_EXPIRY);
  });
  seedDatabase(db);
}
```

- [ ] **Step 4: Commit**

```bash
git add src/db/
git commit -m "feat: add SQLite schema, seed, and database init"
```

---

## Task 6: DB Query Layer

**Files:**
- Create: `src/db/queries/categories.ts`
- Create: `src/db/queries/locations.ts`
- Create: `src/db/queries/items.ts`
- Create: `src/db/utils.ts`

- [ ] **Step 1: Create utils.ts (UUID helper)**

```typescript
// src/db/utils.ts
export function uuid(): string {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
    const r = (Math.random() * 16) | 0;
    return (c === 'x' ? r : (r & 0x3) | 0x8).toString(16);
  });
}

export function nowIso(): string {
  return new Date().toISOString();
}
```

- [ ] **Step 2: Create categories.ts**

```typescript
// src/db/queries/categories.ts
import { getDb } from '../database';
import type { Category } from '../../domain/types';

type DbCategory = {
  id: string;
  name: string;
  icon: string;
  color_key: string;
  sort_order: number;
};

function toCategory(row: DbCategory): Category {
  return {
    id: row.id as Category['id'],
    name: row.name,
    icon: row.icon,
    colorKey: row.color_key,
    sort_order: row.sort_order,
  };
}

export function getCategories(): Category[] {
  const db = getDb();
  const rows = db.getAllSync<DbCategory>('SELECT * FROM categories ORDER BY sort_order');
  return rows.map(toCategory);
}
```

- [ ] **Step 3: Create locations.ts**

```typescript
// src/db/queries/locations.ts
import { getDb } from '../database';
import { uuid, nowIso } from '../utils';
import type { StorageLocation } from '../../domain/types';

type DbLocation = {
  id: string;
  name: string;
  is_default: number;
  sort_order: number;
};

function toLocation(row: DbLocation): StorageLocation {
  return {
    id: row.id,
    name: row.name,
    is_default: row.is_default === 1,
    sort_order: row.sort_order,
  };
}

export function getLocations(): StorageLocation[] {
  const db = getDb();
  const rows = db.getAllSync<DbLocation>('SELECT * FROM storage_locations ORDER BY sort_order');
  return rows.map(toLocation);
}

export function createLocation(name: string): StorageLocation {
  const db = getDb();
  const existing = db.getAllSync<DbLocation>('SELECT sort_order FROM storage_locations ORDER BY sort_order DESC LIMIT 1');
  const nextOrder = (existing[0]?.sort_order ?? 0) + 1;
  const loc: DbLocation = { id: uuid(), name, is_default: 0, sort_order: nextOrder };
  db.runSync(
    'INSERT INTO storage_locations (id, name, is_default, sort_order) VALUES (?, ?, ?, ?)',
    [loc.id, loc.name, loc.is_default, loc.sort_order]
  );
  return toLocation(loc);
}

export function updateLocation(id: string, name: string): void {
  const db = getDb();
  db.runSync('UPDATE storage_locations SET name = ? WHERE id = ?', [name, id]);
}

export function deleteLocation(id: string): void {
  const db = getDb();
  // Nullify references in items before deleting
  db.runSync('UPDATE items SET storage_location_id = NULL WHERE storage_location_id = ?', [id]);
  db.runSync('DELETE FROM storage_locations WHERE id = ?', [id]);
}
```

- [ ] **Step 4: Create items.ts**

```typescript
// src/db/queries/items.ts
import { getDb } from '../database';
import { uuid, nowIso } from '../utils';
import type { Item, CategoryId, ExpiryType } from '../../domain/types';

type DbItem = {
  id: string;
  category_id: string;
  name: string;
  quantity: number;
  unit: string | null;
  expiry_date: string;
  expiry_type: string;
  storage_location_id: string | null;
  storage_note: string | null;
  photo_path: string | null;
  memo: string | null;
  notify_days_before: number;
  notification_id: string | null;
  created_at: string;
  updated_at: string;
};

function toItem(row: DbItem): Item {
  return {
    ...row,
    category_id: row.category_id as CategoryId,
    expiry_type: row.expiry_type as ExpiryType,
  };
}

export function getItems(): Item[] {
  const db = getDb();
  const rows = db.getAllSync<DbItem>('SELECT * FROM items ORDER BY expiry_date ASC');
  return rows.map(toItem);
}

export function getItemsByCategory(categoryId: string): Item[] {
  const db = getDb();
  const rows = db.getAllSync<DbItem>(
    'SELECT * FROM items WHERE category_id = ? ORDER BY expiry_date ASC',
    [categoryId]
  );
  return rows.map(toItem);
}

export function getItem(id: string): Item | null {
  const db = getDb();
  const row = db.getFirstSync<DbItem>('SELECT * FROM items WHERE id = ?', [id]);
  return row ? toItem(row) : null;
}

export type CreateItemInput = Omit<Item, 'id' | 'notification_id' | 'created_at' | 'updated_at'>;

export function createItem(input: CreateItemInput): Item {
  const db = getDb();
  const now = nowIso();
  const item: Item = {
    ...input,
    id: uuid(),
    notification_id: null,
    created_at: now,
    updated_at: now,
  };
  db.runSync(
    `INSERT INTO items
      (id, category_id, name, quantity, unit, expiry_date, expiry_type,
       storage_location_id, storage_note, photo_path, memo,
       notify_days_before, notification_id, created_at, updated_at)
     VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
    [
      item.id, item.category_id, item.name, item.quantity, item.unit,
      item.expiry_date, item.expiry_type, item.storage_location_id,
      item.storage_note, item.photo_path, item.memo,
      item.notify_days_before, item.notification_id, item.created_at, item.updated_at,
    ]
  );
  return item;
}

export type UpdateItemInput = Partial<Omit<Item, 'id' | 'created_at'>>;

export function updateItem(id: string, patch: UpdateItemInput): Item {
  const db = getDb();
  const existing = getItem(id);
  if (!existing) throw new Error(`Item not found: ${id}`);
  const updated: Item = { ...existing, ...patch, updated_at: nowIso() };
  db.runSync(
    `UPDATE items SET
      category_id = ?, name = ?, quantity = ?, unit = ?,
      expiry_date = ?, expiry_type = ?, storage_location_id = ?,
      storage_note = ?, photo_path = ?, memo = ?,
      notify_days_before = ?, notification_id = ?, updated_at = ?
     WHERE id = ?`,
    [
      updated.category_id, updated.name, updated.quantity, updated.unit,
      updated.expiry_date, updated.expiry_type, updated.storage_location_id,
      updated.storage_note, updated.photo_path, updated.memo,
      updated.notify_days_before, updated.notification_id, updated.updated_at,
      id,
    ]
  );
  return updated;
}

export function deleteItem(id: string): void {
  const db = getDb();
  db.runSync('DELETE FROM items WHERE id = ?', [id]);
}
```

- [ ] **Step 5: Commit**

```bash
git add src/db/queries/ src/db/utils.ts
git commit -m "feat: add DB query layer for items, categories, locations"
```

---

## Task 7: Zustand Store

**Files:**
- Create: `src/store/useAppStore.ts`

- [ ] **Step 1: Create useAppStore.ts**

```typescript
// src/store/useAppStore.ts
import { create } from 'zustand';
import type { Item, Category, StorageLocation } from '../domain/types';
import { getItems, getItemsByCategory, createItem, updateItem, deleteItem, type CreateItemInput, type UpdateItemInput } from '../db/queries/items';
import { getCategories } from '../db/queries/categories';
import { getLocations, createLocation, updateLocation, deleteLocation } from '../db/queries/locations';

type AppState = {
  items: Item[];
  categories: Category[];
  locations: StorageLocation[];

  // Load actions (call once on app start)
  loadAll: () => void;

  // Item actions
  addItem: (input: CreateItemInput) => Item;
  editItem: (id: string, patch: UpdateItemInput) => Item;
  removeItem: (id: string) => void;

  // Location actions
  addLocation: (name: string) => StorageLocation;
  editLocation: (id: string, name: string) => void;
  removeLocation: (id: string) => void;

  // Derived: items by category (computed on demand, not stored)
  getItemsByCategory: (categoryId: string) => Item[];
};

export const useAppStore = create<AppState>()((set, get) => ({
  items: [],
  categories: [],
  locations: [],

  loadAll: () => {
    set({
      items: getItems(),
      categories: getCategories(),
      locations: getLocations(),
    });
  },

  addItem: (input) => {
    const item = createItem(input);
    set((s) => ({ items: [...s.items, item].sort((a, b) => a.expiry_date.localeCompare(b.expiry_date)) }));
    return item;
  },

  editItem: (id, patch) => {
    const item = updateItem(id, patch);
    set((s) => ({
      items: s.items.map((i) => (i.id === id ? item : i)),
    }));
    return item;
  },

  removeItem: (id) => {
    deleteItem(id);
    set((s) => ({ items: s.items.filter((i) => i.id !== id) }));
  },

  addLocation: (name) => {
    const loc = createLocation(name);
    set((s) => ({ locations: [...s.locations, loc] }));
    return loc;
  },

  editLocation: (id, name) => {
    updateLocation(id, name);
    set((s) => ({
      locations: s.locations.map((l) => (l.id === id ? { ...l, name } : l)),
    }));
  },

  removeLocation: (id) => {
    deleteLocation(id);
    set((s) => ({
      locations: s.locations.filter((l) => l.id !== id),
      items: s.items.map((i) =>
        i.storage_location_id === id ? { ...i, storage_location_id: null } : i
      ),
    }));
  },

  getItemsByCategory: (categoryId) => {
    return get().items.filter((i) => i.category_id === categoryId);
  },
}));
```

- [ ] **Step 2: Commit**

```bash
git add src/store/useAppStore.ts
git commit -m "feat: add Zustand store with item and location actions"
```

---

## Task 8: expo-router Navigation Layout

**Files:**
- Create: `app/_layout.tsx`
- Create: `app/(tabs)/_layout.tsx`

- [ ] **Step 1: Create root _layout.tsx**

```typescript
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import { StatusBar } from 'expo-status-bar';
import { initDb } from '@/db/database';
import { useAppStore } from '@/store/useAppStore';

export default function RootLayout() {
  const loadAll = useAppStore((s) => s.loadAll);

  useEffect(() => {
    initDb();
    loadAll();
  }, []);

  return (
    <>
      <StatusBar style="dark" />
      <Stack screenOptions={{ headerShown: false }}>
        <Stack.Screen name="(tabs)" />
        <Stack.Screen
          name="item/new"
          options={{ presentation: 'modal', headerShown: true, title: 'アイテムを追加' }}
        />
        <Stack.Screen
          name="item/[id]/index"
          options={{ headerShown: true, title: '詳細' }}
        />
        <Stack.Screen
          name="item/[id]/edit"
          options={{ presentation: 'modal', headerShown: true, title: '編集' }}
        />
        <Stack.Screen
          name="category/[id]"
          options={{ headerShown: true }}
        />
      </Stack>
    </>
  );
}
```

- [ ] **Step 2: Create (tabs)/_layout.tsx**

```typescript
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Colors } from '@/theme/tokens';

export default function TabLayout() {
  return (
    <Tabs
      screenOptions={{
        headerShown: false,
        tabBarActiveTintColor: Colors.primary,
        tabBarInactiveTintColor: Colors.textMuted,
        tabBarStyle: {
          backgroundColor: Colors.cardBg,
          borderTopColor: Colors.divider,
        },
      }}
    >
      <Tabs.Screen
        name="index"
        options={{ title: 'ホーム', tabBarIcon: ({ color }) => <TabIcon name="home" color={color} /> }}
      />
      <Tabs.Screen
        name="list"
        options={{ title: '一覧', tabBarIcon: ({ color }) => <TabIcon name="list" color={color} /> }}
      />
      <Tabs.Screen
        name="notifications"
        options={{ title: '通知', tabBarIcon: ({ color }) => <TabIcon name="bell" color={color} /> }}
      />
      <Tabs.Screen
        name="settings"
        options={{ title: '設定', tabBarIcon: ({ color }) => <TabIcon name="settings" color={color} /> }}
      />
    </Tabs>
  );
}

function TabIcon({ name, color }: { name: string; color: string }) {
  // Using a simple text fallback until icon library is added
  // Replace with @expo/vector-icons or react-native-vector-icons as needed
  const icons: Record<string, string> = { home: '🏠', list: '📋', bell: '🔔', settings: '⚙️' };
  const { Text } = require('react-native');
  return <Text style={{ fontSize: 20 }}>{icons[name] ?? '●'}</Text>;
}
```

Note: The emoji icons are a placeholder. After confirming layout works, replace TabIcon with a proper icon library (e.g., `@expo/vector-icons` — `import { Ionicons } from '@expo/vector-icons'`).

- [ ] **Step 3: Create placeholder tab screens (so expo-router doesn't error)**

```bash
mkdir -p app/item/\[id\] app/category
```

Create `app/(tabs)/index.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function HomeScreen() {
  return <View><Text>Home (placeholder)</Text></View>;
}
```

Create `app/(tabs)/list.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function ListScreen() {
  return <View><Text>List (placeholder)</Text></View>;
}
```

Create `app/(tabs)/notifications.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function NotificationsScreen() {
  return <View><Text>Notifications (placeholder)</Text></View>;
}
```

Create `app/(tabs)/settings.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function SettingsScreen() {
  return <View><Text>Settings (placeholder)</Text></View>;
}
```

Create `app/item/new.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function NewItemScreen() {
  return <View><Text>New Item (placeholder)</Text></View>;
}
```

Create `app/item/[id]/index.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function ItemDetailScreen() {
  return <View><Text>Item Detail (placeholder)</Text></View>;
}
```

Create `app/item/[id]/edit.tsx`:
```typescript
import { View, Text } from 'react-native';
export default function ItemEditScreen() {
  return <View><Text>Item Edit (placeholder)</Text></View>;
}
```

Create `app/category/[id].tsx`:
```typescript
import { View, Text } from 'react-native';
export default function CategoryScreen() {
  return <View><Text>Category (placeholder)</Text></View>;
}
```

- [ ] **Step 4: Verify app builds and tab navigation works**

```bash
npx expo start
```

Open in Expo Go or iOS Simulator. Verify the 4 tabs appear and are tappable.

- [ ] **Step 5: Commit**

```bash
git add app/
git commit -m "feat: add expo-router navigation layout with tab bar"
```

---

## Task 9: Shared UI Components

**Files:**
- Create: `src/components/ExpiryBadge.tsx`
- Create: `src/components/MetricCard.tsx`
- Create: `src/components/AlertBanner.tsx`
- Create: `src/components/CategoryCard.tsx`
- Create: `src/components/ItemCard.tsx`

- [ ] **Step 1: Create ExpiryBadge.tsx**

```typescript
// src/components/ExpiryBadge.tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { Colors, Radius, FontSize } from '../theme/tokens';
import type { ExpiryStatus } from '../domain/types';

type Props = {
  status: ExpiryStatus;
  label: string; // e.g. "あと14日" or "3日前に切れました"
};

const STATUS_COLORS: Record<ExpiryStatus, { bg: string; text: string }> = {
  expired: { bg: Colors.expired.bg, text: Colors.expired.text },
  soon:    { bg: Colors.soon.bg,    text: Colors.soon.text },
  ok:      { bg: Colors.ok.bg,      text: Colors.ok.text },
};

export function ExpiryBadge({ status, label }: Props) {
  const c = STATUS_COLORS[status];
  return (
    <View style={[styles.badge, { backgroundColor: c.bg }]}>
      <Text style={[styles.label, { color: c.text }]}>{label}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  badge: {
    paddingHorizontal: 10,
    paddingVertical: 3,
    borderRadius: Radius.chip,
    alignSelf: 'flex-start',
  },
  label: {
    fontSize: FontSize.caption,
    fontWeight: '500',
  },
});
```

- [ ] **Step 2: Create MetricCard.tsx**

```typescript
// src/components/MetricCard.tsx
import React from 'react';
import { View, Text, StyleSheet } from 'react-native';
import { Colors, Radius, Spacing, FontSize } from '../theme/tokens';

type Props = {
  count: number;
  label: string;
  countColor?: string;
};

export function MetricCard({ count, label, countColor = Colors.textMain }: Props) {
  return (
    <View style={styles.card}>
      <Text style={[styles.count, { color: countColor }]}>{count}</Text>
      <Text style={styles.label}>{label}</Text>
    </View>
  );
}

const styles = StyleSheet.create({
  card: {
    flex: 1,
    backgroundColor: Colors.cardBg,
    borderRadius: Radius.card,
    padding: Spacing.cardPad,
    alignItems: 'center',
    borderWidth: 1,
    borderColor: Colors.divider,
  },
  count: {
    fontSize: 32,
    fontWeight: '700',
    lineHeight: 36,
  },
  label: {
    fontSize: FontSize.caption,
    color: Colors.textSub,
    marginTop: 4,
  },
});
```

- [ ] **Step 3: Create AlertBanner.tsx**

```typescript
// src/components/AlertBanner.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { Colors, Radius, Spacing, FontSize } from '../theme/tokens';
import type { Item } from '../domain/types';
import { formatExpiryDate, getDaysUntil } from '../domain/expiry';
import dayjs from 'dayjs';

type Props = {
  item: Item;
  onPress: () => void;
};

export function AlertBanner({ item, onPress }: Props) {
  const today = dayjs().format('YYYY-MM-DD');
  const days = getDaysUntil(item.expiry_date, today);
  const isExpired = days <= 0;

  const bgColor = isExpired ? Colors.expired.bg : Colors.soon.bg;
  const textColor = isExpired ? Colors.expired.text : Colors.soon.text;
  const borderColor = isExpired ? Colors.expired.border : Colors.soon.border;

  const message = isExpired
    ? `${item.name}の期限が切れています（${formatExpiryDate(item.expiry_date)}）`
    : `${item.name}の期限が近づいています（あと${days}日）`;

  return (
    <TouchableOpacity
      style={[styles.banner, { backgroundColor: bgColor, borderColor }]}
      onPress={onPress}
      activeOpacity={0.8}
    >
      <Text style={[styles.text, { color: textColor }]} numberOfLines={2}>
        ⚠️ {message}
      </Text>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  banner: {
    borderRadius: Radius.card,
    borderWidth: 1,
    padding: Spacing.cardPad,
    marginBottom: Spacing.rowGap,
  },
  text: {
    fontSize: FontSize.sub,
    fontWeight: '500',
  },
});
```

- [ ] **Step 4: Create CategoryCard.tsx**

```typescript
// src/components/CategoryCard.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet } from 'react-native';
import { Colors, Radius, Spacing, FontSize } from '../theme/tokens';
import type { Category, Item } from '../domain/types';
import { getExpiryStatus, getDaysUntil, formatExpiryDate } from '../domain/expiry';
import { ExpiryBadge } from './ExpiryBadge';
import dayjs from 'dayjs';

type Props = {
  category: Category;
  items: Item[];
  onPress: () => void;
};

export function CategoryCard({ category, items, onPress }: Props) {
  const today = dayjs().format('YYYY-MM-DD');
  const catColors = Colors.category[category.id as keyof typeof Colors.category];

  const sorted = [...items].sort((a, b) => a.expiry_date.localeCompare(b.expiry_date));
  const mostUrgent = sorted[0];
  const expiredCount = items.filter((i) => getDaysUntil(i.expiry_date, today) <= 0).length;
  const soonCount = items.filter((i) => {
    const d = getDaysUntil(i.expiry_date, today);
    return d > 0 && d <= 30;
  }).length;

  const badgeStatus = expiredCount > 0 ? 'expired' : soonCount > 0 ? 'soon' : 'ok';
  const badgeLabel = expiredCount > 0
    ? `期限切れ ${expiredCount}件`
    : soonCount > 0
    ? `もうすぐ ${soonCount}件`
    : `${items.length}件`;

  return (
    <TouchableOpacity style={styles.card} onPress={onPress} activeOpacity={0.8}>
      <View style={[styles.iconBox, { backgroundColor: catColors.bg }]}>
        <Text style={{ color: catColors.icon, fontSize: 22 }}>
          {CATEGORY_EMOJI[category.id as keyof typeof CATEGORY_EMOJI] ?? '📦'}
        </Text>
      </View>
      <View style={styles.info}>
        <Text style={styles.name}>{category.name}</Text>
        {mostUrgent && (
          <Text style={styles.sub} numberOfLines={1}>
            直近: {formatExpiryDate(mostUrgent.expiry_date)}
          </Text>
        )}
      </View>
      <ExpiryBadge status={badgeStatus} label={badgeLabel} />
    </TouchableOpacity>
  );
}

const CATEGORY_EMOJI = { food: '🍱', water: '💧', battery: '🔋', medical: '💊', gear: '⛑️' };

const styles = StyleSheet.create({
  card: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.cardBg,
    borderRadius: Radius.card,
    padding: Spacing.cardPad,
    marginBottom: Spacing.rowGap,
    borderWidth: 1,
    borderColor: Colors.divider,
    gap: 12,
  },
  iconBox: {
    width: 44,
    height: 44,
    borderRadius: 10,
    alignItems: 'center',
    justifyContent: 'center',
  },
  info: { flex: 1 },
  name: { fontSize: FontSize.body, fontWeight: '500', color: Colors.textMain },
  sub: { fontSize: FontSize.caption, color: Colors.textSub, marginTop: 2 },
});
```

- [ ] **Step 5: Create ItemCard.tsx**

```typescript
// src/components/ItemCard.tsx
import React from 'react';
import { View, Text, TouchableOpacity, StyleSheet, Image } from 'react-native';
import { Colors, Radius, Spacing, FontSize } from '../theme/tokens';
import type { Item } from '../domain/types';
import { getExpiryStatus, getDaysUntil, formatDaysUntil } from '../domain/expiry';
import { ExpiryBadge } from './ExpiryBadge';
import dayjs from 'dayjs';

type Props = {
  item: Item;
  locationName?: string;
  onPress: () => void;
};

export function ItemCard({ item, locationName, onPress }: Props) {
  const today = dayjs().format('YYYY-MM-DD');
  const days = getDaysUntil(item.expiry_date, today);
  const status = getExpiryStatus(item.expiry_date, today);

  return (
    <TouchableOpacity style={styles.card} onPress={onPress} activeOpacity={0.8}>
      <View style={styles.thumbnail}>
        {item.photo_path ? (
          <Image source={{ uri: item.photo_path }} style={styles.image} />
        ) : (
          <View style={[styles.imagePlaceholder, { backgroundColor: Colors.formBg }]}>
            <Text style={{ fontSize: 20 }}>📦</Text>
          </View>
        )}
      </View>
      <View style={styles.info}>
        <Text style={styles.name} numberOfLines={1}>{item.name}</Text>
        {locationName && (
          <Text style={styles.location} numberOfLines={1}>📍 {locationName}</Text>
        )}
        <View style={styles.badge}>
          <ExpiryBadge status={status} label={formatDaysUntil(days)} />
        </View>
      </View>
      <Text style={styles.qty}>{item.quantity}{item.unit ?? ''}</Text>
    </TouchableOpacity>
  );
}

const styles = StyleSheet.create({
  card: {
    flexDirection: 'row',
    alignItems: 'center',
    backgroundColor: Colors.cardBg,
    borderRadius: Radius.card,
    padding: Spacing.cardPad,
    marginBottom: Spacing.rowGap,
    borderWidth: 1,
    borderColor: Colors.divider,
    gap: 12,
  },
  thumbnail: { width: 56, height: 56 },
  image: { width: 56, height: 56, borderRadius: Radius.thumbnail },
  imagePlaceholder: {
    width: 56, height: 56, borderRadius: Radius.thumbnail,
    alignItems: 'center', justifyContent: 'center',
  },
  info: { flex: 1 },
  name: { fontSize: FontSize.body, fontWeight: '500', color: Colors.textMain },
  location: { fontSize: FontSize.caption, color: Colors.textSub, marginTop: 2 },
  badge: { marginTop: 4 },
  qty: { fontSize: FontSize.sub, color: Colors.textSub, fontWeight: '500' },
});
```

- [ ] **Step 6: Commit**

```bash
git add src/components/
git commit -m "feat: add shared UI components (Badge, MetricCard, AlertBanner, CategoryCard, ItemCard)"
```

---

## Task 10: Home Screen

**Files:**
- Modify: `app/(tabs)/index.tsx`

- [ ] **Step 1: Implement HomeScreen**

```typescript
// app/(tabs)/index.tsx
import React, { useMemo } from 'react';
import {
  View, Text, ScrollView, TouchableOpacity, StyleSheet, SafeAreaView,
} from 'react-native';
import { useRouter } from 'expo-router';
import dayjs from 'dayjs';
import { useAppStore } from '@/store/useAppStore';
import { getDaysUntil, getExpiryStatus } from '@/domain/expiry';
import { Colors, Spacing, FontSize } from '@/theme/tokens';
import { MetricCard } from '@/components/MetricCard';
import { AlertBanner } from '@/components/AlertBanner';
import { CategoryCard } from '@/components/CategoryCard';

export default function HomeScreen() {
  const router = useRouter();
  const { items, categories } = useAppStore();
  const today = dayjs().format('YYYY-MM-DD');

  const expiredCount = useMemo(
    () => items.filter((i) => getDaysUntil(i.expiry_date, today) <= 0).length,
    [items, today]
  );

  const soonCount = useMemo(
    () => items.filter((i) => {
      const d = getDaysUntil(i.expiry_date, today);
      return d > 0 && d <= 30;
    }).length,
    [items, today]
  );

  const mostUrgent = useMemo(() => {
    const sorted = [...items].sort((a, b) => a.expiry_date.localeCompare(b.expiry_date));
    return sorted.find((i) => getExpiryStatus(i.expiry_date, today) !== 'ok') ?? null;
  }, [items, today]);

  return (
    <SafeAreaView style={styles.safe}>
      <ScrollView
        contentContainerStyle={styles.scroll}
        showsVerticalScrollIndicator={false}
      >
        {/* Header */}
        <View style={styles.header}>
          <Text style={styles.title}>防災ストック</Text>
          <Text style={styles.date}>{dayjs().format('YYYY年M月D日')}</Text>
        </View>

        {/* Summary metrics */}
        <View style={styles.metrics}>
          <MetricCard
            count={expiredCount}
            label="期限切れ"
            countColor={expiredCount > 0 ? Colors.expired.text : Colors.textMuted}
          />
          <View style={styles.metricGap} />
          <MetricCard
            count={soonCount}
            label="30日以内"
            countColor={soonCount > 0 ? Colors.soon.text : Colors.textMuted}
          />
        </View>

        {/* Urgent alert banner */}
        {mostUrgent && (
          <AlertBanner
            item={mostUrgent}
            onPress={() => router.push(`/item/${mostUrgent.id}`)}
          />
        )}

        {/* Category list */}
        <Text style={styles.sectionTitle}>カテゴリ別</Text>
        {categories.map((cat) => {
          const catItems = items.filter((i) => i.category_id === cat.id);
          return (
            <CategoryCard
              key={cat.id}
              category={cat}
              items={catItems}
              onPress={() => router.push(`/category/${cat.id}`)}
            />
          );
        })}

        {/* FAB-style button */}
        <TouchableOpacity
          style={styles.addButton}
          onPress={() => router.push('/item/new')}
          activeOpacity={0.85}
        >
          <Text style={styles.addButtonText}>＋ アイテムを追加</Text>
        </TouchableOpacity>
      </ScrollView>
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
  scroll: { padding: Spacing.screenH, paddingBottom: 40 },
  header: { marginBottom: Spacing.sectionGap },
  title: { fontSize: FontSize.title, fontWeight: '700', color: Colors.textMain },
  date: { fontSize: FontSize.caption, color: Colors.textSub, marginTop: 2 },
  metrics: { flexDirection: 'row', marginBottom: Spacing.sectionGap },
  metricGap: { width: 12 },
  sectionTitle: {
    fontSize: FontSize.sub,
    color: Colors.textMuted,
    fontWeight: '500',
    textTransform: 'uppercase',
    marginBottom: Spacing.rowGap,
  },
  addButton: {
    backgroundColor: Colors.primary,
    borderRadius: Radius.card,
    padding: 16,
    alignItems: 'center',
    marginTop: 8,
  },
  addButtonText: { color: '#FFF', fontSize: FontSize.body, fontWeight: '600' },
});
```

Note: Add `import { Radius } from '@/theme/tokens';` — it was used but missing from the import list above.

- [ ] **Step 2: Verify HomeScreen renders correctly**

Run `npx expo start`, open on device/simulator, navigate to Home tab. Verify:
- Summary metrics show (both 0 initially)
- 5 category cards appear with olive-green palette
- "アイテムを追加" button is visible at the bottom

- [ ] **Step 3: Commit**

```bash
git add app/(tabs)/index.tsx
git commit -m "feat: implement Home screen with metrics, alert banner, category cards"
```

---

## Task 11: Category List Screen

**Files:**
- Modify: `app/category/[id].tsx`

- [ ] **Step 1: Implement CategoryScreen**

```typescript
// app/category/[id].tsx
import React from 'react';
import { View, FlatList, Text, StyleSheet, SafeAreaView } from 'react-native';
import { useLocalSearchParams, useRouter, Stack } from 'expo-router';
import { useAppStore } from '@/store/useAppStore';
import { ItemCard } from '@/components/ItemCard';
import { Colors, Spacing, FontSize } from '@/theme/tokens';

export default function CategoryScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const router = useRouter();
  const { items, categories, locations } = useAppStore();

  const category = categories.find((c) => c.id === id);
  const catItems = items.filter((i) => i.category_id === id);

  function getLocationName(locationId: string | null): string | undefined {
    if (!locationId) return undefined;
    return locations.find((l) => l.id === locationId)?.name;
  }

  return (
    <SafeAreaView style={styles.safe}>
      <Stack.Screen options={{ title: category?.name ?? 'カテゴリ' }} />
      <FlatList
        data={catItems}
        keyExtractor={(item) => item.id}
        contentContainerStyle={styles.list}
        ListEmptyComponent={
          <View style={styles.empty}>
            <Text style={styles.emptyText}>アイテムがありません</Text>
          </View>
        }
        renderItem={({ item }) => (
          <ItemCard
            item={item}
            locationName={getLocationName(item.storage_location_id)}
            onPress={() => router.push(`/item/${item.id}`)}
          />
        )}
      />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
  list: { padding: Spacing.screenH, paddingBottom: 40 },
  empty: { alignItems: 'center', paddingTop: 60 },
  emptyText: { fontSize: FontSize.body, color: Colors.textMuted },
});
```

- [ ] **Step 2: Verify navigation from Home → Category works**

Tap a category card on Home → should navigate to category list, back button should work.

- [ ] **Step 3: Commit**

```bash
git add app/category/[id].tsx
git commit -m "feat: implement Category list screen"
```

---

## Task 12: Item Detail Screen

**Files:**
- Modify: `app/item/[id]/index.tsx`

- [ ] **Step 1: Implement ItemDetailScreen**

```typescript
// app/item/[id]/index.tsx
import React, { useState } from 'react';
import {
  View, Text, ScrollView, Image, TouchableOpacity,
  StyleSheet, Modal, SafeAreaView, Alert,
} from 'react-native';
import { useLocalSearchParams, useRouter, Stack } from 'expo-router';
import dayjs from 'dayjs';
import DateTimePicker from '@react-native-community/datetimepicker';
import { useAppStore } from '@/store/useAppStore';
import { getDaysUntil, getExpiryStatus, formatExpiryDate, formatDaysUntil } from '@/domain/expiry';
import { ExpiryBadge } from '@/components/ExpiryBadge';
import { Colors, Spacing, FontSize, Radius } from '@/theme/tokens';

export default function ItemDetailScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const router = useRouter();
  const { items, locations, editItem, removeItem } = useAppStore();
  const [showUpdateModal, setShowUpdateModal] = useState(false);
  const [newDate, setNewDate] = useState(new Date());

  const item = items.find((i) => i.id === id);
  if (!item) return null;

  const today = dayjs().format('YYYY-MM-DD');
  const days = getDaysUntil(item.expiry_date, today);
  const status = getExpiryStatus(item.expiry_date, today);
  const location = locations.find((l) => l.id === item.storage_location_id);

  const isInspection = item.expiry_type === 'inspection';
  const expiryLabel = isInspection ? '次回点検日' : '期限日';

  function handleDelete() {
    Alert.alert('削除', `「${item.name}」を削除しますか?`, [
      { text: 'キャンセル', style: 'cancel' },
      {
        text: '削除', style: 'destructive', onPress: () => {
          removeItem(item.id);
          router.back();
        },
      },
    ]);
  }

  function handleQuickUpdate() {
    setNewDate(new Date(item.expiry_date));
    setShowUpdateModal(true);
  }

  function handleSaveDate() {
    editItem(item.id, { expiry_date: dayjs(newDate).format('YYYY-MM-DD') });
    setShowUpdateModal(false);
  }

  return (
    <SafeAreaView style={styles.safe}>
      <Stack.Screen
        options={{
          title: item.name,
          headerRight: () => (
            <View style={{ flexDirection: 'row', gap: 12 }}>
              <TouchableOpacity onPress={() => router.push(`/item/${item.id}/edit`)}>
                <Text style={{ color: Colors.primary, fontSize: FontSize.body }}>編集</Text>
              </TouchableOpacity>
              <TouchableOpacity onPress={handleDelete}>
                <Text style={{ color: Colors.expired.text, fontSize: FontSize.body }}>削除</Text>
              </TouchableOpacity>
            </View>
          ),
        }}
      />
      <ScrollView contentContainerStyle={styles.scroll}>
        {/* Photo */}
        <View style={styles.photoBox}>
          {item.photo_path ? (
            <Image source={{ uri: item.photo_path }} style={styles.photo} />
          ) : (
            <View style={[styles.photo, styles.photoPlaceholder]}>
              <Text style={{ fontSize: 48 }}>📦</Text>
            </View>
          )}
        </View>

        {/* Status warning */}
        {status !== 'ok' && (
          <View style={[styles.warningBox, { backgroundColor: Colors[status].bg, borderColor: Colors[status].border }]}>
            <Text style={[styles.warningText, { color: Colors[status].text }]}>
              {expiryLabel}: {formatExpiryDate(item.expiry_date)}
            </Text>
            <Text style={[styles.warningDays, { color: Colors[status].text }]}>
              {formatDaysUntil(days)}
            </Text>
          </View>
        )}

        {/* Details */}
        <View style={styles.section}>
          <DetailRow label={expiryLabel} value={formatExpiryDate(item.expiry_date)} />
          {status === 'ok' && (
            <DetailRow label="残り" value={formatDaysUntil(days)} />
          )}
          <DetailRow label="数量" value={`${item.quantity}${item.unit ?? ''}`} />
          {location && <DetailRow label="保管場所" value={`📍 ${location.name}`} />}
          {item.storage_note && <DetailRow label="補足" value={item.storage_note} />}
          {item.memo && <DetailRow label="メモ" value={item.memo} />}
          <DetailRow label="通知" value={`期限の${item.notify_days_before}日前`} />
        </View>

        {/* Quick update button */}
        <TouchableOpacity style={styles.updateBtn} onPress={handleQuickUpdate} activeOpacity={0.85}>
          <Text style={styles.updateBtnText}>🔄 補充・期限を更新する</Text>
        </TouchableOpacity>
      </ScrollView>

      {/* Quick date update modal */}
      <Modal visible={showUpdateModal} transparent animationType="slide">
        <View style={styles.modalOverlay}>
          <View style={styles.modalCard}>
            <Text style={styles.modalTitle}>期限日を更新</Text>
            <DateTimePicker
              value={newDate}
              mode="date"
              display="spinner"
              locale="ja-JP"
              onChange={(_, date) => date && setNewDate(date)}
              style={{ width: '100%' }}
            />
            <View style={styles.modalActions}>
              <TouchableOpacity onPress={() => setShowUpdateModal(false)} style={styles.modalCancelBtn}>
                <Text style={{ color: Colors.textSub }}>キャンセル</Text>
              </TouchableOpacity>
              <TouchableOpacity onPress={handleSaveDate} style={styles.modalSaveBtn}>
                <Text style={{ color: '#FFF', fontWeight: '600' }}>保存</Text>
              </TouchableOpacity>
            </View>
          </View>
        </View>
      </Modal>
    </SafeAreaView>
  );
}

function DetailRow({ label, value }: { label: string; value: string }) {
  return (
    <View style={detailStyles.row}>
      <Text style={detailStyles.label}>{label}</Text>
      <Text style={detailStyles.value}>{value}</Text>
    </View>
  );
}

const detailStyles = StyleSheet.create({
  row: { flexDirection: 'row', paddingVertical: 12, borderBottomWidth: 1, borderBottomColor: Colors.divider },
  label: { flex: 1, fontSize: FontSize.sub, color: Colors.textSub },
  value: { flex: 2, fontSize: FontSize.sub, color: Colors.textMain },
});

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
  scroll: { paddingBottom: 40 },
  photoBox: { width: '100%', height: 220 },
  photo: { width: '100%', height: '100%' },
  photoPlaceholder: { alignItems: 'center', justifyContent: 'center', backgroundColor: Colors.formBg },
  warningBox: {
    margin: Spacing.screenH, borderRadius: Radius.card,
    borderWidth: 1, padding: Spacing.cardPad,
  },
  warningText: { fontSize: FontSize.sub },
  warningDays: { fontSize: FontSize.heading, fontWeight: '700', marginTop: 4 },
  section: { marginHorizontal: Spacing.screenH },
  updateBtn: {
    margin: Spacing.screenH, backgroundColor: Colors.primary,
    borderRadius: Radius.card, padding: 16, alignItems: 'center',
  },
  updateBtnText: { color: '#FFF', fontSize: FontSize.body, fontWeight: '600' },
  modalOverlay: { flex: 1, backgroundColor: 'rgba(0,0,0,0.4)', justifyContent: 'flex-end' },
  modalCard: {
    backgroundColor: Colors.cardBg, borderTopLeftRadius: 20, borderTopRightRadius: 20,
    padding: Spacing.screenH, paddingBottom: 40,
  },
  modalTitle: { fontSize: FontSize.heading, fontWeight: '600', color: Colors.textMain, marginBottom: 16 },
  modalActions: { flexDirection: 'row', gap: 12, marginTop: 16 },
  modalCancelBtn: { flex: 1, padding: 14, borderRadius: Radius.card, borderWidth: 1, borderColor: Colors.divider, alignItems: 'center' },
  modalSaveBtn: { flex: 1, padding: 14, borderRadius: Radius.card, backgroundColor: Colors.primary, alignItems: 'center' },
});
```

- [ ] **Step 2: Verify Item Detail works end-to-end**

Add a test item via the DB directly (or temporarily via the store), then navigate to its detail screen. Verify photo placeholder, status badges, quick-update modal date picker.

- [ ] **Step 3: Commit**

```bash
git add "app/item/[id]/index.tsx"
git commit -m "feat: implement Item detail screen with quick-update modal"
```

---

## Task 13: Photo Service

**Files:**
- Create: `src/services/photos.ts`

- [ ] **Step 1: Create photos.ts**

```typescript
// src/services/photos.ts
import * as ImagePicker from 'expo-image-picker';
import * as FileSystem from 'expo-file-system';

const PHOTOS_DIR = FileSystem.documentDirectory + 'item-photos/';

async function ensurePhotosDir(): Promise<void> {
  const info = await FileSystem.getInfoAsync(PHOTOS_DIR);
  if (!info.exists) {
    await FileSystem.makeDirectoryAsync(PHOTOS_DIR, { intermediates: true });
  }
}

export async function pickPhoto(): Promise<string | null> {
  const perm = await ImagePicker.requestMediaLibraryPermissionsAsync();
  if (!perm.granted) return null;

  const result = await ImagePicker.launchImageLibraryAsync({
    mediaTypes: ImagePicker.MediaTypeOptions.Images,
    allowsEditing: true,
    aspect: [4, 3],
    quality: 0.7,
  });

  if (result.canceled || !result.assets[0]) return null;
  return result.assets[0].uri;
}

export async function takePhoto(): Promise<string | null> {
  const perm = await ImagePicker.requestCameraPermissionsAsync();
  if (!perm.granted) return null;

  const result = await ImagePicker.launchCameraAsync({
    allowsEditing: true,
    aspect: [4, 3],
    quality: 0.7,
  });

  if (result.canceled || !result.assets[0]) return null;
  return result.assets[0].uri;
}

export async function savePhoto(sourceUri: string, itemId: string): Promise<string> {
  await ensurePhotosDir();
  const ext = sourceUri.split('.').pop() ?? 'jpg';
  const destPath = PHOTOS_DIR + `${itemId}.${ext}`;
  await FileSystem.copyAsync({ from: sourceUri, to: destPath });
  return destPath;
}

export async function deletePhoto(photoPath: string): Promise<void> {
  const info = await FileSystem.getInfoAsync(photoPath);
  if (info.exists) {
    await FileSystem.deleteAsync(photoPath, { idempotent: true });
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add src/services/photos.ts
git commit -m "feat: add photo service (pick, take, save, delete)"
```

---

## Task 14: Notification Service

**Files:**
- Create: `src/services/notifications.ts`

- [ ] **Step 1: Create notifications.ts**

```typescript
// src/services/notifications.ts
import * as Notifications from 'expo-notifications';
import dayjs from 'dayjs';
import type { Item } from '../domain/types';

Notifications.setNotificationHandler({
  handleNotification: async () => ({
    shouldShowAlert: true,
    shouldPlaySound: true,
    shouldSetBadge: false,
  }),
});

export async function requestNotificationPermission(): Promise<boolean> {
  const { status } = await Notifications.requestPermissionsAsync();
  return status === 'granted';
}

export async function scheduleItemNotification(item: Item): Promise<string | null> {
  const granted = await requestNotificationPermission();
  if (!granted) return null;

  const notifyDate = dayjs(item.expiry_date).subtract(item.notify_days_before, 'day');
  const notifyTime = notifyDate.hour(9).minute(0).second(0).millisecond(0);

  // Don't schedule if the notification time is in the past
  if (notifyTime.isBefore(dayjs())) return null;

  const notificationId = await Notifications.scheduleNotificationAsync({
    content: {
      title: '防災ストック 期限のお知らせ',
      body: `${item.name}の期限が近づいています（${dayjs(item.expiry_date).format('YYYY年M月D日')}）`,
      data: { itemId: item.id },
    },
    trigger: {
      date: notifyTime.toDate(),
    },
  });

  return notificationId;
}

export async function cancelItemNotification(notificationId: string): Promise<void> {
  await Notifications.cancelScheduledNotificationAsync(notificationId);
}

export async function syncNotificationsForItems(
  items: Item[],
  updateItemNotificationId: (id: string, notificationId: string | null) => void
): Promise<void> {
  const scheduled = await Notifications.getAllScheduledNotificationsAsync();
  const scheduledIds = new Set(scheduled.map((n) => n.identifier));

  for (const item of items) {
    if (item.notification_id && !scheduledIds.has(item.notification_id)) {
      // Notification was lost (e.g. device restart); reschedule
      const newId = await scheduleItemNotification(item);
      updateItemNotificationId(item.id, newId);
    }
  }
}
```

- [ ] **Step 2: Commit**

```bash
git add src/services/notifications.ts
git commit -m "feat: add notification service (schedule, cancel, sync)"
```

---

## Task 15: Item Form (New + Edit)

**Files:**
- Create: `src/components/ItemForm.tsx`
- Modify: `app/item/new.tsx`
- Modify: `app/item/[id]/edit.tsx`

- [ ] **Step 1: Create ItemForm.tsx**

```typescript
// src/components/ItemForm.tsx
import React, { useState } from 'react';
import {
  View, Text, TextInput, TouchableOpacity, StyleSheet,
  ScrollView, Alert, ActionSheetIOS, Platform,
} from 'react-native';
import DateTimePicker from '@react-native-community/datetimepicker';
import dayjs from 'dayjs';
import { Colors, Spacing, FontSize, Radius } from '../theme/tokens';
import type { Item, CategoryId, ExpiryType } from '../domain/types';
import { pickPhoto, takePhoto, savePhoto, deletePhoto } from '../services/photos';
import { scheduleItemNotification, cancelItemNotification } from '../services/notifications';
import { useAppStore } from '../store/useAppStore';
import type { CreateItemInput } from '../db/queries/items';

const NOTIFY_OPTIONS = [3, 7, 14, 30] as const;

type Props = {
  initialValues?: Partial<Item>;
  onSave: (input: CreateItemInput) => void;
  onCancel: () => void;
};

export function ItemForm({ initialValues, onSave, onCancel }: Props) {
  const { categories, locations, addLocation } = useAppStore();

  const [categoryId, setCategoryId] = useState<CategoryId>(
    initialValues?.category_id ?? 'food'
  );
  const [name, setName] = useState(initialValues?.name ?? '');
  const [quantity, setQuantity] = useState(String(initialValues?.quantity ?? '1'));
  const [unit, setUnit] = useState(initialValues?.unit ?? '');
  const [expiryDate, setExpiryDate] = useState(
    new Date(initialValues?.expiry_date ?? dayjs().add(1, 'year').format('YYYY-MM-DD'))
  );
  const [expiryType] = useState<ExpiryType>(initialValues?.expiry_type ?? 'expiry');
  const [locationId, setLocationId] = useState<string | null>(
    initialValues?.storage_location_id ?? null
  );
  const [storageNote, setStorageNote] = useState(initialValues?.storage_note ?? '');
  const [memo, setMemo] = useState(initialValues?.memo ?? '');
  const [notifyDays, setNotifyDays] = useState(initialValues?.notify_days_before ?? 14);
  const [photoPath, setPhotoPath] = useState<string | null>(initialValues?.photo_path ?? null);
  const [newLocationName, setNewLocationName] = useState('');
  const [showNewLocation, setShowNewLocation] = useState(false);

  const isInspection = categoryId === 'gear';
  const expiryLabel = isInspection ? '次回点検日' : '期限日';

  async function handlePhotoPress() {
    if (Platform.OS === 'ios') {
      ActionSheetIOS.showActionSheetWithOptions(
        { options: ['キャンセル', 'カメラで撮影', 'ライブラリから選択'], cancelButtonIndex: 0 },
        async (idx) => {
          if (idx === 1) {
            const uri = await takePhoto();
            if (uri) setPhotoPath(uri);
          } else if (idx === 2) {
            const uri = await pickPhoto();
            if (uri) setPhotoPath(uri);
          }
        }
      );
    } else {
      const uri = await pickPhoto();
      if (uri) setPhotoPath(uri);
    }
  }

  function handleAddLocation() {
    if (!newLocationName.trim()) return;
    addLocation(newLocationName.trim());
    setNewLocationName('');
    setShowNewLocation(false);
  }

  async function handleSave() {
    if (!name.trim()) {
      Alert.alert('エラー', 'アイテム名を入力してください');
      return;
    }

    const expiryDateStr = dayjs(expiryDate).format('YYYY-MM-DD');

    // Handle photo save
    let finalPhotoPath: string | null = null;
    if (photoPath) {
      // If photoPath looks like a temp picker URI (not our persistent dir), copy it
      if (!photoPath.includes('item-photos/')) {
        const itemId = initialValues?.id ?? 'new-' + Date.now();
        finalPhotoPath = await savePhoto(photoPath, itemId);
        // Delete old photo if editing
        if (initialValues?.photo_path && initialValues.photo_path !== finalPhotoPath) {
          await deletePhoto(initialValues.photo_path);
        }
      } else {
        finalPhotoPath = photoPath;
      }
    } else if (initialValues?.photo_path) {
      // Photo was removed
      await deletePhoto(initialValues.photo_path);
    }

    const input: CreateItemInput = {
      category_id: categoryId,
      name: name.trim(),
      quantity: parseInt(quantity, 10) || 1,
      unit: unit.trim() || null,
      expiry_date: expiryDateStr,
      expiry_type: isInspection ? 'inspection' : 'expiry',
      storage_location_id: locationId,
      storage_note: storageNote.trim() || null,
      photo_path: finalPhotoPath,
      memo: memo.trim() || null,
      notify_days_before: notifyDays,
    };

    onSave(input);
  }

  return (
    <ScrollView contentContainerStyle={styles.scroll} keyboardShouldPersistTaps="handled">
      {/* Category */}
      <Label text="カテゴリ *" />
      <View style={styles.chipRow}>
        {categories.map((cat) => (
          <TouchableOpacity
            key={cat.id}
            style={[styles.chip, categoryId === cat.id && styles.chipActive]}
            onPress={() => setCategoryId(cat.id as CategoryId)}
          >
            <Text style={[styles.chipText, categoryId === cat.id && styles.chipTextActive]}>
              {cat.name}
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* Name */}
      <Label text="アイテム名 *" />
      <TextInput
        style={styles.input}
        value={name}
        onChangeText={setName}
        placeholder="例: 2Lミネラルウォーター"
        placeholderTextColor={Colors.textPlaceholder}
      />

      {/* Quantity + Unit */}
      <View style={styles.row}>
        <View style={{ flex: 1 }}>
          <Label text="数量" />
          <TextInput
            style={styles.input}
            value={quantity}
            onChangeText={setQuantity}
            keyboardType="numeric"
            placeholder="1"
            placeholderTextColor={Colors.textPlaceholder}
          />
        </View>
        <View style={{ width: 12 }} />
        <View style={{ flex: 1 }}>
          <Label text="単位" />
          <TextInput
            style={styles.input}
            value={unit}
            onChangeText={setUnit}
            placeholder="本・個・袋"
            placeholderTextColor={Colors.textPlaceholder}
          />
        </View>
      </View>

      {/* Expiry date */}
      <Label text={`${expiryLabel} *`} />
      <DateTimePicker
        value={expiryDate}
        mode="date"
        display="spinner"
        locale="ja-JP"
        onChange={(_, date) => date && setExpiryDate(date)}
        style={{ marginBottom: 8 }}
      />

      {/* Storage location */}
      <Label text="保管場所" />
      <View style={styles.chipRow}>
        {locations.map((loc) => (
          <TouchableOpacity
            key={loc.id}
            style={[styles.chip, locationId === loc.id && styles.chipActive]}
            onPress={() => setLocationId(locationId === loc.id ? null : loc.id)}
          >
            <Text style={[styles.chipText, locationId === loc.id && styles.chipTextActive]}>
              {loc.name}
            </Text>
          </TouchableOpacity>
        ))}
        <TouchableOpacity style={styles.chip} onPress={() => setShowNewLocation(!showNewLocation)}>
          <Text style={styles.chipText}>＋ 追加</Text>
        </TouchableOpacity>
      </View>
      {showNewLocation && (
        <View style={styles.row}>
          <TextInput
            style={[styles.input, { flex: 1 }]}
            value={newLocationName}
            onChangeText={setNewLocationName}
            placeholder="場所名"
            placeholderTextColor={Colors.textPlaceholder}
          />
          <TouchableOpacity style={styles.inlineBtn} onPress={handleAddLocation}>
            <Text style={{ color: Colors.primary }}>追加</Text>
          </TouchableOpacity>
        </View>
      )}

      {/* Storage note */}
      <Label text="保管場所の補足" />
      <TextInput
        style={styles.input}
        value={storageNote}
        onChangeText={setStorageNote}
        placeholder="例: 下段右側のボックス"
        placeholderTextColor={Colors.textPlaceholder}
      />

      {/* Photo */}
      <Label text="写真" />
      <TouchableOpacity style={styles.photoBtn} onPress={handlePhotoPress}>
        <Text style={{ color: Colors.primary }}>
          {photoPath ? '📷 写真を変更' : '📷 写真を追加'}
        </Text>
      </TouchableOpacity>

      {/* Memo */}
      <Label text="メモ" />
      <TextInput
        style={[styles.input, styles.multiline]}
        value={memo}
        onChangeText={setMemo}
        placeholder="自由メモ"
        placeholderTextColor={Colors.textPlaceholder}
        multiline
        numberOfLines={3}
      />

      {/* Notification timing */}
      <Label text="通知タイミング" />
      <View style={styles.chipRow}>
        {NOTIFY_OPTIONS.map((days) => (
          <TouchableOpacity
            key={days}
            style={[styles.chip, notifyDays === days && styles.chipActive]}
            onPress={() => setNotifyDays(days)}
          >
            <Text style={[styles.chipText, notifyDays === days && styles.chipTextActive]}>
              {days}日前
            </Text>
          </TouchableOpacity>
        ))}
      </View>

      {/* Actions */}
      <View style={styles.actions}>
        <TouchableOpacity style={styles.cancelBtn} onPress={onCancel}>
          <Text style={{ color: Colors.textSub, fontWeight: '500' }}>キャンセル</Text>
        </TouchableOpacity>
        <TouchableOpacity style={styles.saveBtn} onPress={handleSave}>
          <Text style={{ color: '#FFF', fontWeight: '600' }}>保存</Text>
        </TouchableOpacity>
      </View>
    </ScrollView>
  );
}

function Label({ text }: { text: string }) {
  return <Text style={labelStyle}>{text}</Text>;
}
const labelStyle: import('react-native').TextStyle = {
  fontSize: FontSize.caption,
  color: Colors.textSub,
  marginBottom: 6,
  marginTop: 16,
  fontWeight: '500',
};

const styles = StyleSheet.create({
  scroll: { padding: Spacing.screenH, paddingBottom: 60 },
  row: { flexDirection: 'row', alignItems: 'flex-end' },
  input: {
    backgroundColor: Colors.formBg,
    borderRadius: Radius.card,
    padding: 12,
    fontSize: FontSize.body,
    color: Colors.textMain,
    borderWidth: 1,
    borderColor: Colors.divider,
  },
  multiline: { height: 80, textAlignVertical: 'top' },
  chipRow: { flexDirection: 'row', flexWrap: 'wrap', gap: 8 },
  chip: {
    paddingHorizontal: 12, paddingVertical: 7,
    borderRadius: Radius.chip, borderWidth: 1,
    borderColor: Colors.divider, backgroundColor: Colors.cardBg,
  },
  chipActive: { backgroundColor: Colors.primaryTint, borderColor: Colors.primaryRing },
  chipText: { fontSize: FontSize.caption, color: Colors.textSub },
  chipTextActive: { color: Colors.primaryDeep, fontWeight: '600' },
  photoBtn: {
    padding: 12, borderRadius: Radius.card,
    borderWidth: 1, borderColor: Colors.primaryRing,
    borderStyle: 'dashed', alignItems: 'center',
  },
  inlineBtn: { padding: 12, marginLeft: 8 },
  actions: { flexDirection: 'row', gap: 12, marginTop: 32 },
  cancelBtn: {
    flex: 1, padding: 14, borderRadius: Radius.card,
    borderWidth: 1, borderColor: Colors.divider, alignItems: 'center',
  },
  saveBtn: {
    flex: 2, padding: 14, borderRadius: Radius.card,
    backgroundColor: Colors.primary, alignItems: 'center',
  },
});
```

- [ ] **Step 2: Implement app/item/new.tsx**

```typescript
// app/item/new.tsx
import React from 'react';
import { SafeAreaView, StyleSheet } from 'react-native';
import { useRouter } from 'expo-router';
import { useAppStore } from '@/store/useAppStore';
import { ItemForm } from '@/components/ItemForm';
import { scheduleItemNotification } from '@/services/notifications';
import { Colors } from '@/theme/tokens';

export default function NewItemScreen() {
  const router = useRouter();
  const { addItem, editItem } = useAppStore();

  async function handleSave(input: Parameters<typeof addItem>[0]) {
    const item = addItem(input);
    const notificationId = await scheduleItemNotification(item);
    if (notificationId) {
      editItem(item.id, { notification_id: notificationId });
    }
    router.back();
  }

  return (
    <SafeAreaView style={styles.safe}>
      <ItemForm onSave={handleSave} onCancel={() => router.back()} />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
});
```

- [ ] **Step 3: Implement app/item/[id]/edit.tsx**

```typescript
// app/item/[id]/edit.tsx
import React from 'react';
import { SafeAreaView, StyleSheet } from 'react-native';
import { useLocalSearchParams, useRouter } from 'expo-router';
import { useAppStore } from '@/store/useAppStore';
import { ItemForm } from '@/components/ItemForm';
import { scheduleItemNotification, cancelItemNotification } from '@/services/notifications';
import { Colors } from '@/theme/tokens';

export default function EditItemScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  const router = useRouter();
  const { items, editItem } = useAppStore();
  const item = items.find((i) => i.id === id);

  if (!item) return null;

  async function handleSave(input: Parameters<typeof editItem>[1]) {
    // Cancel old notification
    if (item!.notification_id) {
      await cancelItemNotification(item!.notification_id);
    }

    const updated = editItem(id, input);

    // Schedule new notification
    const notificationId = await scheduleItemNotification(updated);
    if (notificationId) {
      editItem(id, { notification_id: notificationId });
    }

    router.back();
  }

  return (
    <SafeAreaView style={styles.safe}>
      <ItemForm initialValues={item} onSave={handleSave} onCancel={() => router.back()} />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
});
```

- [ ] **Step 4: Test add + edit flow end to end**

1. Tap "アイテムを追加" on Home → fill form → Save
2. Verify item appears in category list and home counts update
3. Tap item → Item detail shows
4. Tap 編集 → Edit form pre-fills with existing values → Save
5. Verify updates persist (reload app or re-navigate)

- [ ] **Step 5: Commit**

```bash
git add src/components/ItemForm.tsx app/item/new.tsx "app/item/[id]/edit.tsx"
git commit -m "feat: implement ItemForm shared component and new/edit screens"
```

---

## Task 16: All Items List Screen

**Files:**
- Modify: `app/(tabs)/list.tsx`

- [ ] **Step 1: Implement ListScreen**

```typescript
// app/(tabs)/list.tsx
import React, { useMemo, useState } from 'react';
import { View, TextInput, FlatList, StyleSheet, SafeAreaView, Text } from 'react-native';
import { useRouter } from 'expo-router';
import { useAppStore } from '@/store/useAppStore';
import { ItemCard } from '@/components/ItemCard';
import { getExpiryStatus } from '@/domain/expiry';
import { Colors, Spacing, FontSize, Radius } from '@/theme/tokens';
import dayjs from 'dayjs';

type FilterKey = 'all' | 'expired' | 'soon';

const FILTER_LABELS: Record<FilterKey, string> = {
  all: 'すべて',
  expired: '期限切れ',
  soon: '30日以内',
};

export default function ListScreen() {
  const router = useRouter();
  const { items, locations } = useAppStore();
  const [search, setSearch] = useState('');
  const [filter, setFilter] = useState<FilterKey>('all');
  const today = dayjs().format('YYYY-MM-DD');

  const filtered = useMemo(() => {
    let result = items;

    if (filter === 'expired') {
      result = result.filter((i) => getExpiryStatus(i.expiry_date, today) === 'expired');
    } else if (filter === 'soon') {
      result = result.filter((i) => getExpiryStatus(i.expiry_date, today) === 'soon');
    }

    if (search.trim()) {
      const q = search.trim().toLowerCase();
      result = result.filter((i) => i.name.toLowerCase().includes(q));
    }

    return result;
  }, [items, filter, search, today]);

  function getLocationName(locationId: string | null): string | undefined {
    if (!locationId) return undefined;
    return locations.find((l) => l.id === locationId)?.name;
  }

  return (
    <SafeAreaView style={styles.safe}>
      <View style={styles.header}>
        <TextInput
          style={styles.search}
          value={search}
          onChangeText={setSearch}
          placeholder="アイテムを検索..."
          placeholderTextColor={Colors.textPlaceholder}
          clearButtonMode="while-editing"
        />
        <View style={styles.filters}>
          {(Object.keys(FILTER_LABELS) as FilterKey[]).map((key) => (
            <TouchableOpacity
              key={key}
              style={[styles.filterChip, filter === key && styles.filterChipActive]}
              onPress={() => setFilter(key)}
            >
              <Text style={[styles.filterText, filter === key && styles.filterTextActive]}>
                {FILTER_LABELS[key]}
              </Text>
            </TouchableOpacity>
          ))}
        </View>
      </View>
      <FlatList
        data={filtered}
        keyExtractor={(i) => i.id}
        contentContainerStyle={styles.list}
        ListEmptyComponent={
          <View style={styles.empty}>
            <Text style={styles.emptyText}>
              {search ? '検索結果がありません' : 'アイテムがありません'}
            </Text>
          </View>
        }
        renderItem={({ item }) => (
          <ItemCard
            item={item}
            locationName={getLocationName(item.storage_location_id)}
            onPress={() => router.push(`/item/${item.id}`)}
          />
        )}
      />
    </SafeAreaView>
  );
}

// Need to add TouchableOpacity import
import { TouchableOpacity } from 'react-native';

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
  header: { padding: Spacing.screenH, paddingBottom: 0 },
  search: {
    backgroundColor: Colors.formBg,
    borderRadius: Radius.card,
    padding: 12,
    fontSize: FontSize.body,
    color: Colors.textMain,
    borderWidth: 1,
    borderColor: Colors.divider,
    marginBottom: 12,
  },
  filters: { flexDirection: 'row', gap: 8, marginBottom: 12 },
  filterChip: {
    paddingHorizontal: 14, paddingVertical: 7,
    borderRadius: Radius.chip, borderWidth: 1, borderColor: Colors.divider,
  },
  filterChipActive: { backgroundColor: Colors.primaryTint, borderColor: Colors.primaryRing },
  filterText: { fontSize: FontSize.caption, color: Colors.textSub },
  filterTextActive: { color: Colors.primaryDeep, fontWeight: '600' },
  list: { padding: Spacing.screenH, paddingTop: 4, paddingBottom: 40 },
  empty: { alignItems: 'center', paddingTop: 60 },
  emptyText: { fontSize: FontSize.body, color: Colors.textMuted },
});
```

- [ ] **Step 2: Commit**

```bash
git add "app/(tabs)/list.tsx"
git commit -m "feat: implement All Items list screen with search and filter"
```

---

## Task 17: Settings Screen

**Files:**
- Modify: `app/(tabs)/settings.tsx`

- [ ] **Step 1: Implement SettingsScreen**

```typescript
// app/(tabs)/settings.tsx
import React, { useState } from 'react';
import {
  View, Text, FlatList, TextInput, TouchableOpacity,
  StyleSheet, SafeAreaView, Alert,
} from 'react-native';
import { useAppStore } from '@/store/useAppStore';
import { Colors, Spacing, FontSize, Radius } from '@/theme/tokens';

export default function SettingsScreen() {
  const { locations, addLocation, editLocation, removeLocation } = useAppStore();
  const [newName, setNewName] = useState('');
  const [editingId, setEditingId] = useState<string | null>(null);
  const [editingName, setEditingName] = useState('');

  function handleAdd() {
    if (!newName.trim()) return;
    addLocation(newName.trim());
    setNewName('');
  }

  function handleStartEdit(id: string, name: string) {
    setEditingId(id);
    setEditingName(name);
  }

  function handleSaveEdit() {
    if (!editingId || !editingName.trim()) return;
    editLocation(editingId, editingName.trim());
    setEditingId(null);
    setEditingName('');
  }

  function handleDelete(id: string, name: string) {
    Alert.alert('削除', `「${name}」を削除しますか？\n関連するアイテムの保管場所設定が解除されます。`, [
      { text: 'キャンセル', style: 'cancel' },
      { text: '削除', style: 'destructive', onPress: () => removeLocation(id) },
    ]);
  }

  return (
    <SafeAreaView style={styles.safe}>
      <FlatList
        data={locations}
        keyExtractor={(l) => l.id}
        contentContainerStyle={styles.list}
        ListHeaderComponent={
          <>
            <Text style={styles.sectionTitle}>保管場所</Text>
            <View style={styles.addRow}>
              <TextInput
                style={[styles.input, { flex: 1 }]}
                value={newName}
                onChangeText={setNewName}
                placeholder="新しい場所を追加..."
                placeholderTextColor={Colors.textPlaceholder}
                returnKeyType="done"
                onSubmitEditing={handleAdd}
              />
              <TouchableOpacity style={styles.addBtn} onPress={handleAdd}>
                <Text style={{ color: '#FFF', fontWeight: '600' }}>追加</Text>
              </TouchableOpacity>
            </View>
          </>
        }
        renderItem={({ item: loc }) => (
          <View style={styles.row}>
            {editingId === loc.id ? (
              <>
                <TextInput
                  style={[styles.input, { flex: 1 }]}
                  value={editingName}
                  onChangeText={setEditingName}
                  autoFocus
                  returnKeyType="done"
                  onSubmitEditing={handleSaveEdit}
                />
                <TouchableOpacity style={styles.actionBtn} onPress={handleSaveEdit}>
                  <Text style={{ color: Colors.primary }}>保存</Text>
                </TouchableOpacity>
              </>
            ) : (
              <>
                <Text style={styles.locName}>{loc.name}</Text>
                <TouchableOpacity style={styles.actionBtn} onPress={() => handleStartEdit(loc.id, loc.name)}>
                  <Text style={{ color: Colors.textSub }}>編集</Text>
                </TouchableOpacity>
                {!loc.is_default && (
                  <TouchableOpacity onPress={() => handleDelete(loc.id, loc.name)}>
                    <Text style={{ color: Colors.expired.text }}>削除</Text>
                  </TouchableOpacity>
                )}
              </>
            )}
          </View>
        )}
        ListFooterComponent={
          <View style={{ marginTop: 40 }}>
            <Text style={styles.versionText}>防災ストック v1.0.0</Text>
          </View>
        }
      />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
  list: { padding: Spacing.screenH, paddingBottom: 40 },
  sectionTitle: {
    fontSize: FontSize.sub, color: Colors.textMuted, fontWeight: '500',
    textTransform: 'uppercase', marginBottom: 12,
  },
  addRow: { flexDirection: 'row', gap: 8, marginBottom: 20 },
  input: {
    backgroundColor: Colors.cardBg, borderRadius: Radius.card,
    padding: 12, fontSize: FontSize.body, color: Colors.textMain,
    borderWidth: 1, borderColor: Colors.divider,
  },
  addBtn: {
    backgroundColor: Colors.primary, borderRadius: Radius.card,
    paddingHorizontal: 16, justifyContent: 'center',
  },
  row: {
    flexDirection: 'row', alignItems: 'center',
    paddingVertical: 12, borderBottomWidth: 1, borderBottomColor: Colors.divider,
    gap: 8,
  },
  locName: { flex: 1, fontSize: FontSize.body, color: Colors.textMain },
  actionBtn: { paddingHorizontal: 8 },
  versionText: { textAlign: 'center', color: Colors.textMuted, fontSize: FontSize.caption },
});
```

- [ ] **Step 2: Commit**

```bash
git add "app/(tabs)/settings.tsx"
git commit -m "feat: implement Settings screen with storage location CRUD"
```

---

## Task 18: Notifications Screen + App Launch Sync

**Files:**
- Modify: `app/(tabs)/notifications.tsx`
- Modify: `app/_layout.tsx`

- [ ] **Step 1: Implement NotificationsScreen**

```typescript
// app/(tabs)/notifications.tsx
import React, { useEffect, useState } from 'react';
import {
  View, Text, FlatList, Switch, StyleSheet, SafeAreaView,
} from 'react-native';
import * as Notifications from 'expo-notifications';
import dayjs from 'dayjs';
import { useAppStore } from '@/store/useAppStore';
import { getExpiryStatus, getDaysUntil, formatExpiryDate } from '@/domain/expiry';
import { ExpiryBadge } from '@/components/ExpiryBadge';
import { Colors, Spacing, FontSize, Radius } from '@/theme/tokens';

export default function NotificationsScreen() {
  const { items } = useAppStore();
  const [notificationsEnabled, setNotificationsEnabled] = useState(true);
  const today = dayjs().format('YYYY-MM-DD');

  useEffect(() => {
    Notifications.getPermissionsAsync().then(({ status }) => {
      setNotificationsEnabled(status === 'granted');
    });
  }, []);

  async function handleToggle(value: boolean) {
    if (value) {
      const { status } = await Notifications.requestPermissionsAsync();
      setNotificationsEnabled(status === 'granted');
    } else {
      await Notifications.cancelAllScheduledNotificationsAsync();
      setNotificationsEnabled(false);
    }
  }

  const upcomingItems = [...items]
    .filter((i) => {
      const d = getDaysUntil(i.expiry_date, today);
      return d <= 60; // show items with expiry within 60 days
    })
    .sort((a, b) => a.expiry_date.localeCompare(b.expiry_date));

  return (
    <SafeAreaView style={styles.safe}>
      <FlatList
        data={upcomingItems}
        keyExtractor={(i) => i.id}
        contentContainerStyle={styles.list}
        ListHeaderComponent={
          <>
            <Text style={styles.title}>通知</Text>
            <View style={styles.toggleRow}>
              <Text style={styles.toggleLabel}>通知を有効にする</Text>
              <Switch
                value={notificationsEnabled}
                onValueChange={handleToggle}
                trackColor={{ true: Colors.primary }}
              />
            </View>
            {upcomingItems.length > 0 && (
              <Text style={styles.sectionLabel}>期限が近いアイテム</Text>
            )}
          </>
        }
        ListEmptyComponent={
          <View style={styles.empty}>
            <Text style={styles.emptyText}>期限が近いアイテムはありません</Text>
          </View>
        }
        renderItem={({ item }) => {
          const days = getDaysUntil(item.expiry_date, today);
          const status = getExpiryStatus(item.expiry_date, today);
          return (
            <View style={styles.item}>
              <View style={{ flex: 1 }}>
                <Text style={styles.itemName}>{item.name}</Text>
                <Text style={styles.itemDate}>{formatExpiryDate(item.expiry_date)}</Text>
              </View>
              <ExpiryBadge
                status={status}
                label={days <= 0 ? `${Math.abs(days)}日超過` : `あと${days}日`}
              />
            </View>
          );
        }}
      />
    </SafeAreaView>
  );
}

const styles = StyleSheet.create({
  safe: { flex: 1, backgroundColor: Colors.appBg },
  list: { padding: Spacing.screenH, paddingBottom: 40 },
  title: { fontSize: FontSize.title, fontWeight: '700', color: Colors.textMain, marginBottom: 20 },
  toggleRow: {
    flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between',
    backgroundColor: Colors.cardBg, borderRadius: Radius.card,
    padding: Spacing.cardPad, marginBottom: 20,
    borderWidth: 1, borderColor: Colors.divider,
  },
  toggleLabel: { fontSize: FontSize.body, color: Colors.textMain },
  sectionLabel: {
    fontSize: FontSize.caption, color: Colors.textMuted, fontWeight: '500',
    textTransform: 'uppercase', marginBottom: 12,
  },
  item: {
    flexDirection: 'row', alignItems: 'center',
    paddingVertical: 14, borderBottomWidth: 1, borderBottomColor: Colors.divider,
  },
  itemName: { fontSize: FontSize.body, color: Colors.textMain, fontWeight: '500' },
  itemDate: { fontSize: FontSize.caption, color: Colors.textSub, marginTop: 2 },
  empty: { alignItems: 'center', paddingTop: 60 },
  emptyText: { fontSize: FontSize.body, color: Colors.textMuted },
});
```

- [ ] **Step 2: Add notification sync to app/_layout.tsx**

Update the `useEffect` in `app/_layout.tsx` to sync notifications after loading:

```typescript
// In app/_layout.tsx, update the useEffect:
import { syncNotificationsForItems } from '@/services/notifications';

// Replace existing useEffect with:
useEffect(() => {
  initDb();
  loadAll();
}, []);

// Add a second useEffect that runs after items are loaded:
const items = useAppStore((s) => s.items);
const editItem = useAppStore((s) => s.editItem);

useEffect(() => {
  if (items.length === 0) return;
  syncNotificationsForItems(items, (id, notificationId) => {
    editItem(id, { notification_id: notificationId });
  });
}, [items.length]); // run once after initial load
```

The complete updated `app/_layout.tsx`:

```typescript
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import { StatusBar } from 'expo-status-bar';
import { initDb } from '@/db/database';
import { useAppStore } from '@/store/useAppStore';
import { syncNotificationsForItems } from '@/services/notifications';

export default function RootLayout() {
  const loadAll = useAppStore((s) => s.loadAll);
  const items = useAppStore((s) => s.items);
  const editItem = useAppStore((s) => s.editItem);

  useEffect(() => {
    initDb();
    loadAll();
  }, []);

  useEffect(() => {
    if (items.length === 0) return;
    syncNotificationsForItems(items, (id, notificationId) => {
      editItem(id, { notification_id: notificationId });
    });
  }, [items.length]);

  return (
    <>
      <StatusBar style="dark" />
      <Stack screenOptions={{ headerShown: false }}>
        <Stack.Screen name="(tabs)" />
        <Stack.Screen
          name="item/new"
          options={{ presentation: 'modal', headerShown: true, title: 'アイテムを追加' }}
        />
        <Stack.Screen
          name="item/[id]/index"
          options={{ headerShown: true, title: '詳細' }}
        />
        <Stack.Screen
          name="item/[id]/edit"
          options={{ presentation: 'modal', headerShown: true, title: '編集' }}
        />
        <Stack.Screen
          name="category/[id]"
          options={{ headerShown: true }}
        />
      </Stack>
    </>
  );
}
```

- [ ] **Step 3: Commit**

```bash
git add "app/(tabs)/notifications.tsx" app/_layout.tsx
git commit -m "feat: add Notifications screen and startup notification sync"
```

---

## Task 19: Final Integration Test & Polish

- [ ] **Step 1: Run full test suite**

```bash
npx jest
```

Expected: All tests pass (at minimum expiry.test.ts)

- [ ] **Step 2: Manual smoke test on device**

Test every screen and flow:

1. **Home screen** — counts show 0, all 5 categories visible
2. **Add item** — fill all fields, take/pick photo, save → appears in Home counts
3. **Category screen** — item appears with correct badge
4. **Item detail** — photo, info, badge visible
5. **Quick update** — tap "補充・期限を更新する", change date, save → detail updates
6. **Edit item** — pre-filled form, change name, save → updates everywhere
7. **Delete item** — confirmation → item removed
8. **List screen** — search works, filter by "期限切れ" works
9. **Notifications screen** — toggle shows permission dialog, upcoming items listed
10. **Settings** — add/edit/delete custom storage location, verify it appears in item form
11. **Kill + reopen app** — data persists from SQLite

- [ ] **Step 3: Fix any issues found in smoke test**

- [ ] **Step 4: Final commit**

```bash
git add -A
git commit -m "feat: complete bousai-stock MVP"
```

---

## Self-Review Against Spec

### Spec Coverage Check

| Spec Section | Covered? | Task |
|---|---|---|
| 3.1 categories schema | ✅ | Task 5 |
| 3.2 storage_locations schema | ✅ | Task 5 |
| 3.3 items schema (all columns) | ✅ | Task 5 |
| 3.4 indexes | ✅ | Task 5 |
| 4.1 Home screen (metrics, banner, category cards, FAB) | ✅ | Task 10 |
| 4.2 Category list (cards, thumbnail, badge, sort) | ✅ | Task 11 |
| 4.3 Item detail (photo, warning box, details, quick update) | ✅ | Task 12 |
| 4.4 New/Edit form (all fields, notify timing, photo) | ✅ | Task 15 |
| 4.5 Notifications screen (list, toggle) | ✅ | Task 18 |
| 4.6 Settings (storage location CRUD) | ✅ | Task 17 |
| 5. Notification logic (schedule, cancel, sync, morning 9am) | ✅ | Tasks 14, 18 |
| 6.1 Colors (olive green, red/orange for status, category colors) | ✅ | Task 4 |
| 6.2 Typography (2 weights, 12px cards, pill badges) | ✅ | Tasks 4, 9 |
| expo-sqlite (offline, structured data) | ✅ | Tasks 5, 6 |
| expo-router | ✅ | Task 8 |
| expo-notifications (local only) | ✅ | Task 14 |
| expo-image-picker + expo-file-system | ✅ | Task 13 |
| Zustand store | ✅ | Task 7 |
| KrayNote code conventions | ✅ | All tasks |
