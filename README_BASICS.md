# Guía básica de la estructura del proyecto

Este repositorio es un punto de partida para crear una aplicación SaaS con **Next.js 14** y el enrutador de **App Router**. A continuación se explica con detalle cómo está organizado el código para que alguien que toma el proyecto por primera vez entienda dónde se ubica cada pieza y cómo se conectan.

## Carpetas en la raíz

| Carpeta/archivo | Propósito |
|-----------------|-----------|
| `src/` | Código fuente principal. Dentro se ubican rutas, componentes, utilidades y estilos. |
| `public/` | Archivos estáticos servidos tal cual (iconos, imágenes). |
| `migrations/` | Scripts SQL generados por Drizzle para versionar la base de datos. |
| `tests/` | Pruebas automatizadas (`e2e` con Playwright y `integration` con Vitest/Testing Library). |
| `next.config.mjs`, `tailwind.config.ts`, `tsconfig.json`, etc. | Archivos de configuración de Next.js, Tailwind y TypeScript. |

## Dentro de `src/`

### `app/` – Sistema de rutas
Next.js resuelve las rutas según la estructura de carpetas:

- **`[locale]`**: segmento dinámico que captura el idioma. Por ejemplo, `src/app/[locale]/(unauth)/page.tsx` se sirve como `/en` o `/fr` según el valor capturado.
- **Grupos de rutas**: las carpetas entre paréntesis **no** forman parte de la URL y sirven para organizar el código:
  - `(unauth)`: páginas públicas como la landing (`page.tsx`).
  - `(auth)`: páginas protegidas. Su `layout.tsx` envuelve el contenido con [`ClerkProvider`](https://clerk.com/) y ajusta las URL de inicio de sesión.
  - Dentro de `(auth)` existe otro grupo `(center)` que contiene las rutas `sign-in` y `sign-up` centradas en pantalla.
- **`page.tsx`**: archivo que representa la página para esa ruta.
- **`layout.tsx`**: archivo que define el marco visual y proveedores para todas las páginas anidadas. El layout global (`src/app/[locale]/layout.tsx`) carga estilos globales y provee `NextIntlClientProvider` para la internacionalización.
- **Ejemplo práctico**: `src/app/[locale]/(auth)/dashboard/page.tsx` aparece en el navegador como `/en/dashboard` y solo puede verse autenticado. Si un usuario sin sesión intenta entrar, `middleware.ts` lo redirige a `/sign-in`.
- **Catch‑all opcional** `[[...sign-in]]`/`[[...sign-up]]`: dentro de `src/app/[locale]/(auth)/(center)/sign-in/[[...sign-in]]` se permiten subrutas generadas por Clerk (por ejemplo `/en/sign-in/sso-callback`). Al ser opcional (`[[...]]`), la ruta básica `/en/sign-in` también es válida.

### `components/`
Componentes reutilizables y pequeños bloques de UI. Ejemplos:
- `ActiveLink.tsx`: enlace que sabe si está activo.
- `LocaleSwitcher.tsx`: selector de idioma.
- Carpeta `ui/`: contiene componentes atómicos como `button.tsx`, `input.tsx` o `accordion.tsx`, construidos sobre Tailwind y Shadcn.

### `features/`
Módulos agrupados por funcionalidad de negocio. Cada subcarpeta encapsula componentes, pruebas y lógica específica:
- `landing/`: secciones usadas en la página principal como `Hero`, `Features` o `PricingCard`.
- `dashboard/`: componentes que forman el panel del usuario (`MessageState`, `TitleBar`).
- `billing/`, `auth/`, `sponsors/`: otras funcionalidades aisladas.

### `hooks/`
Hooks personalizados de React. Por ejemplo `UseMenu.ts` maneja el estado de un menú desplegable y tiene su prueba asociada.

### `libs/`
Utilidades de infraestructura:
- `DB.ts`: inicializa la conexión con la base de datos mediante Drizzle ORM.
- `Env.ts`: valida variables de entorno.
- `i18n.ts` e `i18nNavigation.ts`: ayudan a configurar `next-intl`.
- `Logger.ts`: configura el logger basado en Pino.

### `locales/`
Archivos JSON con las traducciones (`en.json`, `fr.json`). Se cargan según el `locale` activo.

### `models/`
Definiciones del esquema de base de datos (`Schema.ts`) que Drizzle usa para generar migraciones.

### `styles/`
`global.css` contiene los estilos globales y las directivas de Tailwind.

### `templates/`
Bloques grandes pensados para reutilizarse en páginas (por ejemplo `Navbar`, `Footer`, `Pricing`, `Sponsors`). Cada uno compone varios componentes de menor nivel.

### `types/`
Tipos TypeScript compartidos, como `Auth.ts` o `Subscription.ts`.

### `utils/`
Funciones auxiliares y configuración general:
- `AppConfig.ts` define los idiomas soportados, el idioma por defecto y valores de la aplicación.
- `Helpers.ts` reúne utilidades varias (con pruebas en `Helpers.test.ts`).

### Otros archivos en `src/`
- `middleware.ts`: ejecuta antes de cada petición. Se encarga de resolver el idioma (usando `AppConfig`) y proteger rutas como `/dashboard` u `/api` con Clerk. Si un usuario no autenticado intenta acceder, es redirigido a la página de inicio de sesión.
- `instrumentation.ts`: registra la configuración de Sentry para capturar errores tanto en el runtime de Node.js como en Edge.

## Carpeta `tests/`
- `e2e/`: pruebas de extremo a extremo con Playwright.
- `integration/`: pruebas de integración o unitarias con Vitest y Testing Library.

## Carpeta `migrations/`
Scripts SQL generados automáticamente por Drizzle para versionar la base de datos. Cada archivo refleja un cambio incremental.

## Carpeta `public/`
Activos estáticos accesibles directamente desde el navegador: íconos (`favicon.ico`, `apple-touch-icon.png`) e imágenes para los ejemplos.

## Rutas dinámicas y grupos: `[]`, `()`, `[...slug]`, `[[...slug]]`

| Símbolo | Significado | Ejemplo en el proyecto |
|---------|-------------|-----------------------|
| `[param]` | Segmento dinámico obligatorio. El nombre se usa como clave en `params`. | `[locale]` captura `en` o `fr` y permite URLs como `/en/dashboard`. |
| `(grupo)` | *Route group* para organizar archivos sin afectar la URL. | `(auth)` agrupa todas las rutas que requieren autenticación; `(unauth)` contiene las públicas. |
| `[...param]` | *Catch‑all* obligatorio: captura uno o más segmentos. | No se usa directamente, pero serviría para rutas como `blog/[...slug]`. |
| `[[...param]]` | *Catch‑all* opcional: la ruta base o cualquier nivel adicional son válidos. | `sign-in/[[...sign-in]]` acepta `/sign-in` y subrutas de Clerk. |

### Idioma por defecto y prefijos
`AppConfig` define `defaultLocale: 'en'` y `localePrefix: 'as-needed'`. Esto significa que:
- Acceder a `/` carga la versión en inglés aunque no se ponga `/en` en la URL.
- Para otros idiomas sí se antepone el código: `/fr` para francés.

El middleware usa esos valores para redirigir y generar rutas correctas.

### Estado `auth` vs `unauth`
- Las rutas dentro de `(unauth)` están pensadas para usuarios anónimos (landing page).
- Las rutas en `(auth)` requieren sesión. El `middleware` comprueba con Clerk si hay usuario y, si no lo hay, redirige a `/sign-in`.
- Además, `AuthLayout` ajusta URLs y traducciones de Clerk según el idioma actual.

---
Con esta guía deberías poder orientarte en el código y saber dónde modificar o agregar funcionalidades según tus necesidades.
