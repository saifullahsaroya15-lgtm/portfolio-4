## Overview

`vivek9patel.github.io` is a portfolio that recreates an Ubuntu 20.04 desktop using Next.js, Tailwind CSS, and a set of custom window-management components. The experience includes boot/lock screens, draggable windows, wallpaper controls, faux apps (Chrome, VS Code, Spotify, Calculator, Contact, etc.), and GA4 analytics.

## Tech Stack

- Framework: Next.js 13 (Pages Router)
- UI: React 18 + Tailwind CSS + custom global styles (`styles/index.css`)
- Utilities: `react-draggable`, `react-onclickoutside`, `jquery`
- Analytics: `react-ga4`
- Email: `@emailjs/browser`

## Architecture

| Layer | Description |
| --- | --- |
| Pages | `_app.js` loads global CSS, `_document.js` defines the HTML shell, `index.js` renders SEO meta + the `Ubuntu` component. |
| `components/ubuntu.js` | Class component that orchestrates boot, lock/unlock, wallpaper, and shutdown state by persisting keys into `localStorage` and emitting GA4 events. |
| Screen shell | `components/screen/*` builds boot/lock overlays, navbar, desktop, dock, and application grid. |
| Window manager | `components/base/*` abstracts draggable windows, focus, z-index ordering, and sidebar app shortcuts. |
| Apps | `components/apps/*` exports JSX for each faux application and is referenced inside `apps.config.js`. |
| Utilities/SEO | Common widgets (clock, background image, status cards) plus `components/SEO/Meta`. |
| Assets | `public/` stores wallpapers, icons, resume PDF, robots.txt. |

## State & Data Flow

- Boot/lock/shutdown flags, wallpaper selections, and screen lock states are stored in `localStorage` keys read in `componentDidMount`.
- GA4 events (`ReactGA.send`, `ReactGA.event`) fire when users lock or shut down the simulated OS.
- Contact form (`components/apps/gedit.js`) sends mail via EmailJS using `NEXT_PUBLIC_*` credentials.
- Window creation is driven by `apps.config.js`; each entry wires icons, favorites, shortcuts, and the React component to render inside a `Window` shell.

## Configuration

Create `.env.local` with any of the following:

```
NEXT_PUBLIC_TRACKING_ID=<GA4 measurement id>
NEXT_PUBLIC_USER_ID=<emailjs public key>
NEXT_PUBLIC_TEMPLATE_ID=<emailjs template id>
NEXT_PUBLIC_SERVICE_ID=<emailjs service id>
```

Leaving a value empty disables that integration locally. Anything prefixed with `NEXT_PUBLIC_` becomes part of the browser bundle.

## Scripts

| Command | Purpose |
| --- | --- |
| `yarn dev` | Start dev server on `http://localhost:3000`. |
| `yarn build` | Production build into `.next`. |
| `yarn start` | Serve the production build (requires `yarn build`). |
| `yarn export` | Static export to `out/` (used for GitHub Pages). |

## Project Structure

```
├─ components/         # Ubuntu shell, windows, apps, utilities
├─ pages/              # Next.js pages (_app, _document, index)
├─ public/             # Wallpapers, icons, resume, robots.txt
├─ styles/index.css    # Global overrides on top of Tailwind
├─ apps.config.js      # Application registry consumed by desktop/dock
├─ DOCS.md             # Project documentation
├─ Dockerfile          # Multi-stage production image
└─ .github/workflows/  # CI/CD pipelines (Pages + EC2)
```

## Local Development

1. `yarn install`
2. `yarn dev` and browse `http://localhost:3000`
3. For production parity: `yarn build && yarn start`

## Docker Workflow

```
docker build -t vivek-portfolio .
docker run --rm -p 3000:3000 vivek-portfolio
```

The image uses multi-stage builds (Node 18 Alpine) and serves the optimized build with `yarn start`.

## GitHub Pages Deployment

Workflow: `.github/workflows/gh-deploy.yml`

1. Add repository secrets: `NEXT_PUBLIC_TRACKING_ID`, `NEXT_PUBLIC_SERVICE_ID`, `NEXT_PUBLIC_TEMPLATE_ID`, `NEXT_PUBLIC_USER_ID`.
2. Set the repo’s Pages source to “GitHub Actions”.
3. Push to `master` (or run manually). The job builds, runs `yarn export`, uploads `out/`, and deploys with `actions/deploy-pages`.

## EC2 Deployment Pipeline

Workflow: `.github/workflows/ec2-deploy.yml`

Pipeline outline:

1. Checkout + configure Node 18
2. `yarn install --frozen-lockfile`
3. `yarn build`
4. Package `.next`, `public`, `package.json`, and `yarn.lock` into `release.tar.gz`
5. Upload artifact, copy it to EC2 over SCP, unpack into `EC2_APP_DIR`
6. Install production dependencies on the instance and (re)start the `pm2` process named `vivek-portfolio`

Required repository secrets:

- `NEXT_PUBLIC_TRACKING_ID`, `NEXT_PUBLIC_SERVICE_ID`, `NEXT_PUBLIC_TEMPLATE_ID`, `NEXT_PUBLIC_USER_ID` (optional, for build-time embedding)
- `EC2_HOST`: public DNS or IP
- `EC2_USER`: SSH username (e.g., `ubuntu`, `ec2-user`)
- `EC2_SSH_KEY`: private key contents (PEM) with newline escapes (`-----BEGIN...`)
- `EC2_PORT`: SSH port (set `22` if default)
- `EC2_APP_DIR`: absolute path on the instance where the app should live (e.g., `/var/www/vivek-portfolio`)

Instance prerequisites:

- Node.js 18+, Yarn, and PM2 installed globally (`npm i -g yarn pm2`).
- `EC2_APP_DIR` owned by the SSH user.
- Optional `.env` file placed in `EC2_APP_DIR` for runtime env vars (non-public secrets).

PM2 will create/restart the process automatically; review with `pm2 status` and persist via `pm2 save`.

## Customization Tips

- Add/remove apps in `apps.config.js`, referencing icons from `public/themes`.
- Drop new wallpapers under `public/images/wallpapers` and expose them through the settings screen.
- Update SEO/social metadata inside `components/SEO/Meta.js`.
- Replace the resume PDF in `public/files/` and update links inside the “About” or “Contact” apps.

## Testing

No automated tests ship today. Suggested manual coverage:

- Boot screen vs returning visitor flow
- Lock/unlock persistence with localStorage
- Opening/closing each app window; drag/resize interactions
- Contact form submission (EmailJS)
- GA4 event visibility (optional via DebugView)
- Responsive behavior at tablet widths

Future work could involve adding component tests with React Testing Library for window manager and wallpaper selection logic.
