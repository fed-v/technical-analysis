# Technical Report for Subscription Manager V2

The Subscription Manager is a Nuxt 3 application built to support internal users in configuring and provisioning subscription-based service plans. It enables the dynamic assembly of recurring and one-time components across multiple steps, with real-time validation and pricing logic driven by backend APIs. The app is structured around a modular component architecture with clear separation between container and presentational layers, ensuring maintainability and scalability. Key technical features include composables for shared logic, API integration for account lifecycle management, and a centralized state-driven form workflow that supports conditional rendering, user input validation, and step transitions. Designed for telecom or utility-style use cases, the system abstracts complex billing logic behind a user-friendly interface while maintaining a clean codebase suitable for extension and testing.

## Tech Stack

This project uses a modern, modular frontend stack built on Nuxt 3 and integrates a variety of tools and services to support internationalization, state management, authentication, testing, and UI design.

### Framework and Core Libraries
- Nuxt 3: Main frontend framework.
- Nitro.js: Middleware for route guarding.
- TypeScript: Full TypeScript support for type safety with defined type interfaces.

## Authentication
- AWS Cognito: Authentication handled via AWS Amplify v4 (aws-amplify and @aws-amplify/ui-vue) for user auth and session management.
- @sidebase/nuxt-auth: Server-side session-based auth integration.
- Multiple Cognito user pools supported via runtimeConfig for Customers and CSPs.

## Internationalization (i18n)
- @nuxtjs/i18n:
  - Supported languages: English (en), Spanish (es), Portuguese (pt).
  - Lazy loading JSON translations from the /lang directory.
  - no_prefix strategy for cleaner routes.

## State Management
- Pinia: Modern state management library for Vue 3.
- @pinia-plugin-persistedstate/nuxt: Automatically persists Pinia stores to local storage.

## Form Validation
- VeeValidate : Reactive form validation with custom error handling.

## UI & Styling
- PrimeVue: for a rich component UI suite, with tree-shaken imports.
- Components included: AutoComplete,Toast, Tab Navigation, DataTable, MultiSelect Dropdown Input.
- LESS preprocessor for advanced CSS tooling. (This should be replaced for SASS)
- Ionicons: Loaded from CDN for icon support.

## Editor & Content
- Monaco Editor: Embedded code editor via nuxt-monaco-editor.
- @nuxt/content: Markdown and content management inside Nuxt.
- @sidebase/nuxt-pdf: Server-side PDF rendering.
- Handlebars + Juice: Used for rendering and inlining HTML emails.

## Testing & Linting
- Vitest: Unit testing framework.
- Cypress: End-to-end (E2E) testing (cy:open, cy:run).
- ESLint for static code analysis with Vue plugin and Prettier integration.
- Git hook automation via Husky and lint-staged for linting/formatting before commits.
- Uses @nuxt/eslint for Nuxt-specific linting rules.

## Developer Experience
- @nuxt/devtools: Nuxt DevTools for debugging and inspecting the app during development.
- Runtime Environment Configs: Dev, Production, and Test-specific settings defined in nuxt.config.ts.

## Deployment & Platform Support
- Nitro with Cloudflare:
  - Compatible with Cloudflare Workers using nodeCompat: true.
  - Includes compatibilityDate and deployConfig.

## Third-Party Modules
- @tucowsinc/wavelo-nuxt-module: Internal module for Wavelo platform components and branding. Version installed: 0.0.22

### Stripe Integration
- @stripe/stripe-js + vue-stripe-js for frontend payment handling.

## Frontend-Backend Coupling

The current frontend is moderately coupled with the existing Platypus backend. The `api-url-helper.ts` centralizes URL construction, and `use-fetch-helper.ts` abstracts the network calls and authentication.

However, moving to a completely different backend would primarily involve rewriting the getUrl logic in `api-url-helper.ts` to match the new backend's routing and parameter conventions, and more significantly, implementing a robust data transformation layer (either implicitly in our components or explicitly through adapters) to handle potentially different request and response data structures. This is where most of the effort would lie.

## Analysis of Coupling

