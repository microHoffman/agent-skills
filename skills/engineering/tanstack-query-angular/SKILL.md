---
name: tanstack-query-angular
description: Use this skill when implementing, refactoring, reviewing, or debugging TanStack Query in Angular applications. Use it for Angular Query, TanStack Query Angular, @tanstack/angular-query-experimental, provideTanStackQuery, QueryClient, injectQuery, injectMutation, queryOptions, cache invalidation, optimistic updates, pagination, prefetching, testing, or migrations from manual HttpClient/RxJS server-state handling. Prefer this skill even when the user says "React Query", "TanStack Query", or "angular-query" in an Angular codebase.
---

# TanStack Query Angular

Use this skill to build Angular server-state features with TanStack Query v5's Angular adapter. The Angular adapter is
not React Query with different imports: it is signal-driven, uses Angular injection, and exposes query state through
callable signals such as `query.data()`, `query.isPending()`, and `query.error()`.

If exact API syntax matters, check the current official TanStack Angular Query docs before editing. The package is
currently `@tanstack/angular-query-experimental`, so avoid assuming stability across versions.

## First Pass

Before changing code:

1. Check `package.json` for `@tanstack/angular-query-experimental` and the installed major version.
2. Find the app's existing data-access conventions: `HttpClient`, GraphQL services, entity services, generated clients,
   `lastValueFrom`, error wrappers, interceptors, and test setup.
3. Use Angular Query for server state only: fetched data, cache synchronization, mutations, invalidation, prefetching,
   polling, pagination, and background refetching.
4. Keep UI-only state in Angular signals, component state, forms, or the existing local-state pattern.
5. Do not import from `@tanstack/react-query`, do not write React hooks, and do not use `useQuery`/`useMutation` in
   Angular code.

Read `references/tanstack-query-angular-api.md` when you need a quick API refresher or source links.

## App Setup

Provide one `QueryClient` at application bootstrap or in the closest app-level provider configuration.

```ts
import { ApplicationConfig } from '@angular/core'
import { provideHttpClient } from '@angular/common/http'
import { QueryClient, provideTanStackQuery } from '@tanstack/angular-query-experimental'

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 60_000,
      gcTime: 5 * 60_000,
      retry: 2,
      refetchOnWindowFocus: true,
      refetchOnReconnect: true,
    },
  },
})

export const appConfig: ApplicationConfig = {
  providers: [
    provideHttpClient(),
    provideTanStackQuery(queryClient),
  ],
}
```

Choose defaults based on product behavior:

- Use a short `staleTime` for volatile operational data.
- Use a longer `staleTime` for reference data, permissions, dictionaries, or rarely changing metadata.
- Keep `gcTime` at least as long as `staleTime`; otherwise fresh data can be garbage-collected.
- Disable retries or lower retry counts for mutations and tests.

## Query Shape

Prefer feature query services that return typed `queryOptions`. This keeps query keys, functions, and defaults reusable
from components, prefetching, invalidation, and tests.

```ts
import { HttpClient } from '@angular/common/http'
import { Injectable, inject } from '@angular/core'
import { queryOptions } from '@tanstack/angular-query-experimental'
import { lastValueFrom } from 'rxjs'

import { Product, ProductFilters } from './product.model'

@Injectable({ providedIn: 'root' })
export class ProductQueries {
  private readonly http = inject(HttpClient)

  list(filters: ProductFilters) {
    return queryOptions({
      queryKey: ['products', 'list', filters] as const,
      queryFn: () => lastValueFrom(this.http.get<Product[]>('/api/products', { params: { ...filters } })),
      staleTime: 60_000,
    })
  }

  detail(productId: string) {
    return queryOptions({
      queryKey: ['products', 'detail', productId] as const,
      queryFn: () => lastValueFrom(this.http.get<Product>(`/api/products/${productId}`)),
      enabled: !!productId,
    })
  }
}
```

Use `injectQuery` with a callback so Angular signals read inside the callback become query dependencies.

```ts
import { Component, computed, inject, signal } from '@angular/core'
import { injectQuery } from '@tanstack/angular-query-experimental'

@Component({
  selector: 'app-products',
  template: `
    @if (productsQuery.isPending()) {
      <app-spinner />
    } @else if (productsQuery.isError()) {
      <app-error [message]="productsQuery.error()?.message" />
    } @else {
      <app-product-list [products]="productsQuery.data() ?? []" />
    }
  `,
})
export class ProductsComponent {
  private readonly queries = inject(ProductQueries)

  readonly search = signal('')
  readonly filters = computed(() => ({ search: this.search().trim() }))

  readonly productsQuery = injectQuery(() => this.queries.list(this.filters()))
}
```

Query guidance:

- Query keys should be serializable, stable, and hierarchical: `['orders', 'list', filters]`,
  `['orders', 'detail', orderId]`.
- Include every server-state dependency in the query key.
- Use `enabled` for dependent queries instead of conditionally calling `injectQuery`.
- Use `select` for cheap derived server data when it avoids repeated component transformation.
- If using `fetch`, accept `signal` from the query function context and pass it to `fetch`.
- If using Angular `HttpClient`, follow the app's existing promise/observable boundary. `lastValueFrom` is common for
  query functions.

## Mutations

Use `injectMutation` for writes and invalidate or update affected queries. Inject `QueryClient` from Angular DI.

