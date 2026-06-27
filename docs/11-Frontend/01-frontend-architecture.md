# Frontend Architecture

> **Category:** Frontend Engineering
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Architecture Patterns](#1-architecture-patterns)
2. [React State Management](#2-react-state-management)
3. [Performance Optimization](#3-performance-optimization)
4. [TypeScript in React](#4-typescript-in-react)
5. [Data Fetching Patterns](#5-data-fetching-patterns)
6. [Testing React Applications](#6-testing-react-applications)
7. [Accessibility](#7-accessibility)
8. [Security in Frontend](#8-security-in-frontend)
9. [Build Optimization](#9-build-optimization)
10. [Code Quality](#10-code-quality)

---

## 1. Architecture Patterns

### Feature-Based Folder Structure

```
src/
├── features/                 ← Feature modules (self-contained)
│   ├── bookings/
│   │   ├── api/
│   │   │   └── use-bookings.ts      # Data fetching hook (TanStack Query)
│   │   ├── components/
│   │   │   ├── booking-card.tsx     # Presentational component
│   │   │   ├── booking-list.tsx     # Container/list component
│   │   │   └── booking-form.tsx     # Form component
│   │   ├── hooks/
│   │   │   └── use-booking-form.ts  # Form state/validation logic
│   │   ├── utils/
│   │   │   └── booking.utils.ts     # Booking-specific helpers
│   │   ├── types/
│   │   │   └── booking.types.ts     # TypeScript interfaces
│   │   └── index.ts                 # Public API of this feature
│   └── auth/
│       ├── api/
│       ├── components/
│       └── context/
│
├── shared/                   ← Shared across features
│   ├── components/           # Generic UI components
│   │   ├── button.tsx
│   │   ├── modal.tsx
│   │   └── data-table.tsx
│   ├── hooks/
│   │   ├── use-debounce.ts
│   │   └── use-local-storage.ts
│   ├── utils/
│   │   ├── format-date.ts
│   │   └── cn.ts            # Tailwind class merging
│   └── types/
│       └── api.types.ts
│
├── pages/                    ← Next.js pages / React Router routes
├── app/                      ← App-level providers, router, layout
│   ├── providers.tsx          # React Query, Auth, Theme providers
│   └── router.tsx
└── lib/
    ├── api-client.ts         # Axios/fetch wrapper
    └── constants.ts
```

### Component Separation Pattern

```tsx
// Pattern: separate data fetching from rendering

// 1. Container component (fetches data, handles state)
// features/bookings/components/booking-list.tsx
export function BookingList() {
  const { data, isLoading, error } = useBookings()
  
  if (isLoading) return <BookingListSkeleton />
  if (error) return <ErrorMessage error={error} />
  if (!data?.length) return <EmptyState message="No bookings yet" />
  
  return (
    <div className="space-y-4">
      {data.map(booking => (
        <BookingCard key={booking.id} booking={booking} />
      ))}
    </div>
  )
}

// 2. Presentational component (pure rendering, no side effects)
// features/bookings/components/booking-card.tsx
interface BookingCardProps {
  booking: Booking
  onCancel?: (id: string) => void
}

export function BookingCard({ booking, onCancel }: BookingCardProps) {
  return (
    <article className="rounded-lg border p-4">
      <h3 className="font-semibold">{booking.service.name}</h3>
      <time dateTime={booking.scheduledAt}>
        {formatDate(booking.scheduledAt, 'dd/MM/yyyy HH:mm')}
      </time>
      <Badge variant={getStatusVariant(booking.status)}>
        {booking.status}
      </Badge>
      {onCancel && booking.status === 'pending' && (
        <button onClick={() => onCancel(booking.id)}>Cancel</button>
      )}
    </article>
  )
}
```

---

## 2. React State Management

### State Categories

```
Local UI state:      → useState / useReducer (form inputs, modal open/close, hover)
Server state:        → TanStack Query (API responses, caching, refetching)
Global app state:    → Zustand / Context API (auth user, theme, notifications)
URL state:           → URL params/query string (current page, filters, selected tab)

Rule: put state as low as possible in the component tree
Don't hoist to global store unless multiple disconnected components need it
```

### Zustand for Global State

```typescript
// store/auth.store.ts
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface AuthState {
  user: User | null
  accessToken: string | null
  isAuthenticated: boolean
  
  // Actions
  login: (credentials: LoginCredentials) => Promise<void>
  logout: () => Promise<void>
  refreshToken: () => Promise<void>
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      accessToken: null,
      isAuthenticated: false,
      
      login: async (credentials) => {
        const { data } = await authApi.login(credentials)
        set({
          user: data.user,
          accessToken: data.accessToken,
          isAuthenticated: true,
        })
      },
      
      logout: async () => {
        await authApi.logout()
        set({ user: null, accessToken: null, isAuthenticated: false })
      },
      
      refreshToken: async () => {
        try {
          const { data } = await authApi.refresh()
          set({ accessToken: data.accessToken })
        } catch {
          set({ user: null, accessToken: null, isAuthenticated: false })
        }
      },
    }),
    {
      name: 'auth-store',
      storage: createJSONStorage(() => sessionStorage),  // sessionStorage (not localStorage!)
      partialize: (state) => ({ user: state.user }),     // Don't persist token in storage
    }
  )
)

// Usage in component
function Header() {
  const { user, logout, isAuthenticated } = useAuthStore()
  
  if (!isAuthenticated) return <LoginButton />
  
  return (
    <header>
      <span>Olá, {user?.name}</span>
      <button onClick={logout}>Sair</button>
    </header>
  )
}
```

### Complex Form State (React Hook Form + Zod)

```tsx
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const CreateBookingSchema = z.object({
  serviceId: z.string().uuid('Serviço inválido'),
  scheduledAt: z.string().min(1, 'Data obrigatória').refine(
    d => new Date(d) > new Date(),
    { message: 'A data deve ser no futuro' }
  ),
  notes: z.string().max(500, 'Máximo 500 caracteres').optional(),
})

type CreateBookingForm = z.infer<typeof CreateBookingSchema>

function BookingForm() {
  const createBooking = useCreateBooking()  // TanStack Query mutation
  
  const form = useForm<CreateBookingForm>({
    resolver: zodResolver(CreateBookingSchema),
    defaultValues: { serviceId: '', scheduledAt: '', notes: '' },
  })
  
  const onSubmit = form.handleSubmit(async (data) => {
    await createBooking.mutateAsync(data)
    form.reset()
  })
  
  return (
    <form onSubmit={onSubmit} noValidate>
      <div>
        <label htmlFor="serviceId">Serviço</label>
        <select id="serviceId" {...form.register('serviceId')}>
          <option value="">Selecione um serviço</option>
        </select>
        {form.formState.errors.serviceId && (
          <span role="alert">{form.formState.errors.serviceId.message}</span>
        )}
      </div>
      
      <div>
        <label htmlFor="scheduledAt">Data e hora</label>
        <input
          id="scheduledAt"
          type="datetime-local"
          {...form.register('scheduledAt')}
          aria-invalid={!!form.formState.errors.scheduledAt}
        />
        {form.formState.errors.scheduledAt && (
          <span role="alert">{form.formState.errors.scheduledAt.message}</span>
        )}
      </div>
      
      <button type="submit" disabled={form.formState.isSubmitting || createBooking.isPending}>
        {form.formState.isSubmitting ? 'Agendando...' : 'Confirmar Agendamento'}
      </button>
      
      {createBooking.isError && (
        <div role="alert">Erro ao criar agendamento: {createBooking.error.message}</div>
      )}
    </form>
  )
}
```

---

## 3. Performance Optimization

### React.memo and useMemo

```tsx
// Only memoize when:
// 1. Component renders frequently AND
// 2. Renders are expensive AND
// 3. Props don't change often

// Expensive list item (memoize so parent rerenders don't cause all items to rerender)
const BookingCard = React.memo(({ booking }: { booking: Booking }) => {
  return <div>{/* expensive render */}</div>
}, (prevProps, nextProps) => {
  // Custom equality: only rerender if booking ID or status changed
  return prevProps.booking.id === nextProps.booking.id &&
         prevProps.booking.status === nextProps.booking.status
})

// useMemo for expensive computations (not for objects to prevent rerenders!)
function BookingStats({ bookings }: { bookings: Booking[] }) {
  const stats = useMemo(() => ({
    total: bookings.length,
    pending: bookings.filter(b => b.status === 'pending').length,
    confirmed: bookings.filter(b => b.status === 'confirmed').length,
    revenue: bookings
      .filter(b => b.status === 'completed')
      .reduce((sum, b) => sum + b.price, 0),
  }), [bookings])  // Recalculate only when bookings array changes
  
  return <div>{stats.total} total, {stats.pending} pending</div>
}

// useCallback for stable function references
function BookingList() {
  const { mutate: cancelBooking } = useCancelBooking()
  
  const handleCancel = useCallback((bookingId: string) => {
    cancelBooking({ bookingId })
  }, [cancelBooking])  // Stable reference → BookingCard doesn't rerender when parent does
  
  return bookings.map(b => <BookingCard key={b.id} booking={b} onCancel={handleCancel} />)
}
```

### Code Splitting and Lazy Loading

```tsx
import { lazy, Suspense } from 'react'

// Split by route (most impactful)
const AdminDashboard = lazy(() => import('./features/admin/admin-dashboard'))
const ReportsPage = lazy(() => import('./features/reports/reports-page'))

function App() {
  return (
    <Router>
      <Suspense fallback={<PageSkeleton />}>
        <Routes>
          <Route path="/bookings" element={<BookingsPage />} />
          <Route path="/admin" element={
            <ProtectedRoute role="admin">
              <AdminDashboard />
            </ProtectedRoute>
          } />
          <Route path="/reports" element={<ReportsPage />} />
        </Routes>
      </Suspense>
    </Router>
  )
}

// Split heavy library usage
const BookingCalendar = lazy(() => 
  import('./features/calendar/booking-calendar').then(m => ({ default: m.BookingCalendar }))
)

// Prefetch on hover/focus (Next.js does this automatically for Link)
function NavigationItem({ to, label }: { to: string; label: string }) {
  const prefetchRoute = () => {
    // Dynamic import triggers browser to fetch the chunk
    import(`./pages/${to}`)
  }
  
  return (
    <Link to={to} onMouseEnter={prefetchRoute} onFocus={prefetchRoute}>
      {label}
    </Link>
  )
}
```

### Virtualization for Long Lists

```tsx
import { useVirtualizer } from '@tanstack/react-virtual'

function VirtualBookingList({ bookings }: { bookings: Booking[] }) {
  const containerRef = useRef<HTMLDivElement>(null)
  
  const virtualizer = useVirtualizer({
    count: bookings.length,
    getScrollElement: () => containerRef.current,
    estimateSize: () => 80,    // Estimated row height in px
    overscan: 5,               // Render 5 extra rows above/below viewport
  })
  
  return (
    <div ref={containerRef} style={{ height: '500px', overflow: 'auto' }}>
      <div style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: virtualRow.start,
              left: 0,
              right: 0,
              height: virtualRow.size,
            }}
          >
            <BookingCard booking={bookings[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## 4. TypeScript in React

```tsx
// 1. Strict prop typing (no 'any')
interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'destructive'
  size?: 'sm' | 'md' | 'lg'
  loading?: boolean
  leftIcon?: React.ReactNode
}

export function Button({ variant = 'primary', size = 'md', loading, leftIcon, children, disabled, ...props }: ButtonProps) {
  return (
    <button
      {...props}
      disabled={disabled || loading}
      className={cn(buttonVariants({ variant, size }))}
    >
      {loading ? <Spinner /> : leftIcon}
      {children}
    </button>
  )
}

// 2. Generic components
interface SelectProps<T> {
  options: T[]
  value: T | null
  onChange: (value: T) => void
  getLabel: (option: T) => string
  getValue: (option: T) => string
}

function Select<T>({ options, value, onChange, getLabel, getValue }: SelectProps<T>) {
  return (
    <select value={value ? getValue(value) : ''} onChange={e => {
      const selected = options.find(o => getValue(o) === e.target.value)
      if (selected) onChange(selected)
    }}>
      {options.map(opt => (
        <option key={getValue(opt)} value={getValue(opt)}>{getLabel(opt)}</option>
      ))}
    </select>
  )
}

// Usage: fully typed
<Select<Service>
  options={services}
  value={selectedService}
  onChange={setSelectedService}
  getLabel={s => s.name}
  getValue={s => s.id}
/>

// 3. Type-safe event handlers
function SearchInput({ onSearch }: { onSearch: (query: string) => void }) {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    onSearch(e.target.value)
  }
  
  return <input type="search" onChange={handleChange} />
}

// 4. Discriminated union for async state
type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error }

function useAsyncState<T>(promise: () => Promise<T>): AsyncState<T> {
  const [state, setState] = useState<AsyncState<T>>({ status: 'idle' })
  
  useEffect(() => {
    setState({ status: 'loading' })
    promise()
      .then(data => setState({ status: 'success', data }))
      .catch(error => setState({ status: 'error', error }))
  }, [])
  
  return state
}
```

---

## 5. Data Fetching Patterns

### TanStack Query

```typescript
// features/bookings/api/use-bookings.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'

// Query keys: deterministic, hierarchical
export const bookingKeys = {
  all:    ['bookings'] as const,
  lists:  () => [...bookingKeys.all, 'list'] as const,
  list:   (filters: BookingFilters) => [...bookingKeys.lists(), filters] as const,
  detail: (id: string) => [...bookingKeys.all, 'detail', id] as const,
}

// List query
export function useBookings(filters: BookingFilters = {}) {
  return useQuery({
    queryKey: bookingKeys.list(filters),
    queryFn: () => apiClient.get<PaginatedResponse<Booking>>('/bookings', { params: filters }),
    staleTime: 60_000,    // 1 minute before refetch
    select: (data) => data.data,  // Transform response
  })
}

// Detail query
export function useBooking(id: string | undefined) {
  return useQuery({
    queryKey: bookingKeys.detail(id!),
    queryFn: () => apiClient.get<{ data: Booking }>(`/bookings/${id}`).then(r => r.data),
    enabled: !!id,        // Don't fetch if ID is undefined
    staleTime: 30_000,
  })
}

// Create mutation with optimistic updates
export function useCreateBooking() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: (input: CreateBookingInput) =>
      apiClient.post<{ data: Booking }>('/bookings', input).then(r => r.data),
    
    onSuccess: (newBooking) => {
      // Optimistically add to list cache
      queryClient.invalidateQueries({ queryKey: bookingKeys.lists() })
      
      // Pre-populate detail cache
      queryClient.setQueryData(bookingKeys.detail(newBooking.id), newBooking)
    },
    
    onError: (err) => {
      toast.error('Erro ao criar agendamento: ' + getErrorMessage(err))
    }
  })
}

// Cancel mutation with optimistic update
export function useCancelBooking() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: (bookingId: string) =>
      apiClient.post(`/bookings/${bookingId}/cancel`),
    
    onMutate: async (bookingId) => {
      // Cancel any outgoing refetches
      await queryClient.cancelQueries({ queryKey: bookingKeys.detail(bookingId) })
      
      // Snapshot previous value
      const previous = queryClient.getQueryData(bookingKeys.detail(bookingId))
      
      // Optimistically update
      queryClient.setQueryData(bookingKeys.detail(bookingId), (old: Booking) => ({
        ...old,
        status: 'cancelled',
      }))
      
      return { previous }  // Return context for rollback
    },
    
    onError: (err, bookingId, context) => {
      // Rollback on error
      queryClient.setQueryData(bookingKeys.detail(bookingId), context?.previous)
    },
    
    onSettled: (_, __, bookingId) => {
      // Always refetch after mutation
      queryClient.invalidateQueries({ queryKey: bookingKeys.detail(bookingId) })
    },
  })
}
```

---

## 6. Testing React Applications

```tsx
// Testing Library: test behavior, not implementation

// Unit test for presentational component
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { BookingCard } from './booking-card'

describe('BookingCard', () => {
  const mockBooking: Booking = {
    id: 'booking_123',
    service: { id: 'service_1', name: 'Massagem Relaxante' },
    scheduledAt: '2025-07-01T10:00:00Z',
    status: 'pending',
  }
  
  it('shows booking details', () => {
    render(<BookingCard booking={mockBooking} />)
    
    expect(screen.getByText('Massagem Relaxante')).toBeInTheDocument()
    expect(screen.getByRole('status')).toHaveTextContent('pending')
  })
  
  it('calls onCancel when cancel button clicked', async () => {
    const user = userEvent.setup()
    const onCancel = jest.fn()
    
    render(<BookingCard booking={mockBooking} onCancel={onCancel} />)
    
    await user.click(screen.getByRole('button', { name: /cancel/i }))
    
    expect(onCancel).toHaveBeenCalledWith('booking_123')
  })
  
  it('does not show cancel button for completed bookings', () => {
    const completedBooking = { ...mockBooking, status: 'completed' as const }
    render(<BookingCard booking={completedBooking} />)
    
    expect(screen.queryByRole('button', { name: /cancel/i })).not.toBeInTheDocument()
  })
})

// Integration test with React Query
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { server } from '../../mocks/server'
import { http, HttpResponse } from 'msw'

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } }
  })
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  )
}