### 1. api-url-helper.ts (URL Builder):

#### Positive Aspects (Looser Coupling):
- Abstracted Base URLs: The `buildAPIMap` function dynamically constructs base URLs based on `useProxy` or runtime configuration. It means we can easily switch between proxying requests through Nuxt's own server (for development/SSR benefits) or directly calling different backend services by just changing configuration.
- Centralized Endpoint Definitions: All the API endpoints are defined in a single `getUrl` function using a switch statement. This provides a single source of truth for our API routes.
- Parameterization: The `id`, `secondaryId`, `limit`, and `sort` parameters allow for flexible URL construction, meaning we're not hardcoding specific IDs into each call site. Changing how we construct “accounts” queries (e.g. changing the name of the limit or offset query parameters) only impacts one case in the switch.

#### Areas of Tighter Coupling:
- Endpoint String Literals: The switch statement uses string literals like 'account', 'accounts', 'markets', etc. These strings are directly tied to the backend's expected routes. If the backend changes an endpoint name (e.g., from /account to /user), we'd need to update this file.
- Specific Query Parameters/Paths: Many of the case statements define specific query parameters (e.g., `?query=${id}&limit=${limit || 10}&sort=...`) and path structures (e.g., /account/${id}/address). These directly reflect the backend's API design.
- Implicit Contract: While the URLs are constructed, the meaning of `id`, `secondaryId`, `limit`, and `sort` for each endpoint is implicitly understood between frontend and backend. Any backend change is “invisible” until we manually run the code and see broken pages or HTTP-404s.

### 2. use-fetch-helper.ts (Fetcher):

#### Positive Aspects (Looser Coupling):
- Generic HTTP Methods: The `fetcher` function accepts a generic HttpMethod ('get', 'post', 'put', 'delete'). This is standard and highly flexible. If the new backend still uses standard REST verbs (or something like that), we won't need to rewrite calling code for each HTTP method.
- Centralized Authorization: Fetching the Authorization token from the Pinia store and attaching it to all requests is a good practice for consistent authentication and is decoupled from individual component logic. If the new backend uses Bearer tokens in the Authorization header, we need zero changes here.
- Error Handling: Centralized `onResponseError` and `onRequestError` callbacks for toast notifications are decoupled from the specific API calls. Only if the new backend’s error format is drastically different (e.g. nested under error.details) would we need to rewrite how we parse error objects in these hooks.
- File Upload Handling: Detecting a File and switching to FormData is already built in. If the new service expects multipart form uploads the same way, we need to don’t touch this logic.

#### Areas of Tighter Coupling:
- Implicit Request/Response Structure: While the fetcher handles the request mechanics, the frontend components using it assume a certain structure for the body they send and the response they receive. For instance, if the backend expects camelCase and returns snake_case, our frontend components are handling that implicitly. Likewise, code that reads response.value.data.accounts might break if the new backend nests data differently (response.value.results.data), so components implicitly expect a certain shape.
- Direct body Passing: The body parameter is passed directly. If a new backend requires a different nested structure for certain requests, we'd have to modify the data payload before calling `fetcher` for that specific endpoint.
- Tied to Pinia Store’s Token Getter: Every request grabs `appStore.getToken`. If the new backend uses a different auth mechanism (e.g. session cookie or a custom header key), we’llneed to update this composable—and maybe the store itself—but components don’t need to know these details.

### Difficulty of Decoupling and Integrating a New Backend:

Based on the current setup, it would be moderately difficult, but manageable, to integrate with a completely different backend. Here's why and what it would entail:

#### What would be relatively easy:
- Switching Base URLs: This is the easiest part. We could simply update our `useRuntimeConfig()` public variables (e.g., `billingURL`, `customerURL`) to point to the new backend's base URLs. If we're still using the proxy, we'd update the proxy configuration in nuxt.config.ts.
- Authentication Mechanism: If the new backend uses a standard Bearer token for authentication, our fetcher will likely work as-is for sending the token. If it uses something else (e.g., cookies, different header format), we’ll have to modify the headers part of `fetcher`.
- Basic HTTP Methods: The core GET, POST, PUT, DELETE operations could remain the same.

