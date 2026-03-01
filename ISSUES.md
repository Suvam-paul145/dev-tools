# Proposed Issues for BetterBugs Dev-Tools

> **Repository:** [betterbugs/dev-tools](https://github.com/betterbugs/dev-tools)
> A comprehensive suite of 175+ free developer utility tools built with Next.js 14, React 18, TypeScript, and Tailwind CSS.

---

## Hard-Level Issues

---

### Issue #H1: Implement a Comprehensive Unit and Integration Testing Framework Across All 175+ Tool Components

**Problem Description**

The entire codebase currently has zero test files — no unit tests, no integration tests, and no end-to-end tests. There is no test runner (such as Jest, Vitest, or Playwright) configured in the project. Every tool component inside `app/components/developmentToolsComponent/` (over 174 files) performs real-time data transformations — encoding, decoding, formatting, validating, converting — yet none of these critical logic paths are validated by automated tests. A single regression in any transformation function (for example, the Base64 encoder producing incorrect output, or the JSON validator silently accepting invalid JSON) would go completely undetected until a user encounters the bug in production.

**Why It Is Required**

Without tests, contributors cannot confidently refactor code, update dependencies, or add new features without risking silent breakage of existing tools. The project has over 15 forks and active community contributions, which increases the likelihood of regressions. Testing is a fundamental requirement for any production-grade open-source project and is essential for long-term maintainability. Additionally, tools like the bcrypt generator (`bcryptGenerator.tsx`), the credit card validator (`creditCardValidatorComponent.tsx`), and the JSON validator (`jsonValidator.tsx`) deal with sensitive or precision-critical logic where incorrect output could mislead developers.

**My Approach**

I would set up Jest along with React Testing Library as the testing framework (aligned with the Next.js ecosystem). I would create a test configuration file (`jest.config.ts`) and add the required dev dependencies. Then, I would write unit tests for at least 30–40 of the most critical tool components, focusing first on transformation logic (encoders, decoders, converters, validators, formatters). Each test file would live alongside its component (e.g., `base64Encoder.test.tsx`). I would also add integration tests for the dynamic routing layer in `app/[slug]/page.tsx` to verify that tools render correctly based on slug. Finally, I would add a `test` script to `package.json` and integrate the test run into the existing GitHub Actions release workflow (`release.yml`) so that tests are executed on every push.

**I want to work on this issue.**

---

### Issue #H2: Replace the Monolithic Constants File with an Auto-Discovery Plugin System for Tool Registration

**Problem Description**

Currently, every tool must be manually registered in two separate files: `app/libs/constants.tsx` (which imports all 174+ component files and maps them in a single `PATHS` object) and `app/libs/developmentToolsConstant.tsx` (which stores metadata, SEO data, descriptions, and route information for each tool). The `constants.tsx` file alone is over 77 KB with 174 static imports at the top of the file. This architecture means that every time a new tool is added, the contributor must edit both files manually — if they forget one, the tool either does not render or has no metadata/route. Additionally, importing all 174 components eagerly in a single file defeats Next.js code splitting, potentially loading all tools into the initial JavaScript bundle.

**Why It Is Required**

This manual dual-registration pattern is the number one source of contributor errors and is fundamentally unscalable. As the project grows beyond 200+ tools, the constants files will become increasingly unmaintainable. The eager importing pattern also directly impacts the application's initial load performance — a new visitor downloading the JavaScript for all 174 tools when they only need one is wasteful. Replacing this with an auto-discovery or dynamic import system would make the codebase significantly easier to contribute to, reduce bugs, and improve performance through proper code splitting.

**My Approach**

I would refactor the tool registration system to use a convention-based auto-discovery approach. Each tool component would export its own metadata (slug, title, description, category) as a named export or a separate co-located metadata file (e.g., `base64Encoder.meta.ts`). A build-time script or Next.js dynamic import pattern would automatically discover all tools in the `developmentToolsComponent/` directory and generate the routes and metadata registry. The `PATHS` object in `constants.tsx` would be replaced with `next/dynamic` imports, enabling proper code splitting. The `developmentToolsConstant.tsx` metadata would be generated from the co-located metadata files. This approach ensures that adding a new tool only requires creating a single directory with the component and its metadata — no manual edits to shared files.

**I want to work on this issue.**

---

### Issue #H3: Implement Full WCAG 2.1 AA Accessibility Compliance Across the Application

**Problem Description**

An audit of the codebase reveals that only approximately 11 out of 300+ files contain any accessibility attributes (`aria-label`, `role`, etc.). The vast majority of interactive elements — buttons, text inputs, select dropdowns, file upload areas, and tool output regions — lack proper ARIA labels, roles, and states. The homepage search input has no `aria-label`. Filter buttons do not communicate their pressed/selected state to assistive technologies. Tool components do not announce output changes to screen readers. There are no skip-navigation links, and keyboard navigation through the tool interface is not explicitly managed. The color contrast between the primary green (`#00da92`) on certain backgrounds may not meet the 4.5:1 contrast ratio required by WCAG AA.

**Why It Is Required**

Accessibility is not optional — it is both a legal requirement in many jurisdictions (ADA in the US, EAA in the EU) and a moral imperative. Developer tools should be usable by all developers, including those who rely on screen readers, keyboard-only navigation, or other assistive technologies. The current state of the application would fail a WCAG 2.1 AA audit. Making the application accessible expands its user base, improves SEO (search engines favor accessible markup), and sets a positive example for the open-source community.

**My Approach**

I would conduct a full accessibility audit using automated tools (axe-core, Lighthouse) and manual testing with screen readers (NVDA, VoiceOver). I would then systematically address the findings: add `aria-label` attributes to all interactive elements across tool components, add `role` attributes where semantic HTML is insufficient, implement `aria-live` regions for dynamic tool output so screen readers announce changes, add skip-navigation links to `app/layout.tsx`, ensure all form elements in `app/components/theme/form/` have associated labels, verify and fix color contrast ratios in `tailwind.config.ts`, and add keyboard focus management for tool navigation. I would also add a reusable `AccessibleToolWrapper` component that provides consistent accessibility patterns for all tools.

**I want to work on this issue.**

---

### Issue #H4: Implement Robust Error Handling and User-Facing Error Feedback Across All Tool Components

**Problem Description**

Multiple tool components use empty `try-catch` blocks that silently swallow errors. For example, in `base64Encoder.tsx`, file reading errors are caught but produce no user feedback. In `jsonValidator.tsx`, parsing failures are caught but the error details are sometimes lost. Clipboard operations (`navigator.clipboard.writeText`) fail silently when the browser denies permission. File upload operations do not validate file types, sizes, or encoding before processing, and when processing fails, users see no indication that anything went wrong — the output area simply remains empty or shows stale data. There are no React Error Boundaries wrapping tool components, so a JavaScript error in a single tool can crash the entire application.

**Why It Is Required**

Silent failures are one of the worst user experience patterns. When a developer pastes invalid input, uploads a corrupted file, or encounters a browser permission issue, they need clear, actionable feedback — not a silent no-op. Without proper error handling, users may assume the tool is broken or, worse, trust incorrect output. The absence of Error Boundaries means a single component crash can take down the entire page, forcing a full reload and losing all user work across other tools. This is especially critical for tools handling sensitive operations like the bcrypt generator or JWT decoder.

**My Approach**

I would create a standardized error handling pattern for all tool components. First, I would implement a React Error Boundary component that wraps each tool, catching render errors and displaying a friendly recovery UI instead of crashing the page. Second, I would create a reusable `useToolError` hook that components can use to set, clear, and display error messages. Third, I would systematically audit all 174+ tool components, replacing empty `catch {}` blocks with proper error reporting that uses the hook to show inline error messages (e.g., "Invalid Base64 input", "File too large — maximum 5MB", "Clipboard access denied — please copy manually"). Fourth, I would add input validation before processing (file type checks, size limits, encoding validation). I would focus on the most critical tools first: validators, encoders/decoders, and generators.

**I want to work on this issue.**

---

### Issue #H5: Add Internationalization (i18n) Support with Multi-Language Translation Infrastructure

**Problem Description**

The entire application — including all 175+ tool names, descriptions, usage guides, error messages, button labels, placeholder texts, SEO metadata, and FAQ content — is hardcoded in English. There is no internationalization framework, no locale detection, no language switcher, and no translation file structure. The metadata in `app/libs/developmentToolsConstant.tsx` contains thousands of lines of English-only strings for tool descriptions, step-by-step guides, and SEO content. The UI components throughout `app/components/` have English strings directly embedded in JSX.

**Why It Is Required**

Developer tools have a global audience. A significant portion of developers worldwide are non-native English speakers who would benefit from localized interfaces. Adding i18n support would make the project accessible to a much wider community, improve SEO in non-English markets (each language generates unique search-indexable content), and align the project with modern web application standards. Additionally, an i18n-ready codebase makes it significantly easier for international contributors to participate by translating rather than coding.

**My Approach**

I would integrate `next-intl` (the recommended i18n library for Next.js App Router) into the project. I would restructure the route layout to support locale-prefixed URLs (e.g., `/en/json-formatter`, `/es/json-formatter`). I would create a `messages/` directory with JSON translation files, starting with English (`en.json`) as the base. All hardcoded strings in components and metadata would be replaced with translation keys using the `useTranslations` hook. The tool metadata in `developmentToolsConstant.tsx` would be refactored to use translation keys instead of raw strings. I would set up locale detection based on the browser's `Accept-Language` header with a language switcher in the header component. Initially, only English would be fully translated, but the infrastructure would enable community-driven translations for other languages.

**I want to work on this issue.**

---

### Issue #H6: Implement Performance Optimization with Bundle Analysis, Lazy Loading, and Core Web Vitals Improvement

**Problem Description**

The application eagerly imports all 174+ tool components in `app/libs/constants.tsx`, which means the JavaScript bundle for the entire tool suite may be included in the initial page load. There is no evidence of `React.lazy()`, `next/dynamic`, or any code-splitting strategy for tool components. The homepage (`app/page.tsx`) renders a filterable grid of all tools, but each tool card component and its icon are loaded upfront. The `app/components/theme/Icon/` directory contains over 200 custom SVG icon components, each in its own file, all potentially imported at build time. Animation libraries (Framer Motion, Anime.js) are loaded globally in `app/layout.tsx` even on pages that do not use animations. There is no visible bundle analysis configuration, no performance budget, and no monitoring of Core Web Vitals.

**Why It Is Required**

Page load performance directly impacts user experience, SEO rankings, and engagement. Google uses Core Web Vitals (LCP, FID, CLS) as ranking signals. An initial JavaScript bundle containing 174+ tool components and 200+ icons will result in a poor Time to Interactive and high Largest Contentful Paint. Users visiting a single tool should not download the code for all other tools. Performance optimization is especially critical for the PWA use case, where users on mobile or low-bandwidth connections need fast initial loads.

**My Approach**

I would start by adding `@next/bundle-analyzer` to measure the current bundle size and identify the largest contributors. Then I would convert all tool component imports in `constants.tsx` to use `next/dynamic` with `{ ssr: false }` where appropriate, ensuring each tool is loaded on demand when its route is visited. Icon components would be dynamically imported as well. I would implement `React.Suspense` boundaries with skeleton loading states for tool components. Animation libraries would be conditionally imported only on pages that use them. I would set up Lighthouse CI in the GitHub Actions workflow to track Core Web Vitals over time and enforce a performance budget. The goal would be to reduce the initial JavaScript bundle by at least 60% and achieve a Lighthouse Performance score of 90+.

**I want to work on this issue.**

---

### Issue #H7: Implement End-to-End Testing with Playwright for Critical User Workflows

**Problem Description**

Beyond the absence of unit tests (covered in Issue #H1), there are no end-to-end tests that verify the actual user experience of using the tools in a real browser. The application relies on browser APIs (Clipboard API, FileReader, navigator.userAgent, localStorage, Service Workers for PWA) that cannot be tested with unit tests alone. The dynamic routing system (`app/[slug]/page.tsx`) needs to be verified with actual navigation. The theme toggle (dark/light mode with localStorage persistence) needs browser-level testing. The PWA offline functionality needs verification. The Monaco Editor integration needs testing with real user interactions (typing, pasting, keyboard shortcuts). None of these critical user flows have any automated verification.

**Why It Is Required**

End-to-end tests are the only reliable way to verify that the application works correctly from the user's perspective across different browsers. The project uses several browser-specific APIs that can only be tested in a real browser environment. Without E2E tests, there is no way to catch regressions in user workflows — a CSS change could break the layout, a dependency update could break Monaco Editor, or a configuration change could break PWA caching — and none of these would be detected until users report them. For a tool suite that 175+ utilities depend on, automated E2E regression testing is essential.

**My Approach**

I would set up Playwright as the E2E testing framework (excellent Next.js integration, cross-browser support). I would configure it in `playwright.config.ts` with tests running against the Next.js dev server. I would write E2E tests for the following critical workflows: homepage search and filtering, navigating to a tool via slug, using the Base64 encoder (input text → verify output → copy to clipboard), using the JSON validator (paste invalid JSON → verify error display), theme toggle (switch themes → verify persistence across page reload), file upload in tools that support it, and verifying that the 404 page renders for invalid slugs. I would also add a Playwright test job to the GitHub Actions release workflow to run E2E tests on every pull request.

**I want to work on this issue.**

---

### Issue #H8: Implement a Centralized State Management Solution with Undo/Redo and Cross-Tool Data Sharing

**Problem Description**

Each tool component manages its own state independently using local `useState` hooks. There is no shared state layer between tools, no session persistence (beyond the theme toggle in localStorage), and no undo/redo capability. When a user converts JSON to YAML using one tool and then wants to validate the YAML output using another tool, they must manually copy-paste the output. If a user accidentally clears their input in a tool, there is no way to undo the action. The tool output history is not preserved — navigating away and coming back to a tool loses all previous work. The contexts in `app/contexts/` only handle layout hydration state and theme toggling — there is no application-level state management.

**Why It Is Required**

Developer workflows are inherently multi-step. A developer might encode a string to Base64, then URL-encode the result, then embed it in a JSON payload, then validate the JSON. Currently, each step requires manual copy-paste between tools, which is friction-heavy and error-prone. Adding cross-tool data piping, undo/redo, and session persistence would transform the application from a collection of isolated utilities into a cohesive developer workbench. This would significantly improve the user experience and differentiate the project from competing tool suites.

**My Approach**

I would implement a lightweight state management layer using React Context combined with `useReducer` for predictable state transitions. The state would include: a clipboard/pipe buffer that tools can read from and write to (enabling "Send output to..." functionality), an undo/redo stack per tool (tracking the last 20 input states), and session persistence using localStorage (so tool inputs survive page navigation and browser refresh). I would create a `useToolState` hook that wraps `useState` with undo/redo support and optional persistence. A new "Tool Pipeline" UI element in the header would show the current piped data and allow users to select a destination tool. The implementation would be backward-compatible — existing tool components would continue to work without modification, and the new features would be opt-in.

**I want to work on this issue.**

---

### Issue #H9: Implement Content Security Policy, Input Sanitization, and Security Hardening

**Problem Description**

The application has several security concerns. First, `app/layout.tsx` uses `dangerouslySetInnerHTML` to inject third-party analytics scripts (Google Tag Manager, PostHog, Gleap) directly into the document head, bypassing React's XSS protections. Second, there is no Content Security Policy (CSP) header configured anywhere — not in `next.config.js`, not in middleware, and not in meta tags. Third, user inputs across tool components are processed without sanitization — while most tools operate client-side (reducing server-side injection risk), tools that render HTML output (such as the HTML viewer, markdown-to-HTML converter, and BBCode converter) could execute malicious script tags embedded in user input. Fourth, the `javascript-obfuscator` dependency processes arbitrary user-supplied JavaScript code, which, while client-side, could lead to unexpected behavior. Fifth, environment variables like `GLEAP_API_KEY` and analytics IDs are referenced in client-side code.

**Why It Is Required**

Security is a foundational requirement for any web application, especially one that processes arbitrary user input. Even though the tools run client-side, XSS vulnerabilities in HTML rendering tools could be exploited if users share tool URLs with pre-filled malicious input (a common pattern via query parameters). The absence of CSP headers means the application has no defense against injected scripts. API keys exposed in client-side bundles can be harvested. As the project grows and potentially adds server-side features (APIs, user accounts), the lack of security foundations will become a critical liability.

**My Approach**

I would implement security hardening in layers. First, I would add a strict Content Security Policy via Next.js middleware (`middleware.ts`) that whitelists only trusted script and style sources. Second, I would replace `dangerouslySetInnerHTML` usage in `layout.tsx` with the Next.js `<Script>` component, which provides built-in CSP nonce support. Third, I would add input sanitization (using a library like DOMPurify) to all tools that render HTML output — specifically the HTML viewer, markdown-to-HTML converter, BBCode-to-HTML converter, and any tool that uses `dangerouslySetInnerHTML` or `innerHTML`. Fourth, I would audit environment variable usage and ensure sensitive keys are only available server-side using the `NEXT_PUBLIC_` prefix convention correctly. Fifth, I would add security headers (X-Content-Type-Options, X-Frame-Options, Referrer-Policy) in `next.config.js`.

**I want to work on this issue.**

---

### Issue #H10: Implement an Automated Tool Scaffolding CLI and Contributor Onboarding System

**Problem Description**

Adding a new tool to the project currently requires modifying at least 5 different files: creating the component in `app/components/developmentToolsComponent/`, importing it in `app/libs/constants.tsx`, adding its metadata and SEO information in `app/libs/developmentToolsConstant.tsx`, optionally adding an icon in `app/components/theme/Icon/`, and updating any category-related filtering. The `CONTRIBUTING.md` file provides general guidelines but does not include step-by-step instructions for adding a new tool, and there is no template or generator to scaffold the boilerplate. This high barrier to entry discourages new contributors and makes the process error-prone — contributors frequently forget to register the tool in one of the required files, leading to broken builds or invisible tools.

**Why It Is Required**

The project's primary growth vector is community contributions — new tools added by developers. If adding a tool is difficult, error-prone, and poorly documented, fewer contributors will participate, and those who do will produce inconsistent or incomplete work. A scaffolding CLI would reduce the time to add a new tool from 30+ minutes (including learning the registration pattern) to under 2 minutes, while ensuring consistency and correctness. This is especially important for open-source events (like Hacktoberfest or Apertre) where many first-time contributors attempt to add tools.

**My Approach**

I would create a Node.js CLI script (`scripts/create-tool.js`) that prompts the contributor for the tool name, slug, category, and description, then automatically generates: the component file from a template with the standard tool pattern (input area, output area, copy/clear buttons, file upload), the metadata entry in the constants file, the route registration, and a basic test file. The script would use `inquirer` for interactive prompts. I would also enhance `CONTRIBUTING.md` with a detailed "Adding a New Tool" section that references the CLI and explains each file's purpose. Additionally, I would add a GitHub Issue Template specifically for "New Tool Proposal" that collects the required information (tool name, category, input/output format, use case) to streamline the contribution pipeline.

**I want to work on this issue.**

---

## Medium-Level Issues

---

### Issue #M1: Add Dark Mode Support with Proper Theme Persistence and Consistent Styling Across All Components

**Problem Description**

While the codebase includes a `themeContext.tsx` that toggles between light and dark themes and persists the selection to localStorage (under the key `nestify-theme`), the actual dark mode styling is incomplete across the application. Many tool components in `app/components/developmentToolsComponent/` use hardcoded colors in their SCSS modules or inline styles that do not respond to the theme toggle. The Monaco Editor instances in code formatting tools do not switch their theme when the application theme changes. Some Ant Design components use their default light theme regardless of the application's dark mode state. The SCSS variables in `app/styles/variables.scss` do not define dark mode variants.

**Why It Is Required**

Dark mode is a standard expectation for developer tools — the majority of developers use dark-themed IDEs and prefer dark mode in their web applications. An incomplete or inconsistent dark mode implementation is worse than no dark mode at all, as it creates a jarring experience where some parts of the page are dark and others are light. Completing the dark mode implementation would improve user experience and align with developer expectations.

**My Approach**

I would audit all tool components and SCSS modules to identify hardcoded color values. I would extend `tailwind.config.ts` with a complete dark mode color palette using Tailwind's `dark:` variant. I would update `variables.scss` to use CSS custom properties that change based on a `data-theme` attribute on the root element. I would configure the Monaco Editor instances to use the `vs-dark` theme when dark mode is active. I would ensure Ant Design components respect the theme by wrapping them with Ant Design's `ConfigProvider` with the appropriate theme token. Finally, I would test all 175+ tools in both light and dark modes to ensure visual consistency.

**I want to work on this issue.**

---

### Issue #M2: Implement TypeScript Strict Mode and Eliminate All `any` Type Usage

**Problem Description**

The `tsconfig.json` has TypeScript strict mode partially configured, but the codebase makes extensive use of the `any` type, bypassing TypeScript's type safety guarantees. The most critical instance is in `app/libs/developmentToolsConstant.tsx` where the main tool metadata export is typed as `any` (e.g., `export const DEVELOPMENTTOOLS: any`), meaning the entire metadata structure — used by every page in the application — has no type checking. Tool components frequently use `any` for event handlers, state values, and function parameters. The dynamic routing in `app/[slug]/page.tsx` accesses tool data without type narrowing.

**Why It Is Required**

TypeScript's value proposition is entirely negated when `any` is used pervasively. Bugs that TypeScript should catch at compile time — misspelled property names, wrong argument types, missing required fields — slip through silently. When the tool metadata is typed as `any`, contributors can add malformed metadata entries that break the application at runtime without any compiler warning. Eliminating `any` types and defining proper interfaces would prevent an entire class of bugs and make the codebase significantly safer to modify.

**My Approach**

I would define comprehensive TypeScript interfaces for all major data structures: `ToolMetadata` (title, slug, description, category, SEO, steps, FAQ), `ToolComponent` (the component props contract), `ToolCategory` (category names and filters), and `ThemeConfig`. I would replace the `any` type in `developmentToolsConstant.tsx` with the `ToolMetadata` interface, fix all resulting type errors (which will reveal actual bugs), and then systematically go through the tool components replacing `any` with proper types. I would enable `"noImplicitAny": true` in `tsconfig.json` to prevent future `any` usage and add an ESLint rule (`@typescript-eslint/no-explicit-any`) to enforce this at the linting level.

**I want to work on this issue.**

---

### Issue #M3: Add File Size Validation, Type Checking, and Progress Indicators for File Upload Operations

**Problem Description**

Several tool components support file uploads (Base64 encoder, CSV converters, JSON formatter, code compare tools, etc.), but none of them validate the uploaded file before processing. There is no maximum file size limit — a user could upload a 500 MB file, causing the browser tab to freeze or crash as the `FileReader` API attempts to read the entire file into memory. There is no file type validation — uploading a binary file to a text-based tool produces garbled output with no error message. There is no upload progress indicator — for larger files, the UI appears frozen while the `FileReader` is working. The `FileReader` `onerror` event is not handled in most components.

**Why It Is Required**

File upload is a core feature of many tools, and without proper validation and feedback, it creates a poor and potentially dangerous user experience. A user accidentally uploading a large file could crash their browser tab, losing any other work in the same tab. Uploading an incorrect file type wastes time and produces confusing output. The absence of progress indicators makes the application feel unresponsive. These are standard UX patterns that users expect from any file-handling web application.

**My Approach**

I would create a reusable `useFileUpload` hook that encapsulates file upload logic with built-in validation and feedback. The hook would accept configuration options: `maxSizeBytes` (default 5 MB), `acceptedTypes` (MIME types or extensions), and `encoding` (default UTF-8). It would validate the file before reading, show a progress indicator using the `FileReader` `onprogress` event, handle errors with user-friendly messages, and return the file content along with loading/error states. I would then refactor all file-upload-supporting tool components to use this hook instead of their ad-hoc `FileReader` implementations. The upload area UI would show accepted file types, maximum size, and a progress bar during upload.

**I want to work on this issue.**

---

### Issue #M4: Implement URL-Based State Sharing for Tool Inputs via Query Parameters

**Problem Description**

Currently, there is no way to share a pre-filled tool state via URL. If a developer uses the Base64 encoder to encode a specific string and wants to share the exact input and output with a colleague, they must separately communicate the input text — they cannot simply share a URL. The tool pages use dynamic routes (`/[slug]`) but do not read or write query parameters for tool state. This limits the utility of the tools in collaborative and documentation contexts.

**Why It Is Required**

URL-based state sharing is a standard feature of online developer tools (like jwt.io, base64encode.org, and regex101.com). It enables developers to share encoded tool states in Slack messages, GitHub issues, documentation, Stack Overflow answers, and bug reports. It also enables bookmarking specific tool configurations for repeated use. Without this feature, the tools are limited to individual, ephemeral use — a significant missed opportunity for viral growth and developer productivity.

**My Approach**

I would implement a URL query parameter system where tool inputs are compressed and encoded in the URL hash (using a combination of `encodeURIComponent` and optional LZ-string compression for large inputs). Each tool component would read initial state from the URL on mount and update the URL (using `window.history.replaceState` to avoid navigation) when input changes (debounced to avoid performance issues). A "Share" button would be added to each tool's UI that copies the shareable URL to the clipboard. For large inputs that exceed URL length limits (~2000 characters), the Share button would show a warning suggesting the user share the input separately. The implementation would be backward-compatible — existing bookmarked URLs without query parameters would continue to work normally.

**I want to work on this issue.**

---

### Issue #M5: Add Keyboard Shortcuts and Power-User Navigation for Tool Operations

**Problem Description**

The application has no keyboard shortcuts. Every operation — clearing input, copying output, switching between input and output panels, navigating between tools, toggling the theme, and searching for tools — requires mouse interaction. For a developer tool suite, this is a significant usability gap. Developers are keyboard-centric users who expect keyboard shortcuts in their tools. The homepage search is not focused on page load. There is no way to navigate between tools using the keyboard. Common operations like "copy output" and "clear input" are only accessible via button clicks.

**Why It Is Required**

Developer tools live and die by their keyboard usability. The target audience spends most of their working day with hands on the keyboard, and forcing mouse interactions for common operations creates friction. Keyboard shortcuts also improve accessibility for users who cannot use a mouse. Popular developer tools (VS Code, DevTools, Postman) all provide extensive keyboard shortcut support. Adding shortcuts would make the tools faster to use and align with developer expectations.

**My Approach**

I would implement a keyboard shortcut system using a lightweight event listener approach. I would define global shortcuts (accessible from any page): `Ctrl/Cmd + K` to focus the tool search, `Ctrl/Cmd + /` to toggle the theme, and arrow keys for tool navigation on the homepage. I would define tool-level shortcuts (accessible when inside a tool): `Ctrl/Cmd + Enter` to execute/process, `Ctrl/Cmd + Shift + C` to copy output, `Ctrl/Cmd + Shift + X` to clear input, and `Ctrl/Cmd + Z` for undo (when combined with Issue #H8). I would create a `useKeyboardShortcuts` hook that handles registration, conflict resolution, and platform detection (Ctrl vs Cmd). A keyboard shortcut reference panel (opened with `?`) would list all available shortcuts. The shortcuts would be discoverable via tooltips on action buttons.

**I want to work on this issue.**

---

### Issue #M6: Implement Comprehensive SEO Improvements with Structured Data, Sitemap, and OpenGraph Images

**Problem Description**

While the application has basic SEO elements (meta titles, descriptions, and a canonical link component in `app/components/theme/SEOComponent/`), several important SEO features are missing. There is no `sitemap.xml` generation for the 175+ tool pages. There are no OpenGraph images for social media sharing — when a tool URL is shared on Twitter, LinkedIn, or Slack, it shows no preview image. The structured data (JSON-LD) in tool pages is basic and does not include `HowTo` schema for the step-by-step guides or `FAQPage` schema for the FAQ sections that many tools have. The `robots.txt` configuration is not visible. Tool pages do not include breadcrumb structured data.

**Why It Is Required**

SEO is the primary organic growth channel for developer tools. Developers frequently search for specific utilities (e.g., "base64 encode online", "json formatter", "csv to json converter"), and proper SEO ensures the tools rank well. A comprehensive sitemap helps search engines discover all 175+ tool pages. OpenGraph images dramatically increase click-through rates from social media shares. Structured data (HowTo, FAQ, Breadcrumb schemas) enables rich search results that stand out in Google. These improvements could significantly increase organic traffic without any paid marketing.

**My Approach**

I would implement dynamic `sitemap.xml` generation using Next.js's built-in `sitemap.ts` file in the `app/` directory, iterating over all tool routes in the constants to generate entries with appropriate `changefreq` and `priority` values. I would add a `robots.txt` via `app/robots.ts`. For OpenGraph images, I would use Next.js's `opengraph-image.tsx` route convention to generate dynamic OG images for each tool featuring the tool name, category, and a visual element. I would enhance the structured data in `app/[slug]/page.tsx` to include `HowTo` schema (mapping the existing step-by-step data), `FAQPage` schema (mapping the existing FAQ data), and `BreadcrumbList` schema. I would also add `hreflang` tags to support future internationalization (Issue #H5).

**I want to work on this issue.**

---

### Issue #M7: Refactor Tool Components to Use a Shared Layout System with Consistent Input/Output Panels

**Problem Description**

Each of the 174+ tool components in `app/components/developmentToolsComponent/` independently implements its own layout structure for input areas, output areas, action buttons (copy, clear, upload), and option panels. This leads to significant code duplication and visual inconsistency across tools. Some tools use `<textarea>` for input, others use Monaco Editor, and others use custom div-based inputs — with different padding, border styles, and sizes. The copy-to-clipboard logic is duplicated in every component with slight variations. The clear/reset logic is duplicated. File upload handling is duplicated. Button styling and positioning vary between tools. This duplication makes it difficult to implement cross-cutting changes (like adding a new button or changing the panel layout) because every component must be updated individually.

**Why It Is Required**

Code duplication is the primary source of inconsistency and maintenance burden. When a bug is fixed in one tool's copy-to-clipboard logic, the same bug remains in 173 other tools. When the design team wants to change the tool layout, 174+ components must be updated. A shared layout system would reduce code per tool component by 40–60%, enforce visual consistency, and enable cross-cutting features (like keyboard shortcuts or accessibility improvements) to be implemented once and applied everywhere.

**My Approach**

I would create a `ToolLayout` component system with composable sub-components: `ToolLayout.InputPanel` (supports textarea, Monaco Editor, or custom content), `ToolLayout.OutputPanel` (with built-in copy button and format options), `ToolLayout.ActionBar` (standardized buttons for copy, clear, upload, download), and `ToolLayout.OptionsPanel` (for tool-specific configuration). The clipboard logic, file upload logic, and clear logic would be centralized in this layout system. I would then refactor 10–15 representative tools to use the new layout system as proof of concept, ensuring backward compatibility. The remaining tools would be migrated incrementally by contributors. The shared layout would automatically provide consistent dark mode support, accessibility attributes, and responsive design.

**I want to work on this issue.**

---

### Issue #M8: Add Offline Capability Improvements with Service Worker Caching Strategy and Offline Fallback UI

**Problem Description**

The application uses `next-pwa` for Progressive Web App support, but the configuration in `next.config.js` is minimal — it enables PWA only when `NEXT_ENV` is not `local` and provides no custom caching strategy. The default `next-pwa` behavior may not cache tool component JavaScript optimally, meaning tools that were previously visited might not work offline. There is no offline fallback page — when a user goes offline and navigates to a tool they have not previously visited, they likely see a browser-level "No internet" error instead of a friendly app-level offline page. The service worker registration, update, and lifecycle are not managed — users might be running stale cached versions without knowing.

**Why It Is Required**

Offline capability is one of the key differentiators of the application (highlighted in the README as "Works offline"). However, the current implementation provides unreliable offline support. For a PWA that advertises offline functionality, it needs to deliver a consistent and predictable offline experience. Developers on flights, in areas with poor connectivity, or on metered connections should be able to rely on the cached tools. A proper caching strategy and offline fallback UI would fulfill the PWA promise and improve user trust.

**My Approach**

I would enhance the `next-pwa` configuration in `next.config.js` with a custom runtime caching strategy: tool component JavaScript would use a `StaleWhileRevalidate` strategy (serve cached version immediately, update in background), static assets would use `CacheFirst`, and API routes would use `NetworkFirst` with a fallback. I would create an offline fallback page (`app/offline.tsx`) that displays when a user navigates to an uncached page while offline, showing a friendly message and a list of tools that are available in the cache. I would add a service worker update notification banner that informs users when a new version is available and offers a "Refresh" button. I would also add a visual indicator in the header showing online/offline status.

**I want to work on this issue.**

---

### Issue #M9: Implement Tool Usage Analytics and Popular Tools Ranking on the Homepage

**Problem Description**

The homepage currently displays all 175+ tools in a flat grid with category-based filtering, but there is no indication of which tools are most popular or frequently used. The tools are ordered statically based on their position in the `developmentToolsConstant.tsx` array. While the application integrates PostHog and Google Tag Manager for analytics, these are general-purpose analytics tools that do not feed data back into the application's UI. There is no client-side usage tracking that could power features like "Recently Used Tools", "Most Popular Tools", or "Recommended for You" — all of which would help users discover relevant tools faster.

**Why It Is Required**

With 175+ tools, discoverability is a significant challenge. New users visiting the homepage are overwhelmed by the sheer number of options. Showing popular tools first would guide users to the most valuable tools. A "Recently Used" section would improve repeat-visit efficiency — developers tend to use the same 5–10 tools regularly and currently must search or scroll to find them each time. Tool usage data also informs product decisions — knowing which tools are used most helps prioritize improvements and which are unused helps identify candidates for removal or redesign.

**My Approach**

I would implement client-side usage tracking using localStorage. Each time a user visits a tool, its slug and timestamp would be recorded. I would create a `useToolAnalytics` hook that provides: `recentlyUsed` (last 10 tools visited, ordered by recency), `mostUsed` (top 10 tools by visit count), and `trackToolVisit(slug)`. The homepage would be enhanced with two new sections above the main grid: "Recently Used" (showing the user's last 5 tools with one-click access) and "Popular Tools" (statically curated initially, potentially crowd-sourced later). A "Clear History" option would be available for privacy-conscious users. The tracking would be entirely client-side with no data sent to any server, consistent with the project's privacy-first philosophy.

**I want to work on this issue.**

---

### Issue #M10: Add Export and Download Functionality for Tool Outputs in Multiple Formats

**Problem Description**

Currently, the only way to get output from a tool is to copy it to the clipboard using the copy button. There is no option to download the output as a file. Tools like the JSON formatter, CSV converter, code compare tool, and barcode/QR code generator produce output that users frequently need as files — a formatted JSON file, a converted CSV file, a diff report, or a generated image. Users must copy the output, open a text editor, paste, and save manually. The barcode and QR code generators (`barcodeGenerator.tsx`, QR code component) render visual output on canvas/SVG but provide no "Download as PNG/SVG" option.

**Why It Is Required**

Download functionality is a standard expectation for any online conversion or generation tool. The copy-to-clipboard workflow adds unnecessary friction, especially for binary outputs (images, Excel files) or large outputs (formatted JSON files, CSV conversions) that are awkward to handle via clipboard. For the barcode and QR code generators, downloading the generated image is the primary use case — without it, users must take screenshots, which produces lower quality results. Adding download support would complete the tool workflow: input → process → download.

**My Approach**

I would create a reusable `useDownload` hook and `DownloadButton` component that supports multiple output formats. For text-based tools (formatters, converters, encoders), the download would generate a Blob with the correct MIME type and trigger a file download with an appropriate filename (e.g., `formatted.json`, `converted.csv`, `encoded.txt`). For image-based tools (barcode generator, QR code generator, color picker), the download would export the canvas/SVG as PNG or SVG files. The `DownloadButton` component would offer format selection where applicable (e.g., JSON formatter could download as `.json` or `.txt`). I would integrate this download capability into the shared `ToolLayout.ActionBar` (from Issue #M7) so it is consistently available across all tools. I would prioritize implementing downloads for the 20 most-used tools first.

**I want to work on this issue.**

---

*All issues are scoped to the [betterbugs/dev-tools](https://github.com/betterbugs/dev-tools) repository and its codebase as of the latest release.*