it('shows booking list from API', async () => {
  server.use(
    http.get('/api/v1/bookings', () =>
      HttpResponse.json({ data: [mockBooking] })
    )
  )
  
  renderWithProviders(<BookingList />)
  
  expect(screen.getByText(/loading/i)).toBeInTheDocument()
  expect(await screen.findByText('Massagem Relaxante')).toBeInTheDocument()
})
```

---

## 7. Accessibility

```tsx
// WCAG 2.1 AA compliance checklist and patterns

// 1. Semantic HTML first
<article>      {/* Not <div> for booking card */}
<nav>          {/* Not <div> for navigation */}
<main>         {/* Landmark region */}
<button>       {/* Not <div onClick> for clickable elements */}

// 2. ARIA only when semantic HTML is insufficient
<div role="status" aria-live="polite">{message}</div>  // Live region announcements
<button aria-label="Cancel booking for Massagem on July 1st">×</button>  // Icon buttons

// 3. Focus management for modals
function Modal({ isOpen, onClose, children }: ModalProps) {
  const closeRef = useRef<HTMLButtonElement>(null)
  
  useEffect(() => {
    if (isOpen) {
      closeRef.current?.focus()  // Move focus to modal when opened
      document.body.style.overflow = 'hidden'
    }
    return () => { document.body.style.overflow = '' }
  }, [isOpen])
  
  // Trap focus within modal
  function handleKeyDown(e: React.KeyboardEvent) {
    if (e.key === 'Escape') onClose()
    if (e.key === 'Tab') {
      // Focus trap implementation
    }
  }
  
  if (!isOpen) return null
  
  return createPortal(
    <div role="dialog" aria-modal="true" aria-labelledby="modal-title" onKeyDown={handleKeyDown}>
      <h2 id="modal-title">Confirmar Cancelamento</h2>
      {children}
      <button ref={closeRef} onClick={onClose} aria-label="Fechar">×</button>
    </div>,
    document.body
  )
}