#### What would require significant refactoring:
- api-url-helper.ts (`getUrl` function):
  - Complete Overhaul of switch statement: This is the biggest point of friction. If the new backend has entirely different endpoint naming conventions, different URL structures, or expects parameters in different formats (e.g., path parameters vs. query parameters), we would need to rewrite almost every case statement within `getUrl`.
  - Parameter Mapping: The `id`, `secondaryId`, `limit`, `sort`, query parameters might need to be re-evaluated and potentially remapped for each specific endpoint in the new backend.
  - New API Segmentation: If the new backend combines services that were previously split (e.g., CUSTOMER and BILLING become one), or splits them differently, our APIMap and how we use API.CUSTOMER, API.BILLING, etc., would need adjustment.
  - Potentially Large Effort: If the new backend uses a radically different protocol (GraphQL, gRPC, SOAP), we may need to scrap `api-url-helper.ts` altogether, replacing it with a generated client or a completely new composable.

#### Request Body and Response Data Structures:
- Data Transformation Layer: This is crucial. Even if the URLs are updated, the shape of the data we send (body) and the shape of the data we receive from the new backend (response) will almost certainly be different.
- Outgoing Data: Our frontend components currently prepare data based on the old backend's expectations. We'd need to create mapping functions or transformers before calling `fetcher` for each relevant endpoint. For example, if our old backend expects { firstName: string } and the new one expects { first_name: string }, we'd need a transformation.
- Incoming Data: Similarly, the data received from the new backend will likely need to be transformed into the format our existing Nuxt components expect. This means modifying the code that uses the fetcher and processes the response. We might need to implement a response interceptor or transform the data directly after the fetcher call.
- Error Handling: While fetcher has generic error toasts, the structure of backend error messages (e.g., `response.data.message` vs. `response.data.error.details`) could likely change, requiring updates to how we parse and display specific error messages.

#### Frontend Components (indirect impact):
- Although the composables encapsulate the API calls, the components that use these composables are implicitly coupled to the data structures they expect from the backend. If the backend returns different fields or nested objects, these components would break or display incorrect information. This means a cascade of changes through the component tree as we adapt to the new data shapes. All existing components that depend on specific data shapes will need to be verified.

## Code Quality Evaluation

Overall, the codebase demonstrates clear, consistent naming and generally sound TypeScript usage, with well-defined interfaces and thoughtfully organized getters, actions, and composables. Templates and components follow predictable conventions for props, computed values, and event handling, making them easy to navigate. Formatting is mostly uniform, though there are a few minor inconsistencies in quote style and semicolon usage. Error handling is present in utility functions but is often implicit—components and stores rarely guard against failing fetches or invalid inputs. Readability is strong overall: functions have descriptive names, and the store’s purpose is clearly documented, but a few inline expressions could be refactored into named helpers for clarity. 

### 1. Naming Conventions

Across the application, naming conventions are generally clear and self-descriptive:

- Store Getters and Actions: In the Pinia store, methods like `getAccountId`, `getStripeCustomerID`, and `setCustomerName` immediately convey their purpose. The state properties (`id`, `email, addresses`, `guarantorsList`) match domain concepts precisely.
- Component Props and Functions: In the Payments‐row component, props are defined as `{ id: string; onClick: () => void; data: PaymentsData }`, so it’s evident that `data` holds a `PaymentsData` object, and `onClick` is the refund handler. Helper functions like `getTransactions()` and `getTotal()` read naturally—one returns a joined transaction list, the other computes a total amount.
- Composables: In the Nuxt page, composables such as `useI18n` and `useAppStore` follow standard Vue/Nuxt patterns. Computed hooks named `initHistory` and `initCards` clearly indicate initialization logic for page data.
- `HistoryData` and `HomeCardProps` types are imported from `types/interfaces`, which implies a centralized, descriptive taxonomy of the domain.

Overall, identifiers across state, methods, props, and templates are concise and meaningful, reflecting each item’s role without ambiguity.

### 2. Code Consistency

