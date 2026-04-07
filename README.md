# Epicura-UX (User Interface & Experience)

UI/UX design documentation and interactive mockups for the [Epicura](../README.md) autonomous kitchen robot.

## Structure

```
Epicura-UX/
├── __init.md                    Workspace index
├── 04-UserInterface/
│   └── 01-UI-UX-Design.md      Touchscreen wireframes, companion app flows, multi-language support
└── mocks/                       Vite + React interactive mobile mockups
    ├── src/
    │   ├── App.jsx              Root app with screen routing
    │   ├── components/
    │   │   ├── MiniProgress.jsx Cooking progress indicator
    │   │   ├── OTPInput.jsx     OTP verification input
    │   │   ├── RecipeCard.jsx   Recipe card with image/details
    │   │   └── TabBar.jsx       Bottom navigation tabs
    │   ├── screens/
    │   │   ├── LoginScreen.jsx  Phone number login
    │   │   ├── OTPScreen.jsx    OTP verification
    │   │   ├── RecipeScreen.jsx Recipe browsing home
    │   │   ├── SessionScreen.jsx Live cooking session
    │   │   ├── FavouriteScreen.jsx Saved recipes
    │   │   └── ProfileScreen.jsx User profile/settings
    │   ├── data/mockData.js     Sample recipes and user data
    │   ├── theme.js             Design tokens (colors, spacing, typography)
    │   └── index.css            Global styles
    ├── index.html
    ├── package.json
    └── vite.config.js
```

## Running the Mockups

```bash
cd mocks
npm install
npm run dev       # Start dev server (http://localhost:5173)
npm run build     # Production build
npm run preview   # Preview production build
```

**Stack:** Vite 4 + React 18 + Lucide icons

## Design Scope

The mockups cover the mobile companion app screens. The full UI/UX specification in `01-UI-UX-Design.md` covers:

- **Touchscreen UI** (10" Kivy) — Recipe browsing, cooking progress, camera feed, settings
- **iOS companion app** (SwiftUI) — BLE pairing, recipe management, live cooking, notifications
- **Android companion app** (Kotlin/Compose) — Mirrors iOS with platform-native patterns
- **Multi-language support** — English, Hindi, regional languages
