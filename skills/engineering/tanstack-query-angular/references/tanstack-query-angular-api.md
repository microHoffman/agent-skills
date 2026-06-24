# TanStack Query Angular API Notes

These notes were drafted from Context7 results for TanStack Query Angular on 2026-06-24. Refresh against the official
TanStack docs before publishing or when exact syntax changes matter.

Primary docs:

- Angular quick start: https://tanstack.com/query/v5/docs/framework/angular/quick-start.md
- `injectQuery` reference: https://tanstack.com/query/v5/docs/framework/angular/reference/functions/injectQuery.md
- Query options guide: https://tanstack.com/query/v5/docs/framework/angular/guides/query-options.md
- TypeScript guide: https://tanstack.com/query/v5/docs/framework/angular/typescript.md
- Invalidations from mutations: https://tanstack.com/query/v5/docs/framework/angular/guides/invalidations-from-mutations.md
- Testing guide: https://tanstack.com/query/v5/docs/framework/angular/guides/testing.md

## Package And Imports

Angular Query v5 examples import from:

```ts
import {
  injectMutation,
  injectQuery,
  provideTanStackQuery,
  queryOptions,
  QueryClient,
} from '@tanstack/angular-query-experimental'
```

The package name includes `experimental`. Treat advanced or less common APIs as version-sensitive.

## Provider Setup

The testing guide shows `provideTanStackQuery(queryClient)` with a custom `QueryClient`:

```ts
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: false,
    },
  },
})

TestBed.configureTestingModule({
  providers: [provideTanStackQuery(queryClient)],
})
```

The guide says not to include `withDevtools` in tests and to call `queryClient.clear()` in `afterEach` to prevent data
leaking between tests.

## injectQuery

`injectQuery` injects a query into the current Angular scope. The docs describe the options callback as running in a
reactive context similar to Angular `computed`, so signals read inside the callback can drive query key changes and
`enabled` state.

```ts
filter = signal('')

todosQuery = injectQuery(() => ({
  queryKey: ['todos', this.filter()],
  queryFn: () => fetchTodos(this.filter()),
  enabled: !!this.filter(),
}))
```

The quick start template reads query state as callable signals:

```html
@for (todo of query.data(); track todo.title) {
  <li>{{ todo.title }}</li>
}
```

## queryOptions

The Angular docs recommend `queryOptions` for reusable, typed query configuration that can be passed to `injectQuery`,
`prefetchQuery`, `setQueryData`, and `getQueryData`.

```ts
@Injectable({ providedIn: 'root' })
export class QueriesService {
  private http = inject(HttpClient)

  post(postId: number) {
    return queryOptions({
      queryKey: ['post', postId],
      queryFn: () => {
        return lastValueFrom(
          this.http.get<Post>(`https://jsonplaceholder.typicode.com/posts/${postId}`),
        )
      },
    })
  }
}
```

The docs show these usage patterns:

```ts
postQuery = injectQuery(() => this.queries.post(this.postId()))
queryClient.prefetchQuery(this.queries.post(23))
queryClient.setQueryData(this.queries.post(42).queryKey, newPost)
```

They also show passing a computed signal that returns query options:

```ts
optionsSignal = computed(() => this.queries.post(this.postId()))
postQuery = injectQuery(this.optionsSignal)
```

## Mutations And Invalidation

The quick start and invalidation guides show `injectMutation` plus an injected `QueryClient`.

```ts
queryClient = inject(QueryClient)

mutation = injectMutation(() => ({
  mutationFn: addTodo,
  onSuccess: () => {
    this.queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
}))
```

Returning a promise from `onSuccess` keeps the mutation pending until the awaited invalidation work completes.

## Testing

Testing recommendations from the Angular guide:

- Build a test `QueryClient` with `retry: false`.
- Provide it with `provideTanStackQuery(queryClient)`.
- Do not include `withDevtools` in tests.
- Call `queryClient.clear()` in `afterEach`.
