# Product Requirements Document (PRD) for AI Image Generation App

## 1. Overview

### 1.1 Product Name
GenImage (placeholder; can be customized).

### 1.2 Product Description
GenImage is a mobile-first Progressive Web App (PWA) that allows users to generate AI-powered images based on text prompts, style selections, and optional image attachments. It operates on a subscription model where users receive credits for image generations. Free users have no generation access, while subscribed users can track and manage their credits. The app emphasizes ease of use, creative styles, and social sharing. It draws inspiration from apps like Genpost, incorporating features like prompt-based generation, style presets, and credit limits.

### 1.3 Objectives
- Provide an accessible AI image generation tool with monetization via subscriptions.
- Ensure secure user authentication and data management.
- Limit generations to prevent abuse and encourage upgrades.
- Support image attachments for advanced generations (e.g., image-to-image transformations).
- Deliver a seamless PWA experience for cross-device compatibility.

### 1.4 Scope
- In Scope: User auth, subscription/credit system, image generation, style selection, image attachment, credit tracking, PWA features, multi-language support.
- Out of Scope: Advanced admin dashboard, real-time collaboration, video generation, native mobile builds (focus on PWA).

### 1.5 Key Metrics for Success
- User acquisition: 1,000 active users in first 3 months.
- Retention: 70% monthly active users.
- Conversion: 20% of free users upgrade to pro.
- Generation volume: Average 50 generations per pro user/month.
- App performance: <2s load time, 99% uptime.

## 2. Target Audience
- **Primary Users**: Creative individuals (artists, designers, hobbyists) aged 18-35 seeking quick AI image creation for social media, memes, or personal use, with a focus on Ethiopian and regional users.
- **Secondary Users**: Marketers and educators needing custom visuals.
- **Pain Points Addressed**: High costs of premium AI tools; lack of simple mobile interfaces; credit transparency; limited language support in creative tools.
- **Competitors**: Midjourney, DALL-E apps, Genpost – Differentiators: Affordable credits, image attachment, PWA offline support, multi-language support (starting with English, Amharic, and Oromo for broader accessibility in diverse linguistic regions like Ethiopia).

## 3. Features

### 3.1 Core Features
- **Image Generation**:
  - Users input text prompts (e.g., "make it criminal comedy fun").
  - Select from preset styles (e.g., Vintage, Celebration, Cartoonish, Cinematic, Meme & Trendy, Dreamy, Renaissance Art).
  - Attach images for reference (e.g., upload for image-to-image generation).
  - Generate button triggers AI processing (integrate with external API like Stability AI or Hugging Face).
  - Display generated image with options: Edit (regenerate with tweaks), Download, Share on socials (e.g., Instagram, Snapchat).
  - Auto-generate image descriptions for accessibility/sharing.

- **Credit System**:
  - Subscribed users start with a monthly credit allocation (e.g., 700 generations for pro plan).
  - Deduct 1 credit per generation.
  - Real-time display of remaining credits (e.g., "12 Left").
  - Limit reached: Show popup with upgrade/buy options; prevent further generations.
  - Free users: View app but blocked from generating (prompt to subscribe).

- **Subscription Management**:
  - Plans: Free (view-only), Pro (e.g., 99.99 USD/month or equivalent in local currencies like ETB for ~700 credits).
  - Integrate payment gateway (e.g., Stripe) for purchases in multiple currencies (start with USD and ETB).
  - Options to buy add-on credits (e.g., 5 generations for 15 ETB or equivalent).
  - Auto-renewal with cancellation via app.

- **User Management and Profiles**:
  - Profile page: Display username, pro status, theme mode (light/dark), credit balance.
  - Edit profile: Change username, password, avatar.
  - Account security: Change password, delete account.
  - History: View past generations with download/share options.

- **Authentication**:
  - Sign up/login via email/password, Google, or other OAuth (using Better Auth).
  - Session management: Persistent login across devices.
  - Password reset and email verification.

### 3.2 Additional/Missing Features
Based on the provided design and requirements, the following are added for completeness:
- **Onboarding**: Guided tutorial for first-time users (e.g., explain prompts, styles, attachments), available in supported languages.
- **Gallery/History**: Store up to 100 past images per user (server-side with SQLite references).
- **Style Customization**: Allow adding custom styles or modifiers (e.g., "High resolution").
- **Offline Support (PWA)**: Cache generated images and UI for offline viewing; queue generations for online sync.
- **Notifications**: Push alerts for credit low, subscription renewal, or generation complete (via service workers), localized in user's language.
- **Sharing and Social Integration**: Direct share buttons; generate shareable links with watermarks for free users.
- **Analytics and Feedback**: In-app rating prompt; track usage for improvements.
- **Accessibility**: Alt text for images, voice input for prompts, high-contrast modes; ensure multi-language support for screen readers.
- **Error Handling**: Graceful failures (e.g., "API down – try later") with retries, messages translated.
- **Admin Tools** (Basic): Developer console to manage users, view subscriptions (accessible via auth role).
- **Privacy and Legal**: Display privacy policy and terms (editable; last updated date shown), translated into supported languages.
- **Theming**: Light/dark mode toggle.
- **Internationalization and Multi-Language Support**: 
  - Support multiple languages starting with English (en), Amharic (am), and Oromo (om).
  - Use react-i18next for frontend localization; handle RTL for Amharic if needed.
  - Language selection in profile or onboarding; auto-detect based on device locale.
  - Translate all UI elements, prompts, error messages, and legal texts.
  - Future expansion: Add more languages via dynamic loading of translation files.
  - Currency support tied to language/region (e.g., ETB for Amharic/Oromo users).