Overall, each file adheres to a coherent style guide, with consistent indentation, brace placement, and logical grouping of code sections.

- Indentation & Spacing: All files use consistent two‐space indentation. Imports are grouped at the top, followed by component or store definitions. Blank lines are used to separate logical sections (e.g., between `state, getters`, and `actions` in the Pinia store), aiding readability.
- Curly Braces & Parentheses: Arrow functions and object literals open and close braces predictably. In the Nuxt page and component scripts, multi‐line objects (such as prop definitions and computed return values) maintain aligned braces and indentation.
- Quotes & Semicolons: While there are minor variations (some imports use single quotes, others double), each file consistently applies its chosen style within itself. Semicolons appear reliably at the end of statements in the store and test, and object/array literals follow a stable pattern of commas and line breaks.
- Template Structure: In all Vue files, template tags are properly indented, with each HTML element on its own line. Attributes and directives (e.g. `v-for, :key, :card`) follow the same spacing conventions throughout.
- Consistent Grouping: In the Vitest test, all `vi.stubGlobal` calls are kept together in `beforeEach`, and the it block begins immediately afterward. In the store, getters appear before actions, and state is declared at the top—mirroring typical Pinia conventions.

### 3. Use of TypesScript (Types / Interfaces)

Across the four files, TypeScript usage is deliberate and structured:

- Pinia Store:
  - Defines an `AppState` interface listing each state property (`id: string`, `addresses: AddressData[]`, etc.).
  - Getters and actions are explicitly typed (e.g. `getAccountId(): string, setNote(note: NotesData): void`).
  - The `useAccountStore` factory relies on these interfaces to ensure state mutations and accesses align with the defined shapes.
- Nuxt Page (pages/index.vue):
  - Imports `HistoryData` and `HomeCardProps` for reactive variables (`history` and `cards`) so that `ComputedRef<HistoryData[]>` and `ComputedRef<HomeCardProps[]>` are clear.
  - The `fetcher(...) as { data: Ref<HistoryData[]> }` cast guarantees at compile time that the fetched data matches `HistoryData`.
- Payments‐Row Component:
  - Uses `defineProps<{ id: string; onClick: () => void; data: PaymentsData }>()` so that `props.data` is known to be a fully typed `PaymentsData` object.
  - Computed properties (`transactionTypes, paymentMethodTypes`) infer their types from helper functions, ensuring that calls like `getTransactionType(t)[transaction_type]` produce a string.
- Vitest Test:
  - Though focused on runtime stubbing, the test declares `const MOCK_STRING: string` to satisfy the component’s prop types.
  - When stubbing `useFetch`, the returned object mimics the shape `{ data: { value: Partial<PaymentsData> } }` so TypeScript will flag any missing or mismatched fields.

Overall, each file leverages TypeScript interfaces and type annotations—on state, props, computed values, and mocked data—to enforce correct shapes and reduce runtime errors.

### 4. Error Handling

Across the application files, error handling is present but largely implicit:

- Pinia Store
  - Contains no direct network calls, so it does not include error‐handling logic. Any fetch errors would need to be caught by the components or composables that populate state before calling store actions.
- Nuxt 3 Index Page
  - Invokes `fetcher(...)` but does not check for or react to errors on the resulting `useFetch`. If the request fails, `history.value` remains undefined and the computed filter returns an empty array, effectively hiding the error without notification.
- Payments‐Row Component
  - Does not perform its own fetch; it receives fully formed `PaymentsData`. As written, it assumes valid data and could produce runtime issues if, for example, `data.payment_information` is missing or malformed. There are no try/catch guards around utility calls like `displayDate` or `formatPrice`.
- Vitest Unit Test
  - Stubs `useFetch` to return a successful payload; it does not test error cases. There is no assertion or setup for handling fetch failures, so the test suite does not exercise any error‐handling branches.

In summary, error handling exists at the fetcher/composable level (showing generic toasts), but individual pages and components do not explicitly guard against fetch errors or invalid data shapes. The unit test similarly focuses on a successful fetch scenario and does not validate how the component behaves under error conditions. This is something that needs further work, especially in the Customer Creation Flow.