// 4. Form error association
<div>
  <label htmlFor="email">E-mail</label>
  <input
    id="email"
    type="email"
    aria-describedby="email-error"  // Points to error element
    aria-invalid={!!errors.email}
  />
  {errors.email && (
    <span id="email-error" role="alert">{errors.email.message}</span>
  )}
</div>

// 5. Color contrast: minimum 4.5:1 for normal text, 3:1 for large text
// Use: https://webaim.org/resources/contrastchecker/
// Tailwind colors: bg-gray-800 text-white = 15.8:1 ✓
//                  bg-gray-100 text-gray-400 = 2.5:1 ✗ (too low)
```

---

## 8. Security in Frontend

```typescript
// 1. Never store JWTs in localStorage
// Use: sessionStorage, httpOnly cookies, or in-memory (Zustand)
// Why: localStorage is accessible by any script on the page (XSS risk)

// 2. Prevent XSS: never use dangerouslySetInnerHTML without sanitization
import DOMPurify from 'dompurify'

function RichTextDisplay({ html }: { html: string }) {
  return (
    <div dangerouslySetInnerHTML={{ 
      __html: DOMPurify.sanitize(html, {
        ALLOWED_TAGS: ['p', 'br', 'strong', 'em', 'a', 'ul', 'ol', 'li'],
        ALLOWED_ATTR: ['href', 'target', 'rel'],
        FORBID_SCRIPTS: true,
      })
    }} />
  )
}

