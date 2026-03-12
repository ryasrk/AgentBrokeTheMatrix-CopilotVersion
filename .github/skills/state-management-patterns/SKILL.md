---
name: state-management-patterns
description: Frontend state management patterns for React using useState, useReducer, Context, Zustand, Redux Toolkit, Jotai, and signals. Covers local vs global state, server state, and derived state.
---

# State Management Patterns

Patterns for managing frontend state effectively — local, global, server, and derived.

## When to Activate

- Deciding between local state, context, and external stores
- Implementing global state with Zustand, Redux Toolkit, or Jotai
- Managing server state with React Query / SWR
- Optimizing re-renders caused by state changes
- Implementing undo/redo, optimistic updates, or real-time sync
- Migrating between state management solutions

## Core Principles

### 1. State Hierarchy

Use the simplest solution that works. Escalate only when needed.

```
Level 1: Local State (useState)
  → Component-specific UI state (open/closed, input values)

Level 2: Lifted State (parent component)
  → Shared between 2-3 sibling components

Level 3: Context (useContext)
  → Theme, locale, auth user — rarely changing global values

Level 4: External Store (Zustand, Redux)
  → Complex shared state, many consumers, needs middleware

Level 5: Server State (React Query, SWR)
  → Data from API, needs caching/syncing/revalidation
```

### 2. Single Source of Truth

Never duplicate state. Derive when possible.

```typescript
// BAD: Duplicated/derived state stored separately
const [items, setItems] = useState<Item[]>([]);
const [filteredItems, setFilteredItems] = useState<Item[]>([]);
const [itemCount, setItemCount] = useState(0);

// GOOD: Derive from single source
const [items, setItems] = useState<Item[]>([]);
const [filter, setFilter] = useState('');
const filteredItems = useMemo(
  () => items.filter(i => i.name.includes(filter)),
  [items, filter]
);
const itemCount = filteredItems.length;  // Just compute it!
```

### 3. Collocate State With Its Consumer

Keep state as close to where it's used as possible.

```typescript
// BAD: Everything in global store
const useStore = create((set) => ({
  modalOpen: false,        // Only used in one component!
  searchQuery: '',          // Only used in SearchBar!
  plates: [],              // OK — used by many components
}));

// GOOD: Local state stays local
function SearchBar() {
  const [query, setQuery] = useState('');  // Local — only SearchBar cares
  ...
}
```

## Local State Patterns

### useState for Simple State

```typescript
// Toggle, counters, form inputs
const [isOpen, setIsOpen] = useState(false);
const toggle = useCallback(() => setIsOpen(prev => !prev), []);
```

### useReducer for Complex State

```typescript
type DetectionState = {
  detections: Detection[];
  filter: string;
  sortBy: 'date' | 'confidence';
  selectedId: string | null;
};

type Action =
  | { type: 'SET_DETECTIONS'; payload: Detection[] }
  | { type: 'SET_FILTER'; payload: string }
  | { type: 'SET_SORT'; payload: 'date' | 'confidence' }
  | { type: 'SELECT'; payload: string | null };

function detectionReducer(state: DetectionState, action: Action): DetectionState {
  switch (action.type) {
    case 'SET_DETECTIONS':
      return { ...state, detections: action.payload };
    case 'SET_FILTER':
      return { ...state, filter: action.payload };
    case 'SET_SORT':
      return { ...state, sortBy: action.payload };
    case 'SELECT':
      return { ...state, selectedId: action.payload };
    default:
      return state;
  }
}
```

## Zustand (Recommended External Store)

```typescript
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';
import { immer } from 'zustand/middleware/immer';

interface PlateStore {
  plates: Plate[];
  selectedPlateId: string | null;

  // Actions
  setPlates: (plates: Plate[]) => void;
  selectPlate: (id: string | null) => void;
  addPlate: (plate: Plate) => void;
  removePlate: (id: string) => void;
}

const usePlateStore = create<PlateStore>()(
  devtools(
    immer((set) => ({
      plates: [],
      selectedPlateId: null,

      setPlates: (plates) => set((state) => { state.plates = plates }),
      selectPlate: (id) => set((state) => { state.selectedPlateId = id }),
      addPlate: (plate) => set((state) => { state.plates.push(plate) }),
      removePlate: (id) => set((state) => {
        state.plates = state.plates.filter(p => p.id !== id);
      }),
    })),
    { name: 'plate-store' }
  )
);

// Selectors for minimal re-renders
const usePlates = () => usePlateStore((s) => s.plates);
const useSelectedPlate = () => usePlateStore((s) =>
  s.plates.find(p => p.id === s.selectedPlateId)
);
```

## Server State (React Query / TanStack Query)

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Fetch with caching and revalidation
function usePlates(filter: string) {
  return useQuery({
    queryKey: ['plates', filter],
    queryFn: () => api.getPlates({ filter }),
    staleTime: 30_000,           // Fresh for 30s
    gcTime: 5 * 60_000,         // Keep in cache 5min
    refetchOnWindowFocus: true,
    retry: 2,
  });
}

// Optimistic update mutation
function useRegisterPlate() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: api.registerPlate,
    onMutate: async (newPlate) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['plates'] });

      // Snapshot previous value
      const previous = queryClient.getQueryData(['plates']);

      // Optimistically update
      queryClient.setQueryData(['plates'], (old: Plate[]) =>
        [...old, { ...newPlate, id: 'temp', status: 'pending' }]
      );

      return { previous };
    },
    onError: (_err, _vars, context) => {
      // Rollback on error
      queryClient.setQueryData(['plates'], context?.previous);
    },
    onSettled: () => {
      // Refetch to sync with server
      queryClient.invalidateQueries({ queryKey: ['plates'] });
    },
  });
}
```

## Context Pattern (Low-Frequency Updates Only)

```typescript
// Good use: Theme, locale, auth — changes rarely
interface AuthContextValue {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
}

const AuthContext = createContext<AuthContextValue | undefined>(undefined);

function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be used within AuthProvider');
  return ctx;
}
```

## Avoiding Re-Renders

```typescript
// Split context to separate frequently changing values
const CountContext = createContext(0);        // Changes often
const DispatchContext = createContext(dispatch); // Stable reference

// Use selectors with Zustand (automatic)
const count = useStore(state => state.count); // Only re-renders when count changes

// Memoize expensive derived state
const expensiveResult = useMemo(() =>
  plates.filter(p => p.region === selectedRegion).sort(sortFn),
  [plates, selectedRegion, sortFn]
);
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Global state for local UI state | Use useState in the component |
| Derived state in store | Compute with useMemo or selectors |
| Fetching in useEffect + useState | Use React Query / SWR |
| Context for frequently changing values | Use Zustand or Jotai |
| Re-rendering entire tree on state change | Split context, use selectors |
| useEffect to sync state | Derive during render instead |
| Storing stale copies of server data | Use React Query with staleTime |
| No loading/error states | Always handle all async states |