### 5. Readability and Comments

Throughout the application, readability is generally strong and comments are used sparingly but effectively:

- Pinia Store
  - The top-of-file comment clearly explains the store’s purpose (holding selected account data, not the authenticated user).
  - State properties, getters, and actions are grouped logically, with blank lines separating each section.
  - Method and property names (e.g., `getStripeCustomerID`, `resetAll`) convey intent, reducing the need for many inline comments.
  - There are no excessive comments cluttering the code—only a few guiding notes, keeping the implementation itself easy to follow.
- Nuxt 3 Index Page
  - The template is well-indented, with each wrapper `<div>` and `<HomeCard>` on its own line, making the hierarchy clear.
  - Script logic (e.g., `initHistory`, `initCardsv`) is broken into small, named functions; the single inline comment in initHistory explains the filtering of unique resources.
  - Variable names like `limitedHistory` and `cards` reflect their role in the template, and there are no large blocks of uncommented, dense logic—each computed is concise.
- Payments-Row Component
  - The template’s <td> cells and binding expressions are straightforward and consistently spaced.
  - Helper functions (`getTransactions`, `getPaymentTypes`, `getTotal`) are each only a few lines long, making the data-processing steps easy to trace.
  - Although there are no comments inside these functions, their descriptive names and the clear parameter (props.data) minimize ambiguity.
  - Import statements for utilities like `displayDate` and `formatPrice` underscore where formatting logic lives, aiding navigation.
- Vitest Unit Test
  - The `beforeEach` block groups all global stubs together; while no inline comments explain each stub, the stub names (e.g., "useRoute", "useI18n") make their purpose evident.
  - The `it.skip`("should open a modal") test is concise, with each assertion on its own line and a clear sequence: mount → find button → click → assert modal state.
  - Overall, the test reads like a step-by-step scenario without needing extensive comments.

In summary, the application favors clear naming, small focused functions, and logically grouped sections. Comments are used only when helpful (e.g., explaining filter logic or the store’s role), and otherwise the code’s structure and identifiers guide the reader through each file.

### 6. Test Coverage
The test suite is organized into clearly separated folders—components, composables, pages, and stores—so every part of the app has its own dedicated spec file.

- Under pages/account, each section (History, Invoices, Notes, Payments, Related Accounts, Subscription) has its own test file, and there are also top‐level page tests for things like `admin-roles`, `customer-account`, `payment`, and `profile`.
- In components, files like `BaseInput.test.ts`, `DynamicBreadcrumbs.test.ts`, and `ReportTable.test.ts` ensure that individual UI pieces render and behave correctly.
- The composables folder contains tests for apiUrlHelper and other helper methods, verifying that URL construction and utility functions work as expected. Although the `payment.test.ts` spec is currently skipped, it demonstrates how `useFetch` is stubbed to simulate data and how UI interactions (opening a refund modal) are checked.
- Tests in pages/account/ are currently incomplete and still need to be worked on.

Overall, most core modules—pages, components, and composables—have at least one corresponding test file, giving broad coverage of rendering and basic functionality across the codebase. Some tests still need to be added or completed.

## Component Modularity

The application’s UI is built from a clear hierarchy of modular components, ranging from small, reusable “base” elements (like buttons, menus, and form wrappers) to larger, page‐specific containers that assemble those pieces into complete screens. Presentational components (e.g., `<BaseMenu>` or the one‐step form wrapper) focus solely on layout and styling, exposing flexible props and slots for maximum reuse. Page‐level components (such as the payment details page) compose these building blocks while also handling data‐fetching and business logic, but still delegate most UI concerns to child components. This organization ensures that individual units follow single‐responsibility principles—pure UI elements remain independent of domain logic, and larger containers orchestrate data and presentation without entangling internal rendering details.