- **Multi-Currency Support**: Integrate with payment gateway to handle local currencies (e.g., ETB for Ethiopian users), displayed based on selected language/region.

### 3.3 User Flows
- **Generation Flow**:
  1. Login → Home screen with prompt input, style buttons, attach image option (all localized).
  2. Submit → Check credits → Generate → Display image with actions.
- **Subscription Flow**:
  1. Profile → Upgrade button → Select plan → Payment (in local currency) → Update credits.
- **Limit Reached Flow**:
  1. Attempt generation → Popup: "Daily Limit Reached" (translated) → Buy options.
- **Language Switch Flow**:
  1. Profile → Language selector → Choose Amharic/Oromo/English → App reloads with new locale.

## 4. User Stories
- As a free user, I want to browse styles and prompts so I can preview the app.
- As a pro user, I want to see my credit balance so I can manage usage.
- As a user, I want to attach an image to my prompt so I can refine generations.
- As a user, I want to download/share generated images so I can use them externally.
- As an admin, I want to view user lists so I can monitor activity.
- As an Ethiopian user, I want the app in Amharic or Oromo so I can interact comfortably.
- As a user, I want to switch languages easily so I can share the app with multilingual friends.

## 5. Technical Requirements

### 5.1 Architecture
- **Mono Repo Structure**:
  - Root: Shared configs (e.g., ESLint, TypeScript).
  - /frontend: React + Vite + TanStack Router + Shadcn UI + Tailwind CSS + react-i18next for localization.
  - /backend: Hono API server + SQLite DB.
  - /shared: Types, utils (e.g., auth interfaces, translation keys).

- **Frontend**:
  - Framework: React with Vite for build tooling.
  - Routing: TanStack Router for dynamic routes (e.g., /generate, /profile).
  - UI: Shadcn components (buttons, modals, cards) styled with Tailwind CSS.
  - PWA: Vite PWA plugin for manifest, service workers, offline caching.
  - Auth: Better Auth client integration.
  - Localization: react-i18next with JSON files for English, Amharic, and Oromo; i18n for dynamic text.

- **Backend**:
  - Framework: Hono for lightweight API.
  - Database: SQLite for user data, credits, generation history (migrate to PostgreSQL if scaling); store user language preference.
  - Auth: Better Auth server-side for sessions, JWT.
  - Endpoints: /auth (login/signup), /generate (proxy to AI API, deduct credits), /credits (get balance), /subscriptions (manage plans), /language (update user locale).
  - AI Integration: Call external API (e.g., Replicate or OpenAI) for image generation; handle attachments via multipart uploads; support localized prompts if AI allows.

- **Integrations**:
  - Payments: Stripe for subscriptions/add-ons, with multi-currency support.
  - AI: Stability AI or equivalent (handle API keys securely).
  - Storage: Cloudinary or S3 for image hosting (SQLite stores URLs).
  - Localization: i18next-backend for loading translations if needed.

### 5.2 Non-Functional Requirements
- **Performance**: API responses <500ms; image generation <10s; language switches <1s.
- **Security**: HTTPS, input sanitization, rate limiting; store passwords hashed; handle language-specific data securely.
- **Scalability**: Handle 10k users; SQLite sufficient initially.
- **Compatibility**: Browsers (Chrome, Safari, Firefox); mobile/desktop via PWA; support for Ge'ez script in Amharic.
- **Testing**: Unit (Jest), E2E (Cypress), coverage >80%; include tests for multi-language rendering.
- **Deployment**: Vercel/Netlify for frontend; Fly.io for backend; CI/CD with GitHub Actions.

## 6. Assumptions and Dependencies
- Assumptions: Users have internet for generations; AI API costs covered by subscriptions; translation accuracy verified by native speakers.
- Dependencies: External AI service availability; payment gateway integration; third-party libraries for localization.
- Risks: AI API downtime (mitigate with fallbacks); payment failures (handle retries); translation errors (mitigate with reviews).

## 7. Roadmap
- **MVP (Month 1)**: Auth, basic generation, credit system, PWA setup, initial English support.
- **Phase 2 (Month 2)**: Subscriptions, image attachments, history, add Amharic and Oromo translations.
- **Phase 3 (Month 3)**: Notifications, analytics, polish (e.g., theming, full multi-language testing).
- **Launch**: Beta test with 100 users (including multilingual); full release with marketing.