```ts
import { Component, inject } from '@angular/core'
import { injectMutation, QueryClient } from '@tanstack/angular-query-experimental'

@Component({
  selector: 'app-product-editor',
  template: `
    <button type="button" [disabled]="saveMutation.isPending()" (click)="save()">
      Save
    </button>
  `,
})
export class ProductEditorComponent {
  private readonly api = inject(ProductApi)
  private readonly queryClient = inject(QueryClient)

  readonly saveMutation = injectMutation(() => ({
    mutationFn: (payload: UpdateProductPayload) => this.api.updateProduct(payload),
    onSuccess: product => {
      this.queryClient.setQueryData(['products', 'detail', product.id], product)
      return this.queryClient.invalidateQueries({ queryKey: ['products', 'list'] })
    },
  }))

  save() {
    this.saveMutation.mutate(this.buildPayload())
  }
}
```

For optimistic updates:

```ts
readonly saveMutation = injectMutation(() => ({
  mutationFn: (payload: UpdateProductPayload) => this.api.updateProduct(payload),
  onMutate: async payload => {
    const detailKey = ['products', 'detail', payload.id] as const

    await this.queryClient.cancelQueries({ queryKey: detailKey })
    const previousProduct = this.queryClient.getQueryData<Product>(detailKey)

    this.queryClient.setQueryData<Product>(detailKey, old => old ? { ...old, ...payload } : old)

    return { detailKey, previousProduct }
  },
  onError: (_error, _payload, context) => {
    if (context?.previousProduct) {
      this.queryClient.setQueryData(context.detailKey, context.previousProduct)
    }
  },
  onSettled: (_data, _error, payload) => {
    return Promise.all([
      this.queryClient.invalidateQueries({ queryKey: ['products', 'detail', payload.id] }),
      this.queryClient.invalidateQueries({ queryKey: ['products', 'list'] }),
    ])
  },
}))
```

Mutation guidance:

- Return promises from `onSuccess`/`onSettled` when the mutation should remain pending until invalidation finishes.
- Cancel matching queries before optimistic writes to avoid races.
- Snapshot previous cache values and roll them back in `onError`.
- Invalidate list queries after create/update/delete unless you can prove all affected cache entries were updated.
- Prefer specific invalidation keys over global invalidation.

## Pagination And Lists

For page-based lists, put page, sort, and filters into the key and use `placeholderData` to keep the previous page visible
while the next page loads.

```ts
readonly page = signal(1)

readonly productsQuery = injectQuery(() =>
  queryOptions({
    queryKey: ['products', 'list', { page: this.page(), search: this.search() }] as const,
    queryFn: () => this.api.getProducts({ page: this.page(), search: this.search() }),
    placeholderData: previousData => previousData,
  }),
)
```

For infinite queries, verify the current Angular adapter API before coding. TanStack Query v5 requires an
`initialPageParam` for infinite query options.

## Error And Loading UX

Represent states explicitly in templates:

- `isPending()` or `isLoading()` for the first load.
- `isFetching()` for background refresh indicators.
- `isError()` plus `error()` for failure state.
- Existing cached `data()` can coexist with a background fetch; avoid blanking the UI during refetch unless the product
  requires it.

Keep query functions pure. They should fetch and normalize data, not show toasts, navigate, or mutate unrelated state.
Put user feedback in mutation callbacks or component effects where the app already handles it.

## Testing

Use a fresh `QueryClient` per test module or per spec and clear it between tests.

```ts
import { TestBed } from '@angular/core/testing'
import { QueryClient, provideTanStackQuery } from '@tanstack/angular-query-experimental'

let queryClient: QueryClient

beforeEach(() => {
  queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        retry: false,
        gcTime: Infinity,
      },
    },
  })

  TestBed.configureTestingModule({
    providers: [
      provideTanStackQuery(queryClient),
    ],
  })
})

afterEach(() => {
  queryClient.clear()
})
```

Testing guidance:

- Do not include Angular Query devtools in test providers; it slows tests and adds noise.
- Set `retry: false` so failures surface immediately.
- Seed cache with `queryClient.setQueryData` for component rendering tests.
- Use the app's normal HTTP testing utilities for query functions backed by `HttpClient`.
- Avoid sharing one `QueryClient` across unrelated specs.

## Angular-Specific Pitfalls

- Importing React Query APIs in Angular code.
- Treating query result fields as plain properties instead of callable signals in templates and components.
- Reading signals outside the `injectQuery` callback and expecting automatic query-key reactivity.
- Omitting reactive dependencies from query keys.
- Wrapping query cache state in `BehaviorSubject` or duplicating it into a service store without a product reason.
- Using `initialData` when the goal is to keep previous page data; prefer `placeholderData`.
- Invalidating too broadly after every mutation, causing unnecessary refetches and flicker.
- Forgetting to clear the query cache in tests.

## Output Standard

When using this skill, produce Angular code and guidance that includes:

1. Correct Angular Query imports from `@tanstack/angular-query-experimental`.
2. A clear query-key strategy.
3. Signal-aware `injectQuery` usage.
4. Mutation invalidation or cache-update behavior.
5. Test setup changes when the code adds or changes query behavior.
6. A short note if an API detail should be rechecked against the current TanStack docs because the Angular adapter is
   still experimental.