Some examples of components exhibit a clear separation of concerns and strong reuse potential:
- BaseMenu.vue is a reusable presentational component: it accepts a generic `items` array and optional `componentStyle` props, and exposes a default slot inside the `<Menu>` wrapper. Its sole responsibility is rendering a toggle button and dropdown, with no business logic mixed in.
- [payment].vue is a page‐level container that composes smaller building blocks (e.g., `<DynamicBreadCrumbs>`, `<BaseMenu>`, and `<SeparatorComponent>`) while also handling data fetching, state management (via `fetchPaymentDetails` and `fetchPaidInvoices`), and computed totals. Although it combines layout and domain logic, it delegates most rendering to child components, keeping each unit focused on its own concern.
- The one‐step Form wrapper serves purely as a layout shell: it renders a header and footer around a `<slot />` where actual form fields live. By accepting `title`, `cancelLink`, `onSubmit`, and `initialValues` as props, it remains agnostic of specific form content and can be reused for any single‐step form across the app.
- UI Kit Directory: The components/ui-kit folder houses a collection of reusable base components (e.g., `BaseButton.vue`, `BaseInput.vue`, `BaseCard.vue`, `BaseModal.vue`, etc.), providing a centralized library of generic UI building blocks that can be used consistently across different pages and features.
- Compound Multi‐Step Forms: Multi‐step forms leverage a compound‐component pattern—each step lives in its own .vue file (e.g., `StepOne.vue`, `StepTwo.vue`), and the parent <Form> dynamically renders the current step via `<component :is="currentStepComponent" />`. Shared pieces like the header (`<LazyFormsMultistepHeader>`) and footer (`<LazyFormsMultistepFooter>`) wrap around the slot, ensuring each step component remains focused on its own UI and logic.
- Helper Composable of Utility Methods: A centralized composable exposes reusable functions—such as `toggleBoolean`, `sumCharges`, `prevStep`, `capitalizeWord`, `formatPhoneNumber`, `extractUrlTail`, and HTML sanitizers (`removeScriptsTags`, `sanitizeHTML`)—to perform common tasks (string/number formatting, step navigation, HTML cleaning) across multiple components.

In each case, props and slots are leveraged thoughtfully—presentational components expose only what they need, and container components aggregate those pieces into pages without bleeding layout details into business logic. Common functions are organized into a dedicated helper composable, providing reusable utilities for tasks.

## Areas to Improve

### Strengthen Error Handling
- Add explicit try/catch or reactive error checks around every `useFetch` call (e.g. in `initHistory`, `fetchPaymentDetails`, `fetchPaidInvoices`) so that individual components can surface user‐friendly error messages instead of silently failing or returning empty data.
- Guard utility functions (e.g. `displayDate`, `formatPrice`, sanitizers) with try/catch or null checks to prevent runtime crashes if inputs are malformed.

### Finish & Expand Unit Tests
- Un‐skip and complete existing payment.test.ts to exercise different data scenarios (successful fetch, empty arrays, fetch‐error paths).
- Add “error‐case” tests for composables (e.g. `apiUrlHelper.test.ts`, `helperMethods.test.ts`) to confirm they throw or return defaults when given invalid inputs.
- Ensure each page and “Section” component (HistorySection, InvoicesSection, etc.) has at least one test for rendering, one for empty data, and one for error handling.

### Extract Business Logic into Composables
- In page‐level components (e.g. `[payment].vue`, multi‐step form pages), move data‐fetching and computation (like `fetchPaymentDetails`, `fetchPaidInvoices`, and `formatProductToSubscription`) into dedicated composables (e.g. `usePaymentDetails`, `useSubscriptionFlow`). This keeps the <template> and layout code focused on presentation.

### Clean Up CSS & Migrate from LESS to SASS
- Consolidate scattered .less imports (e.g. ~/assets/styles/less/forms.less, multistep.less) into a single SASS partial structure, taking advantage of variables, mixins, and nesting for easier maintenance.
- Remove deprecated or unused CSS classes (e.g. any leftover “test” class in `index.vue`).
- Use scoped styles only where necessary and extract global utility classes (e.g. spacing, typography) into a shared SASS file referenced in nuxt.config.ts.

### Update Outdated Version Wavelo Nuxt Module
- Update @tucowsinc/wavelo-nuxt-module; to the latest version and replace the base components that are already in the module.