// 3. CSRF protection for cookie-based auth
async function csrfFetch(url: string, options: RequestInit) {
  const csrfToken = document.cookie.match(/csrf-token=([^;]+)/)?.[1]
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'X-CSRF-Token': csrfToken ?? '',
    },
    credentials: 'include',
  })
}

// 4. Content Security Policy (enforced server-side, but inform frontend dev)
// Avoid inline scripts, styles
// Load resources only from trusted origins

// 5. Validate env vars are not sensitive
// NEXT_PUBLIC_ prefix → bundled into client code → visible to everyone
// NEVER expose: JWT secret, DB URL, API private keys via NEXT_PUBLIC_
const publicConfig = {
  apiUrl: process.env.NEXT_PUBLIC_API_URL,       // OK: public URL
  stripePublishableKey: process.env.NEXT_PUBLIC_STRIPE_KEY,  // OK: publishable key
  // NEVER: process.env.NEXT_PUBLIC_JWT_SECRET    // WRONG
}
```

---

## 9. Build Optimization

```javascript
// next.config.js — Next.js production optimization
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Bundle analyzer (run: ANALYZE=true npm run build)
  webpack: (config, { isServer }) => {
    if (process.env.ANALYZE === 'true') {
      const { BundleAnalyzerPlugin } = require('webpack-bundle-analyzer')
      config.plugins.push(new BundleAnalyzerPlugin({ analyzerMode: 'static' }))
    }
    return config
  },
  
  // Image optimization
  images: {
    formats: ['image/avif', 'image/webp'],
    domains: ['cdn.example.com'],
    deviceSizes: [640, 750, 828, 1080, 1200, 1920],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
  
  // Compression
  compress: true,
  
  // Remove console.log in production
  compiler: {
    removeConsole: process.env.NODE_ENV === 'production' ? { exclude: ['error'] } : false,
  },
  
  // Bundle splitting for better caching
  experimental: {
    optimizePackageImports: ['@radix-ui/react-icons', 'lucide-react'],
  },
}
```

---

## 10. Code Quality

```javascript
// .eslintrc.js — enforce quality rules
module.exports = {
  extends: [
    'next/core-web-vitals',
    'plugin:@typescript-eslint/recommended',
    'plugin:jsx-a11y/recommended',
    'plugin:react-hooks/recommended',
  ],
  rules: {
    // TypeScript
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/no-non-null-assertion': 'warn',
    '@typescript-eslint/consistent-type-imports': 'error',
    
    // React
    'react-hooks/rules-of-hooks': 'error',
    'react-hooks/exhaustive-deps': 'error',
    'no-restricted-globals': ['error', { name: 'event' }],
    
    // Security
    'no-eval': 'error',
    'no-implied-eval': 'error',
    
    // Accessibility
    'jsx-a11y/anchor-is-valid': 'error',
    'jsx-a11y/no-autofocus': 'warn',
  },
}
```

```json
// Core Web Vitals targets (measure with Lighthouse, web-vitals library)
{
  "LCP": "< 2.5s",    // Largest Contentful Paint (loading speed)
  "FID": "< 100ms",   // First Input Delay (interactivity)
  "CLS": "< 0.1",     // Cumulative Layout Shift (visual stability)
  "INP": "< 200ms",   // Interaction to Next Paint (responsiveness)
  "TTFB": "< 800ms"   // Time to First Byte (server response)
}
```
