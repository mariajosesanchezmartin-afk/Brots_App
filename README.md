Brots PWA — Projecte desplegable

Crea una carpeta brots-app i hi poses aquests fitxers.


1. package.json

json{
  "name": "brots-app",
  "private": true,
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "lucide-react": "^0.383.0",
    "react": "^18.3.1",
    "react-dom": "^18.3.1",
    "recharts": "^2.12.7"
  },
  "devDependencies": {
    "@vitejs/plugin-react": "^4.3.1",
    "autoprefixer": "^10.4.19",
    "postcss": "^8.4.39",
    "tailwindcss": "^3.4.6",
    "vite": "^5.3.4"
  }
}


2. vite.config.js

jsimport { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
});


3. tailwind.config.js

jsexport default {
  content: ["./index.html", "./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
};


4. postcss.config.js

jsexport default {
  plugins: { tailwindcss: {}, autoprefixer: {} },
};


5. index.html

html<!doctype html>
<html lang="ca">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
    <title>Brots · Educació i desenvolupament 0–6 anys</title>
    <meta name="description" content="App gratuïta en català sobre educació i desenvolupament infantil de 0 a 6 anys. Basada en OMS, AEP i COPC." />
    <link rel="manifest" href="/manifest.json" />
    <meta name="theme-color" content="#059669" />
    <link rel="apple-touch-icon" href="/icons/icon-192.png" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="default" />
    <meta name="apple-mobile-web-app-title" content="Brots" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>


6. src/main.jsx

jsximport React from "react";
import ReactDOM from "react-dom/client";
import App from "./App.jsx";
import "./index.css";

ReactDOM.createRoot(document.getElementById("root")).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);

// Registre del service worker
if ("serviceWorker" in navigator) {
  window.addEventListener("load", () => {
    navigator.serviceWorker.register("/sw.js").catch((err) =>
      console.error("SW no registrat:", err)
    );
  });
}


7. src/index.css

css@tailwind base;
@tailwind components;
@tailwind utilities;

html, body, #root { height: 100%; }
body { background: #FAFAF9; -webkit-tap-highlight-color: transparent; }


8. public/manifest.json

json{
  "name": "Brots · Educació i desenvolupament 0–6 anys",
  "short_name": "Brots",
  "description": "App gratuïta en català sobre educació i desenvolupament infantil de 0 a 6 anys.",
  "lang": "ca",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#FAFAF9",
  "theme_color": "#059669",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" },
    { "src": "/icons/icon-maskable-512.png", "sizes": "512x512", "type": "image/png", "purpose": "maskable" }
  ]
}


9. public/icons/icon.svg

L'icona base. Converteix-la a PNG (192, 512 i maskable 512) amb qualsevol conversor SVG→PNG. Per a la maskable, deixa més marge al voltant del brot (zona segura del 80%).

svg<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 512 512">
  <rect width="512" height="512" rx="96" fill="#059669"/>
  <g fill="none" stroke="#ffffff" stroke-width="28" stroke-linecap="round" stroke-linejoin="round">
    <path d="M256 400V232"/>
    <path d="M256 288c0-53-43-96-96-96h-32v16c0 53 43 96 96 96h32z"/>
    <path d="M256 232c0-53 43-96 96-96h32v16c0 53-43 96-96 96h-32z"/>
  </g>
</svg>


10. public/sw.js

Service worker. Estratègia: cache first per als recursos estàtics, network first per a la navegació.

jsconst CACHE = "brots-v1";
const ESSENTIALS = ["/", "/index.html", "/manifest.json"];

self.addEventListener("install", (e) => {
  e.waitUntil(caches.open(CACHE).then((c) => c.addAll(ESSENTIALS)));
  self.skipWaiting();
});

self.addEventListener("activate", (e) => {
  e.waitUntil(
    caches.keys().then((keys) =>
      Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener("fetch", (e) => {
  const { request } = e;
  if (request.method !== "GET") return;

  // Navegació: xarxa primer, cau si no hi ha connexió
  if (request.mode === "navigate") {
    e.respondWith(
      fetch(request).catch(() => caches.match("/index.html"))
    );
    return;
  }

  // Estàtics: cau primer
  e.respondWith(
    caches.match(request).then((hit) => {
      if (hit) return hit;
      return fetch(request).then((res) => {
        const copy = res.clone();
        caches.open(CACHE).then((c) => c.put(request, copy));
        return res;
      });
    })
  );
});


11. netlify.toml

toml[build]
  command = "npm run build"
  publish = "dist"

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200

[[headers]]
  for = "/sw.js"
  [headers.values]
    Cache-Control = "no-cache"


12. src/App.jsx — canvi de persistència

Agafa el codi de l'app tal com el tens i substitueix només les funcions load i save per aquestes. La resta del component no canvia.

js// ── PERSISTÈNCIA (localStorage) ────────────────────────────
function load(key) {
  try {
    const raw = localStorage.getItem(key);
    return raw ? JSON.parse(raw) : null;
  } catch {
    return null;
  }
}

function save(key, d) {
  try {
    localStorage.setItem(key, JSON.stringify(d));
  } catch {
    /* quota plena o mode privat: ignorem */
  }
}

I com que ara són síncrones, canvia també l'useEffect de càrrega dins de App():

jsuseEffect(() => {
  const p = load("brots-perfil");
  if (p) setPerfil(p);
  setCarregat(true);
}, []);

useEffect(() => {
  if (carregat) save("brots-perfil", perfil);
}, [perfil, carregat]);


Posar-ho en marxa

bashnpm install
npm run dev      # provar en local
npm run build    # generar /dist
