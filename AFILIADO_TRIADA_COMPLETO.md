# PANEL DEL AFILIADO (DISTRIBUIDOR) — TRIADA Business School
## Documento técnico COMPLETO: arquitectura, pantallas, funciones y código fuente

Generado desde el código real del proyecto (no transcrito a mano): cada bloque de
código es el contenido literal del archivo indicado.

## ÍNDICE

- PARTE 1 — VISIÓN GENERAL (arquitectura, acceso, mapa de pantallas, flujos, límites)
- PARTE 2 — ACCESO, SESIÓN Y ARRANQUE
- PARTE 3 — SHELL, MENÚ Y NAVEGACIÓN
- PARTE 4 — LECTURA: CAPA DE DATOS DEL AFILIADO
- PARTE 5 — ESCRITURA: SERVER ACTIONS
- PARTE 6 — PANTALLAS DEL DISTRIBUIDOR
- PARTE 7 — COMPONENTES DEL DASHBOARD Y NEGOCIO
- PARTE 8 — TIENDA E-COMMERCE
- PARTE 9 — MI RED (VISOR INTERACTIVO)
- PARTE 10 — HUB TRIADA (CENTRO DE IA)
- PARTE 11 — HUB: SERVIDOR (RUTAS API Y LIBRERÍAS)
- PARTE 12 — DOMINIO Y MOTOR QUE TOCA AL AFILIADO
- PARTE 13 — BASE DE DATOS

---

# PARTE 1 — VISIÓN GENERAL

## 1.1 Qué es

El panel del afiliado es la zona que ve cada miembro de TRIADA al iniciar sesión: su
dashboard, su organización (árbol binario), sus comisiones, billetera y retiros, su rango,
la tienda con carrito, el **Hub Triada** (centro de IA por créditos), sus proyectos
generados, las academias y su perfil. Rutas: grupo `app/(distribuidor)/`. El registro y el
login viven en `app/(auth)/` y la activación inicial en `app/(onboarding)/`.

## 1.2 Stack

- Next.js 14 (App Router) + TypeScript + Tailwind CSS
- Supabase: PostgreSQL + Auth + Storage + Row Level Security
- **Server Components** para LEER, **Server Actions** para ESCRIBIR; rutas `/api/ia/*` solo para IA (streaming y subida de archivos)
- OpenRouter como puerta única a 380 modelos de IA (la llave vive solo en el servidor)
- `reactflow` (visor de Mi Red), `react-markdown` + `remark-gfm` (respuestas del chat), lucide-react
- Tema día/noche con variables CSS e i18n propio ES/EN/PT (el afiliado ve todo traducido)

## 1.3 Acceso: quién entra y a dónde va

```
Sin sesión .............................. /login
Con sesión, sin compra (has_position=false) → /bienvenida (elige y activa un programa)
Con sesión y posición ................... /dashboard y el resto del panel
Rol is_admin ............................ además /admin
```

El layout `app/(distribuidor)/layout.tsx` hace `requireUser()` y, si no tiene posición, redirige
a `/bienvenida`. Calcula también el puntaje del servicio activo más alto para mostrar u ocultar
las academias en el menú.

## 1.4 Las 3 llaves (lo que decide qué ve cada afiliado)

Tres interruptores INDEPENDIENTES en `users`:

| Llave | Se activa cuando | Qué habilita |
|---|---|---|
| `has_position` | compró cualquier producto/programa | entrar al panel, generar puntos |
| `can_distribute` | IBO activo y vigente | link de referido, "Centro de Negocio" (Mi Red, Comisiones, Rango, Billetera, Herramientas), dashboard de distribuidor |
| `can_earn` | servicio ≥ 100 pts (Nova o Quantum) **y** can_distribute | cobrar comisiones de red |

Un afiliado SIN IBO ve un dashboard de **estudiante** simple; con IBO ve el de **distribuidor**.
El menú lateral se arma dinámicamente con estas llaves.

## 1.5 Mapa de pantallas

| Ruta | Función | Datos | Escritura |
|---|---|---|---|
| `/dashboard` | Dashboard de distribuidor o de estudiante según IBO: perfil, billetera, rango con insignia, gráfico de ingresos (Año/Mes/Día), rendimiento del equipo, ganancias por vía, cuenta regresiva de renovación, link de invitación, retiros, nuevos miembros | `getDashboardData`, `getStudentDashboard`, `getPendingReferrals` | — |
| `/programas` | Programas activos del afiliado y su vigencia | `getPrograms` | — |
| `/tienda` | **E-commerce**: catálogo con imágenes, filtros, buscador, carrito lateral, cupones, pago simulado; y tienda de recarga de créditos de IA | `getStoreCatalog`, `getCreditPacks` | `checkoutAction`, `validateCouponAction`, `buyCreditsAction` |
| `/pedidos` | Historial de pedidos con ítems, descuento y estado | `getMyStoreOrders` | — |
| `/ia` — **Hub Triada** | Centro de IA con pestañas: Chat, Agentes, Código, Imágenes, Editor, Audio, Video (próximamente), Super Prompter, Tendencias | `loadAiModels`, `getActivity`… | `/api/ia/chat`, `/imagen`, `/audio`, `/superprompt`, `/upload` |
| `/proyectos` | **Mis Proyectos**: todo lo generado (imagen/audio/video) con detalle a pantalla completa y opción de publicar en Tendencias | `getCreations` | `toggleCreationPublicAction` |
| `/red` | **Mi Red**: visor interactivo ReactFlow, binario y unilevel (5 niveles), plegar ramas y colocar referidos pendientes | `getStructureData`, `getPendingReferrals` | `placeMemberAction` |
| `/comisiones` | Ganancias detalladas por vía y período | `getEarningsDetailed` | — |
| `/rango` | Rango actual y progreso de las 4 condiciones hacia el siguiente | `getRankData` | — |
| `/billetera` | Saldo, direcciones USDT, historial y retiros (exige KYC) | `getWalletData`, `getWalletAddresses` | `requestWithdrawalAction`, `saveWalletAddressAction` |
| `/herramientas` | Calculadoras del plan (proyección de ingresos) | `loadEngineConfig` | — |
| `/academia/nova`, `/academia/quantum` | Contenido formativo, protegido por servicio activo (≥ 50 / ≥ 100 pts) | `requireService` | — |
| `/perfil` | Datos personales, foto de perfil, cambio de contraseña | `users` | `updateProfileAction`, `changePasswordAction` |
| `/bienvenida` | Primera activación: elegir programa (cuenta sin posición) | `getCatalog` | `simulatePurchaseAction` |
| `/login`, `/registro?ref=CODIGO` | Acceso y alta con link de referido | — | `loginAction`, `registerAction` |

Rutas que solo redirigen a `/ia` (la unificación del Hub): `/ia/asistentes`, `/creativo`,
`/creativo/imagen`, `/test-abacus`.

## 1.6 Patrón de trabajo

```
LEER:     page.tsx (Server Component async) → requireUser() → lib/data/distributor.ts → render
ESCRIBIR: <form action={miAction}> o llamada desde un componente cliente → Server Action:
            1) requireUser()  2) validar en servidor  3) escribir  4) revalidatePath()
```

Regla de seguridad: **el navegador nunca decide precios, puntos ni saldos.** El checkout recibe
solo `{productId, qty}` y relee precio, puntos, tipo y cupón en el servidor.

## 1.7 Flujos de negocio clave

**Registro con link de referido** (`registerDistributor`): valida el código del patrocinador
(debe poder distribuir) → calcula la posición binaria según la pierna del link (spillover) → crea
la cuenta de auth → crea el perfil (el `sponsor_id` puede diferir del `placement_id`) → crea la
billetera y un código de referido único de 8 caracteres. El nuevo miembro queda **pendiente de
colocar**: su patrocinador lo ubica después desde Mi Red.

**Activación / compra** (`simulateActivation`): crea la orden pagada → crea la membresía →
recalcula las 3 llaves → otorga créditos de IA del paquete → bono de afiliación en créditos al
patrocinador (10 % en activación, 5 % en renovación) → comisiones directas y volumen al binario.

**Tienda** (`checkoutAction`): el pedido se guarda en `store_orders` y por cada unidad se llama
`simulateActivation`, que es el flujo de siempre. La tabla `orders` (que lee el motor de
comisiones) no se modificó, así el carrito no altera puntos, binario ni rangos.

**Colocar un referido** (`placeMemberAction`): solo puedes colocar a TUS referidos directos y solo
dentro de TU subárbol binario; las piernas libres muestran el botón "+".

**Retiro** (`requestWithdrawalAction`): exige KYC aprobado, respeta el mínimo configurado y el
saldo; reserva (descuenta) el saldo y crea la solicitud *pending*. El admin la aprueba/paga/rechaza.

**Hub de IA y créditos**: $1 = 1.000 créditos, costo FIJO por tarea (no por token). Dos modos:
**Triada Auto Max** (un router decide el modelo según lo que pides) o elegir tú el modelo del
catálogo (cada uno con su costo). El saldo se cobra antes de llamar al proveedor y se **reembolsa**
si falla. Créditos mensuales por paquete: Básico 2.500 · Estándar 7.500 · Nova 20.000 · Quantum 35.000.
Recargas ($10→10.000, $20→22.000, $50→60.000, $100→130.000) no expiran y dan 10 % en créditos al patrocinador.

**Rango**: 13 niveles. Se evalúan 4 condiciones en orden: puntos por pierna, % máximo por línea,
directos activos por pierna y `can_earn`. La pierna mayor arrastra (carry-over).

## 1.8 Cuentas de prueba

Se crean con `node scripts/bootstrap-users.mjs` (correo y clave de prueba en ese script).
`scripts/seed-red-demo.mjs` siembra una red demo de 5 niveles con rangos calculados por el motor real.

## 1.9 Límites conocidos (antes de producción)

- **`simulatePurchaseAction` y `buyCreditsAction` no tienen compuerta de pago:** cualquier usuario con sesión puede activar un programa o recargar créditos sin pagar (la pasarela USDT real no está conectada). Deben gatearse o eliminarse antes de producción.
- El retiro lee el saldo y luego lo actualiza en dos pasos (no atómico): puede haber condición de carrera con solicitudes simultáneas.
- Imágenes y audio requieren saldo en OpenRouter (hoy sin saldo devuelven 402). **Video no existe en OpenRouter**: queda "Próximamente" hasta conectar fal.ai u otro.
- La condición 2 del rango (máx % por línea) está fijada en 0 en el cierre mensual.
- Subida de avatar sin validar tipo/tamaño; cambio de contraseña sin pedir la actual; sin límite de intentos de login.
- El reinicio mensual de créditos de IA por paquete aún no tiene cron.
- El visor de Mi Red no incluye aún la colocación por arrastre (solo por botón "+").


---

# PARTE 2 — ACCESO, SESIÓN Y ARRANQUE
Cómo se identifica al afiliado y cómo se protege cada página.

### 1. `middleware.ts`
Refresca la sesión de Supabase en cada navegación.
```ts
import { type NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/middleware";

/** Refresca la sesión de Supabase en cada navegación. */
export async function middleware(request: NextRequest) {
  return await updateSession(request);
}

export const config = {
  // Excluye assets estáticos para no correr el middleware innecesariamente.
  matcher: [
    "/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)",
  ],
};
```

### 2. `lib/supabase/middleware.ts`
Lógica de `updateSession`: lee y renueva las cookies de sesión.
```ts
import { createServerClient, type CookieOptions } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

type CookieToSet = { name: string; value: string; options: CookieOptions };

/**
 * Refresca la sesión de Supabase en cada request (patrón oficial SSR).
 * Se invoca desde `middleware.ts`. Mantiene viva la cookie de sesión.
 */
export async function updateSession(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet: CookieToSet[]) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // IMPORTANTE: getUser() revalida el token contra Supabase (no confiar en getSession en server).
  await supabase.auth.getUser();

  return response;
}
```

### 3. `lib/supabase/server.ts`
Cliente de Supabase para Server Components y acciones (usa la sesión del usuario, respeta RLS).
```ts
import { createServerClient, type CookieOptions } from "@supabase/ssr";
import { cookies } from "next/headers";

type CookieToSet = { name: string; value: string; options: CookieOptions };

/**
 * Cliente de Supabase para el SERVIDOR (Server Components, Route Handlers,
 * Server Actions). Usa la clave anónima + la sesión del usuario vía cookies,
 * por lo que respeta RLS en nombre del usuario autenticado.
 */
export function createClient() {
  const cookieStore = cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet: CookieToSet[]) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // El método set falla en Server Components puros (solo lectura).
            // Es seguro ignorarlo si el refresco de sesión lo maneja un middleware.
          }
        },
      },
    }
  );
}
```

### 4. `lib/supabase/client.ts`
Cliente de Supabase para el navegador.
```ts
import { createBrowserClient } from "@supabase/ssr";

/**
 * Cliente de Supabase para el NAVEGADOR (componentes 'use client').
 * Usa la clave anónima pública y respeta Row Level Security (RLS).
 * Nunca expongas aquí la service_role key.
 */
export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

### 5. `lib/supabase/admin.ts`
Cliente `service_role`: salta RLS. SOLO servidor.
```ts
import { createClient } from "@supabase/supabase-js";

/**
 * Cliente ADMIN de Supabase (service_role).
 *
 * ⚠️ CRÍTICO: salta Row Level Security. SOLO usar en el servidor
 * (motor de comisiones, cierres mensuales, operaciones administrativas).
 * NUNCA importar desde un componente 'use client' ni exponer la llave al navegador.
 *
 * El motor de comisiones (Fase C) corre con este cliente porque necesita
 * leer/escribir el árbol completo y todos los ledgers sin restricción de usuario.
 */
export function createAdminClient() {
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const serviceKey = process.env.SUPABASE_SERVICE_ROLE_KEY;

  if (!url || !serviceKey) {
    throw new Error(
      "Faltan NEXT_PUBLIC_SUPABASE_URL o SUPABASE_SERVICE_ROLE_KEY en el entorno."
    );
  }

  return createClient(url, serviceKey, {
    auth: {
      autoRefreshToken: false,
      persistSession: false,
    },
  });
}
```

### 6. `lib/auth/session.ts`
`getCurrentUser`, `requireUser` (→ /login) y `requireAdmin`.
```ts
/**
 * Helpers de sesión y rol para el servidor.
 * Une la sesión de Supabase Auth con el perfil de `public.users`.
 */
import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import type { User } from "@/types/database";

/**
 * Devuelve el perfil del usuario autenticado (o null si no hay sesión).
 * Usa getUser() (revalida contra Supabase, seguro en servidor).
 */
export async function getCurrentUser(): Promise<User | null> {
  const supabase = createClient();

  const {
    data: { user: authUser },
  } = await supabase.auth.getUser();
  if (!authUser) return null;

  const { data: profile } = await supabase
    .from("users")
    .select("*")
    .eq("id", authUser.id)
    .maybeSingle();

  return (profile as User) ?? null;
}

/** Exige sesión; si no hay, redirige a /login. Devuelve el perfil. */
export async function requireUser(): Promise<User> {
  const user = await getCurrentUser();
  if (!user) redirect("/login");
  return user;
}

/** Exige rol admin; si no, redirige. Devuelve el perfil admin. */
export async function requireAdmin(): Promise<User> {
  const user = await requireUser();
  if (!user.is_admin) redirect("/dashboard");
  return user;
}
```

### 7. `lib/auth/guards.ts`
`requireService(minPoints)`: compuerta por servicio activo (Academia Nova ≥ 50, Quantum ≥ 100; si no, redirige a /tienda).
```ts
/**
 * Compuertas de acceso por activación (servicio del usuario).
 * ⚠️ SOLO SERVIDOR.
 */
import { redirect } from "next/navigation";
import { requireUser } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";

/** Puntos del servicio activo de mayor valor del usuario actual. */
export async function maxActiveServicePoints(userId: string): Promise<number> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("memberships")
    .select("points")
    .eq("user_id", userId)
    .eq("type", "service")
    .eq("status", "active");
  return (data ?? []).reduce((m, x) => Math.max(m, x.points ?? 0), 0);
}

/** Exige que el usuario tenga un servicio activo de al menos `minPoints`. */
export async function requireService(minPoints: number) {
  const user = await requireUser();
  const pts = await maxActiveServicePoints(user.id);
  if (pts < minPoints) redirect("/tienda");
  return user;
}
```

### 8. `lib/auth/password.ts`
Utilidades de validación de contraseña.
```ts
/**
 * Reglas de contraseña de TRIADA (mismas en cliente y servidor).
 */
export interface PasswordRule {
  label: string;
  ok: (pw: string) => boolean;
}

export const PASSWORD_RULES: PasswordRule[] = [
  { label: "Al menos 9 caracteres", ok: (p) => p.length >= 9 },
  { label: "Menos de 16 caracteres", ok: (p) => p.length < 16 },
  { label: "Una mayúscula (A-Z)", ok: (p) => /[A-Z]/.test(p) },
  { label: "Una minúscula (a-z)", ok: (p) => /[a-z]/.test(p) },
  { label: "Un número (0-9)", ok: (p) => /[0-9]/.test(p) },
  {
    label: "Un carácter especial (!@#$%^&*.?\\:{}<>)",
    ok: (p) => /[!@#$%^&*()\.\?\\:\{\}<>\[\]\-_+=/|~`'";,]/.test(p),
  },
];

/** Evalúa cada regla. */
export function checkPassword(pw: string): { label: string; ok: boolean }[] {
  return PASSWORD_RULES.map((r) => ({ label: r.label, ok: r.ok(pw) }));
}

/** ¿Cumple todas las reglas? */
export function isPasswordValid(pw: string): boolean {
  return PASSWORD_RULES.every((r) => r.ok(pw));
}
```

### 9. `app/layout.tsx`
Layout raíz: fuentes, tema día/noche y metadatos.
```tsx
import type { Metadata } from "next";
import { Inter, Montserrat } from "next/font/google";
import "./globals.css";

// Tipografía oficial TRIADA: Montserrat (display) + Inter (cuerpo).
const inter = Inter({
  subsets: ["latin"],
  variable: "--font-inter",
  display: "swap",
});

const montserrat = Montserrat({
  subsets: ["latin"],
  weight: ["600", "700", "800"],
  variable: "--font-montserrat",
  display: "swap",
});

export const metadata: Metadata = {
  title: "TRIADA Business School — Back Office",
  description: "Plataforma de distribución TRIADA. APRENDE. CONSTRUYE. PROTEGE.",
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="es" className={`${inter.variable} ${montserrat.variable}`}>
      <head>
        {/* Fija el tema antes de pintar (evita parpadeo). */}
        <script
          dangerouslySetInnerHTML={{
            __html: `try{if(localStorage.getItem('triada-theme')==='light')document.documentElement.setAttribute('data-theme','light');}catch(e){}`,
          }}
        />
      </head>
      <body>{children}</body>
    </html>
  );
}
```

### 10. `app/(auth)/layout.tsx`
Layout de las pantallas de acceso (fondo con circuito animado).
```tsx
import Link from "next/link";
import { Wordmark } from "@/components/ui/Wordmark";
import { CircuitBackground } from "@/components/ui/CircuitBackground";
import { LanguageProvider } from "@/components/i18n/LanguageProvider";
import { LanguageSwitcher } from "@/components/i18n/LanguageSwitcher";
import { getLocale, getT } from "@/lib/i18n/server";
import { messages } from "@/lib/i18n/messages";

/** Layout de las páginas de autenticación (vitrina → con circuito animado). */
export default function AuthLayout({ children }: { children: React.ReactNode }) {
  const locale = getLocale();
  const t = getT();
  return (
    <LanguageProvider locale={locale} dict={messages[locale]}>
      <main className="relative isolate flex min-h-screen items-center justify-center overflow-hidden px-6 py-16">
        <CircuitBackground />
        <div className="absolute right-4 top-4 z-20">
          <LanguageSwitcher />
        </div>
        <div className="relative z-10 w-full max-w-md">
          <Link href="/" className="mb-8 flex justify-center">
            <Wordmark />
          </Link>
          {children}
          <p className="mt-10 text-center text-[11px] uppercase tracking-label text-muted">
            {t("common.tagline")}
          </p>
        </div>
      </main>
    </LanguageProvider>
  );
}
```

### 11. `app/(auth)/login/page.tsx`
Pantalla de inicio de sesión.
```tsx
import Link from "next/link";
import { Card } from "@/components/ui/Card";
import { LoginForm } from "@/components/auth/LoginForm";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Iniciar sesión — TRIADA" };

export default function LoginPage() {
  const t = getT();
  return (
    <Card premium>
      <h1 className="text-2xl font-extrabold text-cream">{t("login.title")}</h1>
      <p className="mt-1 mb-6 text-sm text-muted">{t("login.subtitle")}</p>
      <LoginForm />
      <p className="mt-6 text-center text-sm text-muted">
        {t("login.noAccount")}{" "}
        <Link href="/registro" className="text-gold hover:text-goldLight">
          {t("login.registerLink")}
        </Link>
      </p>
    </Card>
  );
}
```

### 12. `app/(auth)/registro/page.tsx`
Alta con link de referido: `?ref=CODIGO`.
```tsx
import Link from "next/link";
import { Card } from "@/components/ui/Card";
import { RegisterForm } from "@/components/auth/RegisterForm";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Crear cuenta — TRIADA" };

/**
 * Registro con link de referido (un solo link: ?ref=CODIGO).
 * El nuevo miembro queda con patrocinador pero sin posición (se coloca después).
 */
export default function RegistroPage({
  searchParams,
}: {
  searchParams: { ref?: string };
}) {
  const refCode = (searchParams.ref ?? "").toUpperCase();
  const t = getT();

  return (
    <Card premium>
      <h1 className="text-2xl font-extrabold text-cream">{t("reg.title")}</h1>
      <p className="mt-1 mb-6 text-sm text-muted">{t("reg.subtitle")}</p>
      <RegisterForm refCode={refCode} />
      <p className="mt-6 text-center text-sm text-muted">
        {t("reg.haveAccount")}{" "}
        <Link href="/login" className="text-gold hover:text-goldLight">
          {t("reg.loginLink")}
        </Link>
      </p>
    </Card>
  );
}
```

### 13. `app/(auth)/actions.ts`
`loginAction`, `registerAction` y `logoutAction`.
```ts
"use server";

import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";
import { registerDistributor } from "@/lib/domain/registration";
import { isPasswordValid } from "@/lib/auth/password";

export interface FormState {
  error?: string;
}

/** Inicia sesión. Redirige a /admin o /dashboard según el rol. */
export async function loginAction(
  _prev: FormState,
  formData: FormData
): Promise<FormState> {
  const email = String(formData.get("email") ?? "").trim();
  const password = String(formData.get("password") ?? "");

  if (!email || !password) return { error: "Correo y contraseña son obligatorios." };

  const supabase = createClient();
  const { data, error } = await supabase.auth.signInWithPassword({ email, password });
  if (error) return { error: "Correo o contraseña incorrectos." };

  // ¿Es admin? Para decidir a dónde mandarlo.
  const { data: profile } = await supabase
    .from("users")
    .select("is_admin")
    .eq("id", data.user.id)
    .maybeSingle();

  redirect(profile?.is_admin ? "/admin" : "/dashboard");
}

/** Registra un nuevo distribuidor con el link de referido y lo inicia sesión. */
export async function registerAction(
  _prev: FormState,
  formData: FormData
): Promise<FormState> {
  const firstName = String(formData.get("firstName") ?? "").trim();
  const lastName = String(formData.get("lastName") ?? "").trim();
  const email = String(formData.get("email") ?? "").trim();
  const password = String(formData.get("password") ?? "");
  const country = String(formData.get("country") ?? "").trim();
  const dialCode = String(formData.get("dialCode") ?? "").trim();
  const phoneNumber = String(formData.get("phoneNumber") ?? "").trim();
  const birthDate = String(formData.get("birthDate") ?? "").trim();
  const city = String(formData.get("city") ?? "").trim();
  const postalCode = String(formData.get("postalCode") ?? "").trim();
  const gender = String(formData.get("gender") ?? "").trim();
  const address = String(formData.get("address") ?? "").trim();
  const refCode = String(formData.get("refCode") ?? "").trim();

  const fullName = `${firstName} ${lastName}`.trim();
  const phone = phoneNumber ? `${dialCode} ${phoneNumber}`.trim() : "";

  // Validación: todos los campos son obligatorios.
  if (!firstName || !lastName) return { error: "Nombre y apellido son obligatorios." };
  if (!email || !password) return { error: "Correo y contraseña son obligatorios." };
  if (!isPasswordValid(password)) return { error: "La contraseña no cumple los requisitos." };
  if (!country) return { error: "Selecciona tu país." };
  if (!gender) return { error: "Selecciona tu género." };
  if (!phoneNumber) return { error: "Ingresa tu número de teléfono." };
  if (!birthDate) return { error: "Ingresa tu fecha de nacimiento." };
  if (!address) return { error: "Ingresa tu dirección." };
  if (!city) return { error: "Ingresa tu ciudad." };
  if (!postalCode) return { error: "Ingresa tu código postal." };

  try {
    await registerDistributor({
      fullName,
      email,
      password,
      country,
      phone,
      birthDate,
      city,
      postalCode,
      gender,
      address,
      refCode: refCode || undefined,
    });
  } catch (err) {
    return { error: err instanceof Error ? err.message : "No se pudo completar el registro." };
  }

  // Iniciar sesión automáticamente y mandar a elegir paquete (debe comprar para entrar).
  const supabase = createClient();
  await supabase.auth.signInWithPassword({ email, password });
  redirect("/bienvenida");
}

/** Cierra la sesión. */
export async function logoutAction() {
  const supabase = createClient();
  await supabase.auth.signOut();
  redirect("/login");
}
```

### 14. `components/auth/LoginForm.tsx`
Formulario de login con manejo de errores.
```tsx
"use client";

import { useFormState, useFormStatus } from "react-dom";
import { loginAction, type FormState } from "@/app/(auth)/actions";
import { Field } from "@/components/ui/Input";
import { Button } from "@/components/ui/Button";
import { useT } from "@/components/i18n/LanguageProvider";

const initial: FormState = {};

function SubmitButton() {
  const { pending } = useFormStatus();
  const { t } = useT();
  return (
    <Button type="submit" className="w-full" disabled={pending}>
      {pending ? t("login.submitting") : t("login.submit")}
    </Button>
  );
}

export function LoginForm() {
  const { t } = useT();
  const [state, formAction] = useFormState(loginAction, initial);

  return (
    <form action={formAction} className="space-y-4">
      <Field label={t("login.email")} name="email" type="email" autoComplete="email" required />
      <Field
        label={t("login.password")}
        name="password"
        type="password"
        autoComplete="current-password"
        required
      />
      {state.error && (
        <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">
          {state.error}
        </p>
      )}
      <SubmitButton />
    </form>
  );
}
```

### 15. `components/auth/RegisterForm.tsx`
Formulario de registro con país, género y código de patrocinador.
```tsx
"use client";

import { useState } from "react";
import { useFormState, useFormStatus } from "react-dom";
import { Check, X } from "lucide-react";
import { registerAction, type FormState } from "@/app/(auth)/actions";
import { Field } from "@/components/ui/Input";
import { Button } from "@/components/ui/Button";
import { COUNTRIES } from "@/lib/data/countries";
import { checkPassword } from "@/lib/auth/password";
import { useT } from "@/components/i18n/LanguageProvider";

const initial: FormState = {};
const PWD_KEYS = ["pwd.min9", "pwd.max16", "pwd.upper", "pwd.lower", "pwd.number", "pwd.special"];

function SubmitButton() {
  const { pending } = useFormStatus();
  const { t } = useT();
  return (
    <Button type="submit" className="w-full" disabled={pending}>
      {pending ? t("reg.submitting") : t("reg.submit")}
    </Button>
  );
}

const selectCls =
  "w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream outline-none focus:border-gold";

export function RegisterForm({ refCode }: { refCode: string }) {
  const { t } = useT();
  const [state, formAction] = useFormState(registerAction, initial);
  const [country, setCountry] = useState("");
  const [password, setPassword] = useState("");

  const dial = COUNTRIES.find((c) => c.name === country)?.dial ?? "";
  const rules = checkPassword(password);

  return (
    <form action={formAction} className="space-y-4">
      <input type="hidden" name="refCode" value={refCode} />
      <input type="hidden" name="dialCode" value={dial} />

      {refCode ? (
        <p className="rounded-md border border-border bg-navyDeep px-3 py-2 text-xs text-muted">
          {t("reg.byInvite")}
        </p>
      ) : (
        <p className="rounded-md border border-gold/30 bg-gold/10 px-3 py-2 text-xs text-gold">
          {t("reg.rootReg")}
        </p>
      )}

      <div className="grid grid-cols-2 gap-4">
        <Field label={t("reg.firstName")} name="firstName" autoComplete="given-name" required />
        <Field label={t("reg.lastName")} name="lastName" autoComplete="family-name" required />
      </div>

      <Field label={t("reg.email")} name="email" type="email" autoComplete="email" required />

      <label className="block">
        <span className="mb-1.5 block font-sans text-xs font-medium uppercase tracking-[0.08em] text-muted">
          {t("reg.password")}
        </span>
        <input
          name="password"
          type="password"
          autoComplete="new-password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
          className={selectCls}
        />
        {password.length > 0 && (
          <ul className="mt-2 space-y-1 rounded-md border border-border bg-navyDeep p-2">
            {rules.map((r, i) => (
              <li key={i} className="flex items-center gap-2 text-[11px]">
                {r.ok ? <Check size={12} className="text-success" /> : <X size={12} className="text-danger" />}
                <span className={r.ok ? "text-success" : "text-muted"}>{t(PWD_KEYS[i])}</span>
              </li>
            ))}
          </ul>
        )}
      </label>

      <div className="grid grid-cols-2 gap-4">
        <label className="block">
          <span className="mb-1.5 block font-sans text-xs font-medium uppercase tracking-[0.08em] text-muted">
            {t("reg.country")}
          </span>
          <select name="country" value={country} onChange={(e) => setCountry(e.target.value)} required className={selectCls}>
            <option value="" disabled>{t("reg.select")}</option>
            {COUNTRIES.map((c) => (
              <option key={c.name} value={c.name}>{c.flag} {c.name}</option>
            ))}
          </select>
        </label>
        <label className="block">
          <span className="mb-1.5 block font-sans text-xs font-medium uppercase tracking-[0.08em] text-muted">
            {t("reg.gender")}
          </span>
          <select name="gender" defaultValue="" required className={selectCls}>
            <option value="" disabled>{t("reg.select")}</option>
            <option value="M">{t("reg.genderM")}</option>
            <option value="F">{t("reg.genderF")}</option>
            <option value="X">{t("reg.genderX")}</option>
          </select>
        </label>
      </div>

      <label className="block">
        <span className="mb-1.5 block font-sans text-xs font-medium uppercase tracking-[0.08em] text-muted">
          {t("reg.phone")}
        </span>
        <div className="flex gap-2">
          <span className="flex min-w-[64px] items-center justify-center rounded-md border border-border bg-navyDeep px-3 text-sm text-gold">
            {dial || "+__"}
          </span>
          <input
            name="phoneNumber"
            inputMode="tel"
            placeholder={t("reg.phonePh")}
            required
            className="w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream placeholder:text-muted/60 outline-none focus:border-gold"
          />
        </div>
      </label>

      <Field label={t("reg.birthDate")} name="birthDate" type="date" required />
      <Field label={t("reg.address")} name="address" placeholder={t("reg.addressPh")} required />

      <div className="grid grid-cols-2 gap-4">
        <Field label={t("reg.city")} name="city" required />
        <Field label={t("reg.postalCode")} name="postalCode" required />
      </div>

      {state.error && (
        <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">
          {state.error}
        </p>
      )}
      <SubmitButton />
    </form>
  );
}
```

### 16. `lib/domain/registration.ts`
Registro de distribuidores: valida patrocinador, calcula posición con spillover, crea cuenta, perfil, billetera y código único.
```ts
/**
 * REGISTRO DE DISTRIBUIDORES (con link de referido).
 *
 * ⚠️ SOLO SERVIDOR (usa el cliente admin: crea la cuenta de auth y el perfil).
 *
 * Flujo:
 *  1. Valida el código de referido → patrocinador (debe poder distribuir).
 *  2. Calcula la posición binaria según la pierna del link (spillover).
 *  3. Crea la cuenta de auth (confirmada al instante — sin correo, decisión MVP).
 *  4. Crea el perfil en `users` (sponsor_id ≠ placement_id si hubo spillover).
 *  5. Crea su billetera y su propio código de referido único.
 *
 * Bootstrap: el PRIMER usuario del sistema (raíz) puede registrarse sin código.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { findAutoPlacementSlot } from "./placement";
import type { BinaryLeg } from "@/types/database";

export interface RegisterInput {
  email: string;
  password: string;
  fullName: string;
  country?: string;
  phone?: string;
  birthDate?: string; // 'YYYY-MM-DD'
  city?: string;
  postalCode?: string;
  gender?: string; // 'M' | 'F' | 'X'
  address?: string;
  refCode?: string; // código del patrocinador (vacío solo para el usuario raíz)
  leg?: BinaryLeg; // pierna elegida por el link (izq/der)
  isAdmin?: boolean; // SOLO desde scripts internos, nunca desde el form público
  /**
   * Colocación binaria EXPLÍCITA (admin): ubica al miembro exactamente en este
   * placement_id + pierna, sin spillover. El patrocinio (sponsor) sigue al refCode.
   */
  explicitPlacement?: { placementId: string; leg: BinaryLeg };
}

export interface RegisterResult {
  userId: string;
  referralCode: string;
  placementId: string | null;
  binaryLeg: BinaryLeg | null;
}

/** Genera un código de referido único de 8 caracteres (sin caracteres ambiguos). */
async function generateUniqueReferralCode(): Promise<string> {
  const supabase = createAdminClient();
  const alphabet = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789"; // sin O/0/I/1/L

  for (let attempt = 0; attempt < 20; attempt++) {
    let code = "";
    for (let i = 0; i < 8; i++) {
      code += alphabet[Math.floor(Math.random() * alphabet.length)];
    }
    const { data } = await supabase
      .from("users")
      .select("id")
      .eq("referral_code", code)
      .maybeSingle();
    if (!data) return code;
  }
  throw new Error("No se pudo generar un código de referido único.");
}

export async function registerDistributor(
  input: RegisterInput
): Promise<RegisterResult> {
  const supabase = createAdminClient();

  // --- 1. Resolver patrocinador y posición binaria ---
  let sponsorId: string | null = null;
  let placementId: string | null = null;
  let binaryLeg: BinaryLeg | null = null;

  if (input.explicitPlacement) {
    // --- Colocación manual del admin: posición exacta, sin spillover ---
    const { placementId: pid, leg } = input.explicitPlacement;

    // El slot debe estar libre.
    const { data: occupied } = await supabase
      .from("users")
      .select("id")
      .eq("placement_id", pid)
      .eq("binary_leg", leg)
      .maybeSingle();
    if (occupied) throw new Error("Esa posición del árbol ya está ocupada.");

    placementId = pid;
    binaryLeg = leg;

    // Patrocinio: del refCode si se indicó; si no, el nodo padre patrocina.
    if (input.refCode) {
      const { data: sponsor } = await supabase
        .from("users")
        .select("id")
        .eq("referral_code", input.refCode.toUpperCase())
        .maybeSingle();
      if (!sponsor) throw new Error("El código de patrocinador no existe.");
      sponsorId = sponsor.id;
    } else {
      sponsorId = pid;
    }
  } else if (input.refCode) {
    const { data: sponsor, error: sponsorErr } = await supabase
      .from("users")
      .select("id, can_distribute")
      .eq("referral_code", input.refCode.toUpperCase())
      .maybeSingle();

    if (sponsorErr) throw new Error(`Error validando referido: ${sponsorErr.message}`);
    if (!sponsor) throw new Error("El código de referido no existe.");

    // CUALQUIER usuario puede referir (incluido el estudiante, para su "3 & Gratis").
    sponsorId = sponsor.id;

    if (sponsor.can_distribute) {
      // Patrocinador DISTRIBUIDOR (con IBO): queda PENDIENTE; lo coloca a mano.
      placementId = null;
      binaryLeg = null;
    } else {
      // Patrocinador ESTUDIANTE (sin IBO): colocación AUTOMÁTICA por spillover.
      const slot = await findAutoPlacementSlot(sponsor.id);
      placementId = slot.placementId;
      binaryLeg = slot.leg;
    }
  } else {
    // Bootstrap: solo permitido si NO existe ningún usuario todavía (raíz).
    const { count } = await supabase
      .from("users")
      .select("*", { count: "exact", head: true });
    if ((count ?? 0) > 0) {
      throw new Error("Se requiere un link de referido para registrarse.");
    }
  }

  // --- 2. Crear la cuenta de auth (confirmada al instante) ---
  const { data: authData, error: authErr } = await supabase.auth.admin.createUser({
    email: input.email,
    password: input.password,
    email_confirm: true,
    user_metadata: { full_name: input.fullName },
  });
  if (authErr || !authData.user) {
    throw new Error(`No se pudo crear la cuenta: ${authErr?.message ?? "desconocido"}`);
  }
  const userId = authData.user.id;

  try {
    // --- 3. Crear el perfil en public.users ---
    const referralCode = await generateUniqueReferralCode();

    const { error: userErr } = await supabase.from("users").insert({
      id: userId,
      email: input.email,
      full_name: input.fullName,
      country: input.country ?? null,
      phone: input.phone ?? null,
      birth_date: input.birthDate ?? null,
      city: input.city ?? null,
      postal_code: input.postalCode ?? null,
      gender: input.gender ?? null,
      address: input.address ?? null,
      sponsor_id: sponsorId,
      placement_id: placementId,
      binary_leg: binaryLeg,
      referral_code: referralCode,
      is_admin: input.isAdmin ?? false,
      status: "active",
      // las 3 llaves quedan en false hasta la primera compra
    });
    if (userErr) throw new Error(`No se pudo crear el perfil: ${userErr.message}`);

    // --- 4. Crear su billetera (solo comisiones; fondo va aparte) ---
    const { error: walletErr } = await supabase
      .from("wallet")
      .insert({ user_id: userId });
    if (walletErr) throw new Error(`No se pudo crear la billetera: ${walletErr.message}`);

    return { userId, referralCode, placementId, binaryLeg };
  } catch (err) {
    // Rollback: si algo falló tras crear la cuenta de auth, la eliminamos
    // para no dejar cuentas huérfanas.
    await supabase.auth.admin.deleteUser(userId);
    throw err;
  }
}
```

### 17. `app/(onboarding)/layout.tsx`
Layout de la activación inicial.
```tsx
import { redirect } from "next/navigation";
import { requireUser } from "@/lib/auth/session";
import { logoutAction } from "@/app/(auth)/actions";
import { Wordmark } from "@/components/ui/Wordmark";
import { CircuitBackground } from "@/components/ui/CircuitBackground";
import { LanguageProvider } from "@/components/i18n/LanguageProvider";
import { LanguageSwitcher } from "@/components/i18n/LanguageSwitcher";
import { getLocale, getT } from "@/lib/i18n/server";
import { messages } from "@/lib/i18n/messages";

/**
 * Layout de onboarding (elección de paquete obligatoria).
 * Si el usuario ya tiene posición (ya compró), entra al backoffice.
 */
export default async function OnboardingLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await requireUser();
  if (user.has_position) redirect("/dashboard");

  const locale = getLocale();
  const t = getT();

  return (
    <LanguageProvider locale={locale} dict={messages[locale]}>
      <main className="circuit-overlay relative isolate min-h-screen overflow-hidden">
        <CircuitBackground />
        <header className="relative z-10 border-b border-border bg-navyDeep/80 backdrop-blur">
          <div className="mx-auto flex max-w-content items-center justify-between px-6 py-4">
            <Wordmark showSubtitle={false} />
            <div className="flex items-center gap-3">
              <LanguageSwitcher />
              <form action={logoutAction}>
                <button className="rounded-md border border-border px-3 py-1.5 text-xs text-cream hover:border-gold">
                  {t("common.logout")}
                </button>
              </form>
            </div>
          </div>
        </header>
        <div className="relative z-10 mx-auto max-w-content px-6 py-12">{children}</div>
      </main>
    </LanguageProvider>
  );
}
```

### 18. `app/(onboarding)/bienvenida/page.tsx`
Primera compra: elige y activa un programa para obtener posición.
```tsx
import { requireUser } from "@/lib/auth/session";
import { getCatalog } from "@/lib/data/distributor";
import { Store } from "@/components/dashboard/Store";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Activa tu cuenta — TRIADA" };

export default async function BienvenidaPage() {
  const user = await requireUser();
  const catalog = await getCatalog();
  const t = getT();

  return (
    <div className="space-y-8">
      <div className="text-center">
        <p className="label">{t("onb.welcome")}</p>
        <h1 className="mt-2 text-4xl font-extrabold leading-tight text-cream">
          {(user.full_name?.split(" ")[0] ?? "")} — {t("onb.activate")}
        </h1>
        <p className="mx-auto mt-3 max-w-xl text-muted">{t("onb.subtitle")}</p>
      </div>

      <Store catalog={catalog} />
    </div>
  );
}
```


---

# PARTE 3 — SHELL, MENÚ Y NAVEGACIÓN
Estructura visual del panel: sidebar dinámico por llaves, cabecera, tema e idioma.

### 19. `app/(distribuidor)/layout.tsx`
Layout protegido del afiliado: exige sesión y posición, calcula qué academias mostrar y monta el shell. Separa `right` (idioma/tema) de `userMenu`, que se oculta dentro de /ia porque el Hub trae su propio menú de perfil.
```tsx
import { redirect } from "next/navigation";
import { requireUser } from "@/lib/auth/session";
import { Sidebar } from "@/components/dashboard/Sidebar";
import { DashboardShell } from "@/components/dashboard/DashboardShell";
import { UserMenu } from "@/components/dashboard/UserMenu";
import { ThemeToggle } from "@/components/ui/ThemeToggle";
import { LanguageProvider } from "@/components/i18n/LanguageProvider";
import { LanguageSwitcher } from "@/components/i18n/LanguageSwitcher";
import { getLocale } from "@/lib/i18n/server";
import { messages } from "@/lib/i18n/messages";
import { createAdminClient } from "@/lib/supabase/admin";

/** Layout protegido del distribuidor con sidebar dinámico según activación. */
export default async function DistribuidorLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const user = await requireUser();

  // Debe comprar al menos un producto para acceder al back office.
  if (!user.has_position) redirect("/bienvenida");

  const supabase = createAdminClient();
  const { data: mems } = await supabase
    .from("memberships")
    .select("points")
    .eq("user_id", user.id)
    .eq("type", "service")
    .eq("status", "active");
  const maxServicePoints = (mems ?? []).reduce((m, x) => Math.max(m, x.points ?? 0), 0);

  const locale = getLocale();

  return (
    <LanguageProvider locale={locale} dict={messages[locale]}>
      <DashboardShell
        sidebar={
          <Sidebar
            canDistribute={user.can_distribute}
            hasNova={maxServicePoints >= 50}
            hasQuantum={maxServicePoints >= 100}
          />
        }
        right={
          <>
            <LanguageSwitcher />
            <ThemeToggle />
          </>
        }
        userMenu={
          <UserMenu name={user.full_name ?? user.email} code={user.referral_code} avatarUrl={user.avatar_url} rankLevel={user.current_rank} />
        }
      >
        {children}
      </DashboardShell>
    </LanguageProvider>
  );
}
```

### 20. `components/dashboard/DashboardShell.tsx`
Shell responsive: sidebar colapsable (se recuerda en localStorage y se oculta solo al entrar a /ia), drawer en móvil y cabecera.
```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import { usePathname } from "next/navigation";
import { Menu, X, PanelLeftClose, PanelLeftOpen } from "lucide-react";

const COLLAPSE_KEY = "triada-sidebar-collapsed";

/**
 * Estructura responsive del back office: sidebar plegable (overlay en móvil,
 * fijo en desktop) + barra superior + contenido.
 *
 * El sidebar principal tiene un colapso de escritorio independiente del
 * drawer móvil: el usuario lo puede ocultar/mostrar con el botón del header
 * (se recuerda entre sesiones), y además se OCULTA AUTOMÁTICAMENTE al entrar
 * a /ia (el Centro de IA ya trae su propia navegación — no necesita el
 * sidebar principal estorbando). El panel de historial DENTRO del Centro de
 * IA es independiente de este y nunca se auto-oculta.
 */
export function DashboardShell({
  sidebar,
  right,
  userMenu,
  children,
}: {
  sidebar: React.ReactNode;
  right: React.ReactNode;
  /** Menú de cuenta genérico (nombre/foto/salir). Se oculta dentro de /ia:
   *  el Hub Triada ya trae su propio menú de perfil más completo (créditos,
   *  paquete, galería...), así que mostrar ambos sería redundante. */
  userMenu?: React.ReactNode;
  children: React.ReactNode;
}) {
  const [open, setOpen] = useState(false);
  const [collapsed, setCollapsed] = useState(false);
  const pathname = usePathname();
  const hydrated = useRef(false);
  const inHub = pathname === "/ia" || pathname.startsWith("/ia/");

  // Cerrar el menú al navegar (en móvil).
  useEffect(() => setOpen(false), [pathname]);

  // Restaurar la preferencia guardada al montar.
  useEffect(() => {
    try {
      const saved = localStorage.getItem(COLLAPSE_KEY);
      if (saved === "1") setCollapsed(true);
    } catch {
      /* localStorage no disponible: seguir expandido */
    }
    hydrated.current = true;
  }, []);

  // Auto-ocultar al ENTRAR al Centro de IA (una vez por navegación a /ia).
  useEffect(() => {
    if (!hydrated.current) return;
    if (pathname === "/ia" || pathname.startsWith("/ia/")) setCollapsed(true);
  }, [pathname]);

  function toggleCollapsed() {
    setCollapsed((c) => {
      const next = !c;
      try {
        localStorage.setItem(COLLAPSE_KEY, next ? "1" : "0");
      } catch {
        /* ignorar */
      }
      return next;
    });
  }

  return (
    <div className="flex min-h-screen">
      {/* Backdrop (móvil) */}
      {open && (
        <div
          onClick={() => setOpen(false)}
          className="fixed inset-0 z-30 bg-black/50 lg:hidden"
          aria-hidden
        />
      )}

      {/* Sidebar */}
      <aside
        className={`fixed inset-y-0 left-0 z-40 w-60 shrink-0 transform border-r border-border bg-navyDeep transition-all duration-200 lg:static lg:translate-x-0 ${
          open ? "translate-x-0" : "-translate-x-full"
        } ${collapsed ? "lg:w-0 lg:overflow-hidden lg:border-r-0" : "lg:w-60"}`}
      >
        <button
          onClick={() => setOpen(false)}
          className="absolute right-3 top-3 text-muted hover:text-cream lg:hidden"
          aria-label="Cerrar menú"
        >
          <X size={20} />
        </button>
        <div className="w-60">{sidebar}</div>
      </aside>

      {/* Columna principal */}
      <div className="flex min-w-0 flex-1 flex-col">
        <header className="border-b border-border bg-navyDeep/80 backdrop-blur">
          <div className="flex items-center gap-2 px-4 py-3 sm:px-6">
            <button
              onClick={() => setOpen(true)}
              className="rounded-md border border-border p-2 text-cream hover:border-gold lg:hidden"
              aria-label="Abrir menú"
            >
              <Menu size={18} />
            </button>
            <button
              onClick={toggleCollapsed}
              className="hidden rounded-md border border-border p-2 text-cream hover:border-gold lg:inline-flex"
              aria-label={collapsed ? "Mostrar panel" : "Ocultar panel"}
              title={collapsed ? "Mostrar panel" : "Ocultar panel"}
            >
              {collapsed ? <PanelLeftOpen size={18} /> : <PanelLeftClose size={18} />}
            </button>
            <div className="ml-auto flex items-center gap-3">
              {right}
              {!inHub && userMenu}
            </div>
          </div>
        </header>
        <main className="flex-1 px-4 py-6 sm:px-6 sm:py-8">{children}</main>
      </div>
    </div>
  );
}
```

### 21. `components/dashboard/Sidebar.tsx`
Menú lateral dinámico: Inicio · Centro de IA · Centro de Negocio (solo con IBO) · Centro de Estudio (según servicio).
```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import {
  LayoutDashboard,
  Share2,
  Coins,
  Award,
  Wallet,
  KeySquare,
  ShoppingBag,
  Receipt,
  BookOpen,
  Calculator,
  Sparkles,
  FolderOpen,
  LogOut,
} from "lucide-react";
import { Wordmark } from "@/components/ui/Wordmark";
import { useT } from "@/components/i18n/LanguageProvider";
import { logoutAction } from "@/app/(auth)/actions";
import { cn } from "@/lib/utils";

type Item = { href: string; key: string; icon: typeof LayoutDashboard };
type Group = { titleKey: string | null; items: Item[] };

interface Props {
  canDistribute: boolean;
  hasNova: boolean;
  hasQuantum: boolean;
}

export function Sidebar({ canDistribute, hasNova, hasQuantum }: Props) {
  const pathname = usePathname();
  const { t } = useT();

  const groups: Group[] = [
    {
      titleKey: "group.inicio",
      items: [
        { href: "/dashboard", key: "nav.dashboard", icon: LayoutDashboard },
        { href: "/programas", key: "nav.programs", icon: KeySquare },
        { href: "/tienda", key: "nav.store", icon: ShoppingBag },
        { href: "/pedidos", key: "store.orders", icon: Receipt },
      ],
    },
    {
      titleKey: "group.center",
      items: [
        { href: "/ia", key: "nav.ia", icon: Sparkles },
        { href: "/proyectos", key: "nav.projects", icon: FolderOpen },
      ],
    },
  ];

  if (canDistribute) {
    groups.push({
      titleKey: "group.business",
      items: [
        { href: "/red", key: "nav.network", icon: Share2 },
        { href: "/comisiones", key: "nav.commissions", icon: Coins },
        { href: "/rango", key: "nav.rank", icon: Award },
        { href: "/billetera", key: "nav.wallet", icon: Wallet },
        { href: "/herramientas", key: "nav.tools", icon: Calculator },
      ],
    });
  }

  if (hasNova || hasQuantum) {
    const study: Item[] = [];
    if (hasNova) study.push({ href: "/academia/nova", key: "nav.academiaNova", icon: BookOpen });
    if (hasQuantum) study.push({ href: "/academia/quantum", key: "nav.academiaQuantum", icon: BookOpen });
    groups.push({ titleKey: "group.study", items: study });
  }

  return (
    <div className="flex h-full min-h-screen flex-col lg:h-screen lg:sticky lg:top-0">
      <div className="px-6 py-6">
        <Link href="/dashboard">
          <Wordmark />
        </Link>
      </div>
      <nav className="flex-1 space-y-5 overflow-y-auto px-3 pb-4">
        {groups.map((group) => (
          <div key={group.titleKey ?? "base"}>
            {group.titleKey && (
              <p className="px-3 pb-2 text-[10px] font-semibold uppercase tracking-label text-muted/60">
                {t(group.titleKey)}
              </p>
            )}
            <div className="space-y-1">
              {group.items.map(({ href, key, icon: Icon }) => {
                const active = pathname === href || pathname.startsWith(href + "/");
                return (
                  <Link
                    key={href}
                    href={href}
                    className={cn(
                      "flex items-center gap-3 rounded-lg px-3 py-2.5 text-sm transition-colors",
                      active
                        ? "border-l-2 border-gold bg-card font-medium text-cream"
                        : "border-l-2 border-transparent text-muted hover:text-cream"
                    )}
                  >
                    <Icon size={18} className={active ? "text-gold" : "text-muted"} strokeWidth={1.75} />
                    {t(key)}
                  </Link>
                );
              })}
            </div>
          </div>
        ))}
      </nav>
      {/* Cerrar sesión al fondo del menú */}
      <div className="border-t border-border p-3">
        <form action={logoutAction}>
          <button className="flex w-full items-center gap-3 rounded-lg px-3 py-2.5 text-sm text-danger transition-colors hover:bg-card">
            <LogOut size={18} strokeWidth={1.75} />
            {t("common.logout")}
          </button>
        </form>
      </div>
    </div>
  );
}
```

### 22. `components/dashboard/UserMenu.tsx`
Menú de cuenta de la cabecera: foto, nombre, código, perfil y cerrar sesión.
```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import Link from "next/link";
import { User, ChevronDown, UserCog, KeyRound, ShieldCheck, LogOut } from "lucide-react";
import { logoutAction } from "@/app/(auth)/actions";
import { useT } from "@/components/i18n/LanguageProvider";
import { RankBadge } from "@/components/ui/RankBadge";
import { rankNameFor } from "@/lib/ranks";

/** Menú de usuario (arriba derecha): foto + nombre → opciones de cuenta. */
export function UserMenu({
  name,
  code,
  avatarUrl,
  rankLevel,
}: {
  name: string;
  code: string;
  avatarUrl?: string | null;
  rankLevel?: number | null;
}) {
  const { t } = useT();
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    function onClick(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false);
    }
    document.addEventListener("mousedown", onClick);
    return () => document.removeEventListener("mousedown", onClick);
  }, []);

  const items = [
    { href: "/perfil", label: t("menu.profile"), icon: UserCog },
    { href: "/perfil?tab=password", label: t("menu.changePassword"), icon: KeyRound },
    { href: "/perfil?tab=security", label: t("menu.security"), icon: ShieldCheck },
  ];

  return (
    <div className="relative" ref={ref}>
      <button
        onClick={() => setOpen((o) => !o)}
        className="flex items-center gap-2 rounded-full border border-border py-1 pl-1 pr-3 transition-colors hover:border-gold"
      >
        <span className="flex h-8 w-8 items-center justify-center overflow-hidden rounded-full bg-navyDeep">
          {avatarUrl ? (
            // eslint-disable-next-line @next/next/no-img-element
            <img src={avatarUrl} alt="" className="h-full w-full object-cover" />
          ) : (
            <User size={16} className="text-gold" />
          )}
        </span>
        <span className="hidden max-w-[140px] truncate text-sm text-cream sm:inline">{name}</span>
        <ChevronDown size={14} className="text-muted" />
      </button>

      {open && (
        <div className="absolute right-0 z-50 mt-2 w-60 overflow-hidden rounded-card border border-border bg-card shadow-card">
          <div className="flex items-center gap-3 border-b border-border px-4 py-3">
            <RankBadge level={rankLevel} size="sm" />
            <div className="min-w-0">
              <p className="truncate text-sm font-semibold text-cream">{name}</p>
              <p className="text-[11px] uppercase tracking-wide text-muted">
                {rankLevel ? rankNameFor(rankLevel) : code}
              </p>
            </div>
          </div>
          <nav className="py-1">
            {items.map(({ href, label, icon: Icon }) => (
              <Link
                key={href}
                href={href}
                onClick={() => setOpen(false)}
                className="flex items-center gap-3 px-4 py-2.5 text-sm text-cream hover:bg-navyDeep"
              >
                <Icon size={16} className="text-muted" />
                {label}
              </Link>
            ))}
            <form action={logoutAction} className="border-t border-border">
              <button className="flex w-full items-center gap-3 px-4 py-2.5 text-sm text-danger hover:bg-navyDeep">
                <LogOut size={16} />
                {t("common.logout")}
              </button>
            </form>
          </nav>
        </div>
      )}
    </div>
  );
}
```

### 23. `components/ui/ThemeToggle.tsx`
Botón de tema día/noche.
```tsx
"use client";

import { useEffect, useState } from "react";
import { Sun, Moon } from "lucide-react";

/**
 * Toggle de tema día/noche. Escribe `data-theme="light"` en <html> y lo
 * persiste en localStorage. El valor inicial lo fija un script en el layout
 * raíz (antes de pintar) para evitar parpadeo.
 */
export function ThemeToggle() {
  const [theme, setTheme] = useState<"dark" | "light">("dark");

  useEffect(() => {
    const current = document.documentElement.getAttribute("data-theme");
    setTheme(current === "light" ? "light" : "dark");
  }, []);

  function toggle() {
    const next = theme === "dark" ? "light" : "dark";
    setTheme(next);
    if (next === "light") {
      document.documentElement.setAttribute("data-theme", "light");
    } else {
      document.documentElement.removeAttribute("data-theme");
    }
    try {
      localStorage.setItem("triada-theme", next);
    } catch {
      /* almacenamiento no disponible */
    }
  }

  return (
    <button
      onClick={toggle}
      aria-label={theme === "dark" ? "Activar tema claro" : "Activar tema oscuro"}
      title={theme === "dark" ? "Tema claro" : "Tema oscuro"}
      className="flex h-8 w-8 items-center justify-center rounded-md border border-border text-cream transition-colors hover:border-gold"
    >
      {theme === "dark" ? <Sun size={16} /> : <Moon size={16} />}
    </button>
  );
}
```

### 24. `components/ui/Wordmark.tsx`
Logotipo TRIADA.
```tsx
import { cn } from "@/lib/utils";

/**
 * Wordmark TRIADA con la ligadura IA en oro.
 * Reglas de marca: TR_DA en crema, la "IA" central en oro (#C2B280),
 * representa Inteligencia Artificial incrustada en el nombre.
 * "BUSINESS SCHOOL" debajo en muted con tracking amplio.
 */
export function Wordmark({
  className,
  showSubtitle = true,
}: {
  className?: string;
  showSubtitle?: boolean;
}) {
  return (
    <div className={cn("inline-flex flex-col", className)}>
      <span className="font-display text-3xl font-extrabold leading-none tracking-tight text-cream">
        TR<span className="text-gold">IA</span>DA
      </span>
      {showSubtitle && (
        <span className="mt-1 font-display text-[10px] font-semibold uppercase tracking-label text-muted">
          Business School
        </span>
      )}
    </div>
  );
}
```

### 25. `components/ui/Card.tsx`
`Card` y `StatCard` que usan todas las pantallas.
```tsx
import { cn } from "@/lib/utils";

/** Card base TRIADA: navy card, borde sutil, radius 12px. */
export function Card({
  className,
  premium = false,
  children,
}: {
  className?: string;
  premium?: boolean;
  children: React.ReactNode;
}) {
  return (
    <div
      className={cn(
        "rounded-card border bg-card p-6 shadow-card",
        premium
          ? "border-gold/40 border-t-2 border-t-gold bg-gradient-to-br from-card to-navy"
          : "border-border",
        className
      )}
    >
      {children}
    </div>
  );
}

/** StatCard: métrica de dashboard (valor en oro, label muted). */
export function StatCard({
  label,
  value,
  hint,
}: {
  label: string;
  value: string;
  hint?: string;
}) {
  return (
    <Card>
      <p className="label">{label}</p>
      <p className="mt-2 font-display text-4xl font-extrabold text-gold">{value}</p>
      {hint && <p className="mt-1 text-xs text-muted">{hint}</p>}
    </Card>
  );
}
```

### 26. `components/ui/Button.tsx`
Botón base.
```tsx
import { cn } from "@/lib/utils";

type Variant = "primary" | "secondary" | "ghost" | "danger";

const variants: Record<Variant, string> = {
  // PRIMARY: fondo oro, texto navy.
  primary:
    "bg-gold text-goldInk hover:bg-goldLight hover:scale-[1.02] font-semibold",
  // SECONDARY: borde oro, texto oro.
  secondary:
    "bg-transparent text-gold border border-gold hover:bg-gold/10",
  // GHOST: borde sutil, texto crema.
  ghost:
    "bg-transparent text-cream border border-border hover:border-gold",
  // DANGER
  danger: "bg-danger text-white hover:opacity-90",
};

export function Button({
  variant = "primary",
  className,
  ...props
}: React.ButtonHTMLAttributes<HTMLButtonElement> & { variant?: Variant }) {
  return (
    <button
      className={cn(
        "inline-flex items-center justify-center rounded-[4px] px-8 py-3.5",
        "font-display text-sm uppercase tracking-[0.1em]",
        "transition-all duration-150 ease-out disabled:opacity-50 disabled:pointer-events-none",
        variants[variant],
        className
      )}
      {...props}
    />
  );
}
```

### 27. `components/ui/Input.tsx`
Campo de formulario base.
```tsx
import { cn } from "@/lib/utils";

/** Input con estilo TRIADA (label uppercase muted + focus dorado). */
export function Field({
  label,
  className,
  ...props
}: React.InputHTMLAttributes<HTMLInputElement> & { label: string }) {
  return (
    <label className="block">
      <span className="mb-1.5 block font-sans text-xs font-medium uppercase tracking-[0.08em] text-muted">
        {label}
      </span>
      <input
        className={cn(
          "w-full rounded-md border border-border bg-navyDeep px-3 py-2.5",
          "text-sm text-cream placeholder:text-muted/60",
          "outline-none transition-colors focus:border-gold focus:ring-2 focus:ring-gold/15",
          className
        )}
        {...props}
      />
    </label>
  );
}
```

### 28. `components/ui/RankBadge.tsx`
Insignia PNG del rango (13 niveles) o marcador neutro.
```tsx
import Image from "next/image";
import { rankNameFor } from "@/lib/ranks";

/**
 * Insignia visual de un rango (PNG con fondo transparente en /public/rangos).
 *
 * Componente presentacional puro: sirve tanto en server como en client.
 * Recibe el NIVEL (1..13); si el nivel es 0/nulo o fuera de rango, muestra un
 * marcador neutro "sin rango". El nombre se usa como texto alternativo.
 */
const SIZE_PX = {
  xs: 24,
  sm: 32,
  md: 56,
  lg: 96,
  xl: 160,
} as const;

type Size = keyof typeof SIZE_PX;

export function RankBadge({
  level,
  size = "md",
  className = "",
}: {
  level: number | null | undefined;
  size?: Size;
  className?: string;
}) {
  const px = SIZE_PX[size];

  // Sin rango aún: marcador neutro (mismo tamaño para no romper el layout).
  if (!level || level < 1 || level > 13) {
    return (
      <span
        aria-label="Sin rango"
        className={`inline-flex shrink-0 items-center justify-center rounded-full border border-border bg-navyDeep text-muted ${className}`}
        style={{ width: px, height: px, fontSize: Math.round(px * 0.42), lineHeight: 1 }}
      >
        –
      </span>
    );
  }

  const name = rankNameFor(level);
  return (
    <Image
      src={`/rangos/rango-${level}.png`}
      alt={name}
      title={name}
      width={px}
      height={px}
      className={`shrink-0 select-none object-contain ${className}`}
    />
  );
}
```

### 29. `components/ui/Flag.tsx`
Banderas de países.
```tsx
import { countryIso } from "@/lib/data/countries";

/**
 * Bandera SVG (flag-icons) por nombre de país. Se ve en todos los sistemas
 * (a diferencia de los emojis de bandera, que Windows no dibuja).
 */
export function Flag({ country, className = "" }: { country: string | null; className?: string }) {
  const iso = countryIso(country);
  if (!iso) return <span className={className}>🌐</span>;
  return (
    <span
      className={`fi fi-${iso} inline-block rounded-sm ${className}`}
      style={{ width: "1.25em", height: "0.9em" }}
      title={country ?? undefined}
    />
  );
}
```

### 30. `components/ui/CircuitBackground.tsx`
Fondo animado de circuito (pantallas de acceso).
```tsx
/**
 * Fondo animado de circuito tech (textura característica de TRIADA).
 *
 * Marca: el circuito de IA es LA textura de TRIADA. Aquí va como fondo sutil
 * (baja opacidad sobre navy) con pulsos de luz que viajan por las trazas —
 * movimiento "preciso, arquitectónico, intencional", NO partículas aleatorias.
 *
 * - Trazas base: tenues, color borde.
 * - Pulsos: un destello corto (oro/cobalto) recorre cada traza lentamente.
 * - Respeta `prefers-reduced-motion` (los pulsos se ocultan; quedan las trazas).
 * - Solo para páginas "vitrina" (inicio/login/registro). En el dashboard NO,
 *   para no competir con las cifras financieras.
 */

// Trazas estilo PCB (ángulos rectos) repartidas por el lienzo 1440x900.
const TRACES = [
  "M-20 90 H160 L200 130 H380 L420 90 H640",
  "M1460 180 H1240 L1200 220 H1000 L960 180 H760",
  "M-20 300 H120 L160 340 H320 L360 300 H560",
  "M1460 360 H1300 L1260 320 H1080 L1040 360 H880",
  "M-20 520 H200 L240 560 H440 L480 520 H700",
  "M1460 560 H1260 L1220 600 H1020 L980 560 H820",
  "M-20 720 H160 L200 680 H380 L420 720 H620",
  "M1460 760 H1280 L1240 720 H1060 L1020 760 H860",
  "M720 -20 V120 L760 160 V320",
  "M520 920 V760 L560 720 V540",
];

// Nodos (puntos de conexión) con brillo pulsante.
const NODES = [
  { x: 640, y: 90 }, { x: 760, y: 180 }, { x: 560, y: 300 },
  { x: 880, y: 360 }, { x: 700, y: 520 }, { x: 820, y: 560 },
  { x: 620, y: 720 }, { x: 860, y: 760 }, { x: 760, y: 160 },
  { x: 560, y: 720 },
];

// Colores alternados de los pulsos: oro y cobalto.
const PULSE_COLORS = ["#C2B280", "#0353A4"];

export function CircuitBackground() {
  return (
    <div
      aria-hidden="true"
      className="pointer-events-none absolute inset-0 -z-10 overflow-hidden"
    >
      <svg
        viewBox="0 0 1440 900"
        preserveAspectRatio="xMidYMid slice"
        className="h-full w-full opacity-[0.55]"
      >
        {/* Capa 1 — trazas base tenues */}
        <g stroke="#1A3570" strokeWidth={1.5} fill="none" strokeLinejoin="round">
          {TRACES.map((d, i) => (
            <path key={`base-${i}`} d={d} />
          ))}
        </g>

        {/* Capa 2 — pulsos de luz que recorren cada traza */}
        <g className="circuit-pulse" fill="none" strokeLinecap="round">
          {TRACES.map((d, i) => (
            <path
              key={`pulse-${i}`}
              d={d}
              pathLength={1000}
              stroke="currentColor"
              strokeWidth={2}
              style={{
                color: PULSE_COLORS[i % PULSE_COLORS.length],
                // dash corto que viaja por toda la traza (pathLength normalizado a 1000)
                strokeDasharray: "16 984",
                animationName: "circuit-travel",
                animationTimingFunction: "linear",
                animationIterationCount: "infinite",
                animationDuration: `${9 + (i % 5) * 2.5}s`,
                animationDelay: `${i * 1.3}s`,
              }}
            />
          ))}
        </g>

        {/* Capa 3 — nodos con brillo pulsante */}
        <g fill="#C2B280" className="circuit-nodes">
          {NODES.map((n, i) => (
            <circle
              key={`node-${i}`}
              cx={n.x}
              cy={n.y}
              r={2.5}
              style={{
                animationName: "circuit-node-glow",
                animationTimingFunction: "ease-in-out",
                animationIterationCount: "infinite",
                animationDuration: `${4 + (i % 4)}s`,
                animationDelay: `${i * 0.6}s`,
              }}
            />
          ))}
        </g>
      </svg>
    </div>
  );
}
```

### 31. `components/i18n/LanguageProvider.tsx`
Contexto de idioma: `useT()` devuelve la función `t(clave)`.
```tsx
"use client";

import { createContext, useContext } from "react";
import type { Locale } from "@/lib/i18n/config";

type Dict = Record<string, string>;
interface Ctx {
  locale: Locale;
  t: (key: string) => string;
}

const LanguageContext = createContext<Ctx>({ locale: "es", t: (k) => k });

/** Provee la traducción a componentes cliente. El dict viene del servidor. */
export function LanguageProvider({
  locale,
  dict,
  children,
}: {
  locale: Locale;
  dict: Dict;
  children: React.ReactNode;
}) {
  const t = (key: string) => dict[key] ?? key;
  return <LanguageContext.Provider value={{ locale, t }}>{children}</LanguageContext.Provider>;
}

export function useT() {
  return useContext(LanguageContext);
}
```

### 32. `components/i18n/LanguageSwitcher.tsx`
Selector ES / EN / PT.
```tsx
"use client";

import { Globe } from "lucide-react";
import { useT } from "./LanguageProvider";
import { LOCALES, LOCALE_LABELS, type Locale } from "@/lib/i18n/config";

/** Selector de idioma. Guarda la cookie y recarga para re-renderizar todo. */
export function LanguageSwitcher() {
  const { locale } = useT();

  function change(next: Locale) {
    document.cookie = `triada-lang=${next}; path=/; max-age=31536000`;
    window.location.reload();
  }

  return (
    <div className="relative inline-flex items-center">
      <Globe size={14} className="pointer-events-none absolute left-2 text-muted" />
      <select
        value={locale}
        onChange={(e) => change(e.target.value as Locale)}
        aria-label="Idioma"
        className="appearance-none rounded-md border border-border bg-navyDeep py-1.5 pl-7 pr-3 text-xs text-cream outline-none hover:border-gold"
      >
        {LOCALES.map((l) => (
          <option key={l} value={l}>{LOCALE_LABELS[l]}</option>
        ))}
      </select>
    </div>
  );
}
```

### 33. `lib/i18n/config.ts`
Idiomas soportados.
```ts
/** Idiomas soportados. */
export const LOCALES = ["es", "en", "pt"] as const;
export type Locale = (typeof LOCALES)[number];
export const DEFAULT_LOCALE: Locale = "es";

export const LOCALE_LABELS: Record<Locale, string> = {
  es: "Español",
  en: "English",
  pt: "Português",
};
```

### 34. `lib/i18n/server.ts`
`getLocale()` y `getT()` para Server Components.
```ts
import { cookies } from "next/headers";
import { messages } from "./messages";
import { DEFAULT_LOCALE, LOCALES, type Locale } from "./config";

/** Idioma actual (cookie 'triada-lang', por defecto español). SOLO servidor. */
export function getLocale(): Locale {
  const c = cookies().get("triada-lang")?.value as Locale | undefined;
  return c && LOCALES.includes(c) ? c : DEFAULT_LOCALE;
}

/** Devuelve la función de traducción `t(key)` para el idioma actual. */
export function getT() {
  const locale = getLocale();
  const dict = messages[locale];
  const fallback = messages[DEFAULT_LOCALE];
  return (key: string) => dict[key] ?? fallback[key] ?? key;
}
```

### 35. `lib/ranks.ts`
Nombres canónicos de los 13 rangos y utilidades nivel ↔ nombre.
```ts
/**
 * Catálogo canónico de los 13 rangos (nivel ↔ nombre).
 *
 * La fuente de verdad de los datos del plan es la tabla `ranks` en la BD; esto
 * es solo el mapa de NOMBRES para la capa visual (texto alternativo de las
 * insignias y resolución nombre→nivel donde solo tenemos el nombre).
 * Si se renombra un rango, actualizar aquí y en `db/seeds/002_seed.sql`.
 */
export const RANK_NAMES: Record<number, string> = {
  1: "PIONERO",
  2: "IMPULSOR",
  3: "ÉLITE",
  4: "EXECUTIVE",
  5: "VISIONARIO",
  6: "CONQUISTADOR",
  7: "SOBERANO",
  8: "DIAMANTE",
  9: "DIAMANTE IMPERIAL",
  10: "DIAMANTE MONARCA",
  11: "LEYENDA",
  12: "LEYENDA TRIADA",
  13: "LEYENDA CORONA",
};

export const MAX_RANK = 13;

/** Nombre del rango para un nivel (fallback genérico si está fuera de rango). */
export function rankNameFor(level: number): string {
  return RANK_NAMES[level] ?? `Rango ${level}`;
}

/** Normaliza para comparar nombres (sin acentos, mayúsculas, sin espacios extra). */
function normalize(s: string): string {
  return s
    .normalize("NFD")
    .replace(/[̀-ͯ]/g, "")
    .trim()
    .toUpperCase();
}

const LEVEL_BY_NAME = new Map<string, number>(
  Object.entries(RANK_NAMES).map(([lvl, name]) => [normalize(name), Number(lvl)])
);

/** Resuelve el nivel (1..13) a partir del nombre del rango. 0 si no se reconoce. */
export function rankLevelByName(name: string | null | undefined): number {
  if (!name) return 0;
  return LEVEL_BY_NAME.get(normalize(name)) ?? 0;
}
```

### 36. `lib/utils.ts`
`cn()` y `formatUSD()`.
```ts
/**
 * cn — une clases condicionales sin dependencias externas.
 * (Equivalente minimalista a clsx; suficiente para el prototipo.)
 */
export function cn(...classes: Array<string | false | null | undefined>): string {
  return classes.filter(Boolean).join(" ");
}

/**
 * Formatea un monto en USD para mostrar en UI (billetera, comisiones).
 */
export function formatUSD(amount: number): string {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: "USD",
  }).format(amount);
}
```

### 37. `types/database.ts`
Tipos TypeScript del esquema de la base de datos.
```ts
/**
 * Tipos del dominio TRIADA, alineados con db/migrations/001_schema.sql.
 *
 * En Fase B/C, una vez conectado Supabase, se pueden REGENERAR automáticamente con:
 *   npx supabase gen types typescript --project-id <id> > types/supabase.ts
 * Por ahora se mantienen a mano para tener el contrato visible en el prototipo.
 */

// ---- Enums ----
export type BinaryLeg = "left" | "right";
export type UserStatus = "active" | "inactive" | "suspended";
export type MembershipType = "ibo" | "service";
export type MembershipStatus = "active" | "expired" | "cancelled";
export type ProductType = "service" | "physical" | "ibo";
export type OrderType = "enrollment" | "renewal" | "product";
export type PaymentStatus = "pending" | "paid" | "refunded";
export type CommissionType =
  | "direct_l1"
  | "direct_l2"
  | "binary"
  | "rank"
  | "matching"
  | "lifestyle";
export type CommissionStatus = "pending" | "validated" | "paid" | "rejected";
export type WithdrawalStatus = "pending" | "approved" | "paid" | "rejected";

// ---- Tablas ----
export interface User {
  id: string;
  username: string | null;
  email: string;
  full_name: string | null;
  country: string | null;
  phone: string | null;
  birth_date: string | null;
  city: string | null;
  postal_code: string | null;
  gender: string | null;
  address: string | null;
  avatar_url: string | null;
  sponsor_id: string | null;
  placement_id: string | null;
  binary_leg: BinaryLeg | null;
  // Las 3 llaves de activación (independientes):
  has_position: boolean;
  can_distribute: boolean;
  can_earn: boolean;
  referral_code: string;
  current_rank: number;
  kyc_verified: boolean;
  is_admin: boolean;
  status: UserStatus;
  created_at: string;
  updated_at: string;
}

export interface Product {
  id: string;
  code: string;
  name: string;
  type: ProductType;
  enrollment_price: number;
  renewal_price: number | null;
  points: number;
  duration_months: number;
  activates_earning: boolean;
  fund_amount: number; // capital NO comisionable (Nexus = 500)
  is_active: boolean;
  created_at: string;
}

export interface Membership {
  id: string;
  user_id: string;
  type: MembershipType;
  program_code: string | null;
  points: number;
  status: MembershipStatus;
  starts_at: string;
  expires_at: string | null;
  auto_renew: boolean;
  created_at: string;
}

export interface Order {
  id: string;
  user_id: string;
  product_id: string;
  order_type: OrderType;
  amount: number;
  points_awarded: number;
  is_promotional: boolean;
  payment_status: PaymentStatus;
  payment_method: string | null;
  tx_hash: string | null;
  created_at: string;
}

export interface VolumeLedger {
  id: string;
  user_id: string;
  period: string; // 'YYYY-MM'
  left_volume: number;
  right_volume: number;
  left_carryover: number;
  right_carryover: number;
  paid_leg_volume: number;
  created_at: string;
}

export interface PromotionalPoints {
  id: string;
  user_id: string;
  bonus_points: number;
  reason: string;
  expires_at: string;
  active: boolean;
  created_at: string;
}

export interface Commission {
  id: string;
  user_id: string;
  source_user_id: string | null;
  period: string;
  type: CommissionType;
  amount: number;
  status: CommissionStatus;
  created_at: string;
}

export interface Wallet {
  id: string;
  user_id: string;
  balance: number;
  total_earned: number;
  total_withdrawn: number;
  updated_at: string;
}

export interface Withdrawal {
  id: string;
  user_id: string;
  amount: number;
  wallet_address: string;
  status: WithdrawalStatus;
  tx_hash: string | null;
  requested_at: string;
  processed_at: string | null;
}

export interface Rank {
  level: number; // 1-13
  name: string;
  zone: string | null;
  points_per_leg: number;
  max_pct_line: number;
  min_directs_left: number;
  min_directs_right: number;
  base_check: number;
  matching_levels: number[]; // [0.50, 0.10, ...]
  lifestyle_bonus: number;
  lifestyle_directs: number;
}

export interface CompensationConfig {
  key: string;
  value: number | string;
  description: string | null;
  updated_at: string;
}
```


---

# PARTE 4 — LECTURA: CAPA DE DATOS DEL AFILIADO
Todas las consultas que alimentan las pantallas. Solo servidor.

### 38. `lib/data/distributor.ts`
Dashboard de distribuidor y de estudiante, comisiones, ganancias detalladas, rango y sus condiciones, billetera y direcciones, programas, catálogo, referidos pendientes y consultas de red.
```ts
/**
 * CAPA DE DATOS DEL DISTRIBUIDOR.
 *
 * ⚠️ SOLO SERVIDOR. Usa el cliente admin pero SIEMPRE acota las consultas al
 * `userId` del usuario autenticado (que viene de requireUser en las páginas),
 * de modo que cada quien ve únicamente su propia información y su downline.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { currentPeriod, previousPeriod } from "@/lib/engine/period";
import { loadRanks, loadEngineConfig, type RankRule } from "@/lib/engine/config";
import { qualifyRank } from "@/lib/engine/calculations";
import type { BinaryLeg, CommissionType } from "@/types/database";

// ----------------------------------------------------------------------------
// Helpers de árbol
// ----------------------------------------------------------------------------

/** Pierna de `ancestorId` bajo la que cae `descendantId` (subiendo el binario). */
async function findLegUnder(
  ancestorId: string,
  descendantId: string
): Promise<BinaryLeg | null> {
  const supabase = createAdminClient();
  let current: { placement_id: string | null; binary_leg: BinaryLeg | null } | null = (
    await supabase
      .from("users")
      .select("placement_id, binary_leg")
      .eq("id", descendantId)
      .maybeSingle()
  ).data;

  for (let d = 0; d < 100000 && current?.placement_id; d++) {
    if (current.placement_id === ancestorId) return current.binary_leg;
    current = (
      await supabase
        .from("users")
        .select("placement_id, binary_leg")
        .eq("id", current.placement_id)
        .maybeSingle()
    ).data;
  }
  return null;
}

/** Cuenta directos (patrocinados) activos separados por pierna binaria. */
async function directsActiveByLeg(
  userId: string
): Promise<{ left: number; right: number; total: number }> {
  const supabase = createAdminClient();
  const { data: directs } = await supabase
    .from("users")
    .select("id")
    .eq("sponsor_id", userId)
    .eq("can_earn", true);

  let left = 0;
  let right = 0;
  for (const d of directs ?? []) {
    const leg = await findLegUnder(userId, d.id);
    if (leg === "left") left++;
    else if (leg === "right") right++;
  }
  return { left, right, total: (directs ?? []).length };
}

/** Volúmenes del período en curso incluyendo el carry-over del mes anterior. */
async function liveTotals(userId: string, period: string) {
  const supabase = createAdminClient();
  const prev = previousPeriod(period);

  const { data: cur } = await supabase
    .from("volume_ledger")
    .select("left_volume, right_volume")
    .eq("user_id", userId)
    .eq("period", period)
    .maybeSingle();
  const { data: prevL } = await supabase
    .from("volume_ledger")
    .select("left_carryover, right_carryover")
    .eq("user_id", userId)
    .eq("period", prev)
    .maybeSingle();

  const leftTotal = (cur?.left_volume ?? 0) + (prevL?.left_carryover ?? 0);
  const rightTotal = (cur?.right_volume ?? 0) + (prevL?.right_carryover ?? 0);
  return { leftTotal, rightTotal };
}

// ----------------------------------------------------------------------------
// DASHBOARD
// ----------------------------------------------------------------------------
export interface DashboardData {
  period: string;
  leftTotal: number;
  rightTotal: number;
  directsActive: number;
  currentRank: number;
  currentRankName: string | null;
  nextRank: RankRule | null;
  nextRankProgressPct: number;
  check: Record<"direct" | "binary" | "rank" | "matching" | "lifestyle" | "total", number>;
  walletBalance: number;
  // --- Extras del dashboard enriquecido ---
  totalEarned: number;
  pendingToCollect: number; // comisiones validadas aún no pagadas
  pvPersonal: number; // puntos del servicio activo propio
  pvGroup: number; // volumen de grupo del período (izq + der)
  sponsorCode: string | null;
  monthlyIncome: { period: string; amount: number }[];
  incomeDaily: { date: string; amount: number }[]; // serie diaria YYYY-MM-DD
  topEarners: { name: string; code: string; amount: number }[];
  newMembers: { name: string; code: string; date: string }[];
  withdrawals: { solicitado: number; aprobado: number; pagado: number; rechazado: number };
  // --- Perfil / estado de paquete ---
  packageName: string | null; // servicio activo (NOVA/QUANTUM…)
  nextPurchaseDate: string | null; // próxima renovación (expira más cercano)
  hasActivePackage: boolean;
}

export async function getDashboardData(userId: string): Promise<DashboardData> {
  const supabase = createAdminClient();
  const period = currentPeriod();
  const ranks = await loadRanks();

  const [{ leftTotal, rightTotal }, directsLeg, user, wallet, periodComms, allComms, directs, wds, pvPersonal, mems] =
    await Promise.all([
      liveTotals(userId, period),
      directsActiveByLeg(userId),
      supabase.from("users").select("can_earn, sponsor_id").eq("id", userId).maybeSingle(),
      supabase.from("wallet").select("balance, total_earned").eq("user_id", userId).maybeSingle(),
      supabase.from("commissions").select("type, amount").eq("user_id", userId).eq("period", period),
      supabase.from("commissions").select("period, amount, status, created_at").eq("user_id", userId),
      supabase.from("users").select("id, full_name, referral_code, created_at").eq("sponsor_id", userId),
      supabase.from("withdrawals").select("amount, status").eq("user_id", userId),
      maxServicePointsOf(userId),
      supabase.from("memberships").select("type, program_code, points, expires_at").eq("user_id", userId).eq("status", "active"),
    ]);

  // RANGO EN VIVO: calificación con el volumen/directos del momento (no espera al cierre).
  const canEarn = user.data?.can_earn ?? false;
  const liveRank = qualifyRank(
    {
      leftTotal,
      rightTotal,
      maxLineFraction: 0,
      directsLeftActive: directsLeg.left,
      directsRightActive: directsLeg.right,
      canEarn,
    },
    ranks
  );
  const currentRank = liveRank?.level ?? 0;
  const currentRankName = liveRank?.name ?? null;
  const nextRank = ranks.find((r) => r.level === currentRank + 1) ?? null;
  const weakest = Math.min(leftTotal, rightTotal);
  const nextRankProgressPct = nextRank
    ? Math.min(100, Math.round((weakest / nextRank.points_per_leg) * 100))
    : 100;

  // Cheque del período por vía.
  const check = { direct: 0, binary: 0, rank: 0, matching: 0, lifestyle: 0, total: 0 };
  for (const c of periodComms.data ?? []) {
    const amt = Number(c.amount);
    if (c.type === "direct_l1" || c.type === "direct_l2") check.direct += amt;
    else if (c.type === "binary") check.binary += amt;
    else if (c.type === "rank") check.rank += amt;
    else if (c.type === "matching") check.matching += amt;
    else if (c.type === "lifestyle") check.lifestyle += amt;
    check.total += amt;
  }

  // Ingresos por mes (últimos 6 períodos) + por cobrar.
  const incomeByPeriod = new Map<string, number>();
  let pendingToCollect = 0;
  for (const c of allComms.data ?? []) {
    incomeByPeriod.set(c.period, (incomeByPeriod.get(c.period) ?? 0) + Number(c.amount));
    if (c.status === "validated") pendingToCollect += Number(c.amount);
  }
  const monthlyIncome = [...incomeByPeriod.entries()]
    .map(([p, amount]) => ({ period: p, amount }))
    .sort((a, b) => a.period.localeCompare(b.period))
    .slice(-6);

  // Serie diaria (YYYY-MM-DD) de ingresos: la gráfica del dashboard la reagrupa
  // por día/mes/año en el cliente. Usa la fecha real de la comisión (created_at).
  const incomeByDay = new Map<string, number>();
  for (const c of allComms.data ?? []) {
    const day = (c.created_at ?? "").slice(0, 10); // YYYY-MM-DD
    if (!day) continue;
    incomeByDay.set(day, (incomeByDay.get(day) ?? 0) + Number(c.amount));
  }
  const incomeDaily = [...incomeByDay.entries()]
    .map(([date, amount]) => ({ date, amount }))
    .sort((a, b) => a.date.localeCompare(b.date));

  // Top del equipo (directos) por comisiones del período + nuevos miembros.
  const directIds = (directs.data ?? []).map((d) => d.id);
  const earningsByUser = new Map<string, number>();
  if (directIds.length > 0) {
    const { data: dc } = await supabase
      .from("commissions")
      .select("user_id, amount")
      .eq("period", period)
      .in("user_id", directIds);
    for (const c of dc ?? []) {
      earningsByUser.set(c.user_id, (earningsByUser.get(c.user_id) ?? 0) + Number(c.amount));
    }
  }
  const directInfo = new Map((directs.data ?? []).map((d) => [d.id, d]));
  const topEarners = [...earningsByUser.entries()]
    .map(([id, amount]) => ({
      name: directInfo.get(id)?.full_name ?? "—",
      code: directInfo.get(id)?.referral_code ?? "",
      amount,
    }))
    .sort((a, b) => b.amount - a.amount)
    .slice(0, 4);

  const newMembers = (directs.data ?? [])
    .slice()
    .sort((a, b) => (a.created_at < b.created_at ? 1 : -1))
    .slice(0, 5)
    .map((d) => ({ name: d.full_name ?? "—", code: d.referral_code, date: d.created_at }));

  // Retiros por estado.
  const withdrawals = { solicitado: 0, aprobado: 0, pagado: 0, rechazado: 0 };
  for (const w of wds.data ?? []) {
    const amt = Number(w.amount);
    if (w.status === "pending") withdrawals.solicitado += amt;
    else if (w.status === "approved") withdrawals.aprobado += amt;
    else if (w.status === "paid") withdrawals.pagado += amt;
    else if (w.status === "rejected") withdrawals.rechazado += amt;
  }

  // Código del patrocinador.
  let sponsorCode: string | null = null;
  if (user.data?.sponsor_id) {
    const { data: sp } = await supabase
      .from("users")
      .select("referral_code")
      .eq("id", user.data.sponsor_id)
      .maybeSingle();
    sponsorCode = sp?.referral_code ?? null;
  }

  // Paquete activo + próxima compra (renovación más cercana).
  const services = (mems.data ?? []).filter((m) => m.type === "service");
  const topService = services.sort((a, b) => (b.points ?? 0) - (a.points ?? 0))[0];
  const packageName = topService?.program_code ?? null;
  const hasActivePackage = services.length > 0;
  const expiries = (mems.data ?? [])
    .map((m) => m.expires_at)
    .filter((e): e is string => !!e)
    .sort();
  const nextPurchaseDate = expiries[0] ?? null;

  return {
    period,
    leftTotal,
    rightTotal,
    directsActive: directsLeg.total,
    currentRank,
    currentRankName,
    nextRank,
    nextRankProgressPct,
    check,
    walletBalance: Number(wallet.data?.balance ?? 0),
    totalEarned: Number(wallet.data?.total_earned ?? 0),
    pendingToCollect,
    pvPersonal,
    pvGroup: leftTotal + rightTotal,
    sponsorCode,
    monthlyIncome,
    incomeDaily,
    topEarners,
    newMembers,
    withdrawals,
    packageName,
    nextPurchaseDate,
    hasActivePackage,
  };
}

// ----------------------------------------------------------------------------
// DASHBOARD DEL ESTUDIANTE (sin IBO) — simple
// ----------------------------------------------------------------------------
export interface StudentDashboard {
  qualifiedReferrals: number; // referidos directos que son clientes activos
  nextPurchaseDate: string | null;
}

export async function getStudentDashboard(userId: string): Promise<StudentDashboard> {
  const supabase = createAdminClient();
  const nowIso = new Date().toISOString();

  // Referidos directos.
  const { data: directs } = await supabase
    .from("users")
    .select("id")
    .eq("sponsor_id", userId);
  const ids = (directs ?? []).map((d) => d.id);

  // De esos, cuántos tienen un servicio activo vigente (clientes activos).
  let qualifiedReferrals = 0;
  if (ids.length > 0) {
    const { data: mems } = await supabase
      .from("memberships")
      .select("user_id, expires_at")
      .in("user_id", ids)
      .eq("type", "service")
      .eq("status", "active");
    const activeUsers = new Set(
      (mems ?? []).filter((m) => !m.expires_at || m.expires_at > nowIso).map((m) => m.user_id)
    );
    qualifiedReferrals = activeUsers.size;
  }

  // Próxima renovación (expira más cercano del propio usuario).
  const { data: myMems } = await supabase
    .from("memberships")
    .select("expires_at")
    .eq("user_id", userId)
    .eq("status", "active");
  const expiries = (myMems ?? [])
    .map((m) => m.expires_at)
    .filter((e): e is string => !!e)
    .sort();

  return { qualifiedReferrals, nextPurchaseDate: expiries[0] ?? null };
}

/** Puntos del servicio activo de mayor valor de un usuario. */
async function maxServicePointsOf(userId: string): Promise<number> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("memberships")
    .select("points")
    .eq("user_id", userId)
    .eq("type", "service")
    .eq("status", "active");
  return (data ?? []).reduce((m, x) => Math.max(m, x.points ?? 0), 0);
}

// ----------------------------------------------------------------------------
// MIS COMISIONES
// ----------------------------------------------------------------------------
export interface CommissionRow {
  type: CommissionType;
  amount: number;
  status: string;
  period: string;
  created_at: string;
}

export async function getCommissions(userId: string): Promise<CommissionRow[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("commissions")
    .select("type, amount, status, period, created_at")
    .eq("user_id", userId)
    .order("created_at", { ascending: false });
  return (data ?? []).map((c) => ({
    type: c.type,
    amount: Number(c.amount),
    status: c.status,
    period: c.period,
    created_at: c.created_at,
  }));
}

// ----------------------------------------------------------------------------
// LISTA DE CARTERAS (direcciones USDT guardadas)
// ----------------------------------------------------------------------------
export interface WalletAddress {
  id: string;
  address: string;
  currency: string;
  is_default: boolean;
}

export async function getWalletAddresses(userId: string): Promise<WalletAddress[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("wallet_addresses")
    .select("id, address, currency, is_default")
    .eq("user_id", userId)
    .order("is_default", { ascending: false })
    .order("created_at", { ascending: true });
  return (data ?? []) as WalletAddress[];
}

// ----------------------------------------------------------------------------
// COMISIONES DETALLADAS (con la persona origen y su paquete)
// ----------------------------------------------------------------------------
export interface EarningRow {
  type: CommissionType;
  amount: number;
  status: string;
  period: string;
  created_at: string;
  sourceName: string | null; // de quién proviene (ventas directas / matching)
  sourceCode: string | null;
  sourcePackage: string | null; // paquete con el que ingresó/compró
}

export async function getEarningsDetailed(userId: string): Promise<EarningRow[]> {
  const supabase = createAdminClient();
  const { data: comms } = await supabase
    .from("commissions")
    .select("source_user_id, type, amount, status, period, created_at")
    .eq("user_id", userId)
    .order("created_at", { ascending: false });

  const sourceIds = [...new Set((comms ?? []).map((c) => c.source_user_id).filter(Boolean) as string[])];

  const nameById = new Map<string, { name: string; code: string }>();
  const pkgById = new Map<string, string>();
  if (sourceIds.length > 0) {
    const { data: users } = await supabase
      .from("users")
      .select("id, full_name, referral_code")
      .in("id", sourceIds);
    for (const u of users ?? []) nameById.set(u.id, { name: u.full_name ?? "—", code: u.referral_code });

    const { data: mems } = await supabase
      .from("memberships")
      .select("user_id, program_code, points")
      .in("user_id", sourceIds)
      .eq("type", "service")
      .eq("status", "active");
    for (const m of mems ?? []) {
      const cur = pkgById.get(m.user_id);
      // Quedarse con el de mayor puntos (paquete principal).
      if (!cur) pkgById.set(m.user_id, m.program_code ?? "");
    }
  }

  return (comms ?? []).map((c) => {
    const src = c.source_user_id ? nameById.get(c.source_user_id) : undefined;
    return {
      type: c.type,
      amount: Number(c.amount),
      status: c.status,
      period: c.period,
      created_at: c.created_at,
      sourceName: src?.name ?? null,
      sourceCode: src?.code ?? null,
      sourcePackage: c.source_user_id ? pkgById.get(c.source_user_id) ?? null : null,
    };
  });
}

// ----------------------------------------------------------------------------
// MI RANGO — 4 condiciones en vivo hacia el siguiente rango
// ----------------------------------------------------------------------------
export interface RankCondition {
  label: string;
  current: string;
  required: string;
  met: boolean;
}
export interface RankData {
  currentRank: number;
  currentRankName: string | null;
  ranks: RankRule[];
  targetRank: RankRule | null;
  conditions: RankCondition[];
  // Valores EN VIVO (para el explorador interactivo de rangos).
  leftTotal: number;
  rightTotal: number;
  directsLeft: number;
  directsRight: number;
  canEarn: boolean;
}

export async function getRankData(userId: string): Promise<RankData> {
  const supabase = createAdminClient();
  const period = currentPeriod();
  const ranks = await loadRanks();

  const [{ leftTotal, rightTotal }, directs, user] = await Promise.all([
    liveTotals(userId, period),
    directsActiveByLeg(userId),
    supabase.from("users").select("current_rank, can_earn").eq("id", userId).maybeSingle(),
  ]);

  const canEarn = user.data?.can_earn ?? false;
  // Rango EN VIVO (no espera al cierre mensual).
  const live = qualifyRank(
    {
      leftTotal,
      rightTotal,
      maxLineFraction: 0,
      directsLeftActive: directs.left,
      directsRightActive: directs.right,
      canEarn,
    },
    ranks
  );
  const currentRank = live?.level ?? 0;
  const target = ranks.find((r) => r.level === currentRank + 1) ?? null;

  const conditions: RankCondition[] = target
    ? [
        {
          label: "Puntos pierna izquierda",
          current: `${leftTotal}`,
          required: `${target.points_per_leg}`,
          met: leftTotal >= target.points_per_leg,
        },
        {
          label: "Puntos pierna derecha",
          current: `${rightTotal}`,
          required: `${target.points_per_leg}`,
          met: rightTotal >= target.points_per_leg,
        },
        {
          label: "Directos activos (izq / der)",
          current: `${directs.left} / ${directs.right}`,
          required: `${target.min_directs_left} / ${target.min_directs_right}`,
          met: directs.left >= target.min_directs_left && directs.right >= target.min_directs_right,
        },
        {
          label: "Servicio activo para cobrar",
          current: canEarn ? "Sí" : "No",
          required: "Sí",
          met: canEarn,
        },
      ]
    : [];

  return {
    currentRank,
    currentRankName: ranks.find((r) => r.level === currentRank)?.name ?? null,
    ranks,
    targetRank: target,
    conditions,
    leftTotal,
    rightTotal,
    directsLeft: directs.left,
    directsRight: directs.right,
    canEarn,
  };
}

// ----------------------------------------------------------------------------
// BILLETERA
// ----------------------------------------------------------------------------
export interface WalletData {
  balance: number;
  totalEarned: number;
  totalWithdrawn: number;
  kycVerified: boolean;
  minWithdrawal: number;
  withdrawals: {
    amount: number;
    status: string;
    wallet_address: string;
    requested_at: string;
  }[];
}

export async function getWalletData(userId: string): Promise<WalletData> {
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();

  const [wallet, user, wds] = await Promise.all([
    supabase.from("wallet").select("balance, total_earned, total_withdrawn").eq("user_id", userId).maybeSingle(),
    supabase.from("users").select("kyc_verified").eq("id", userId).maybeSingle(),
    supabase.from("withdrawals").select("amount, status, wallet_address, requested_at").eq("user_id", userId).order("requested_at", { ascending: false }),
  ]);

  return {
    balance: Number(wallet.data?.balance ?? 0),
    totalEarned: Number(wallet.data?.total_earned ?? 0),
    totalWithdrawn: Number(wallet.data?.total_withdrawn ?? 0),
    kycVerified: user.data?.kyc_verified ?? false,
    minWithdrawal: cfg.MIN_WITHDRAWAL,
    withdrawals: (wds.data ?? []).map((w) => ({
      amount: Number(w.amount),
      status: w.status,
      wallet_address: w.wallet_address,
      requested_at: w.requested_at,
    })),
  };
}

// ----------------------------------------------------------------------------
// MIS PROGRAMAS
// ----------------------------------------------------------------------------
export interface ProgramsData {
  keys: { has_position: boolean; can_distribute: boolean; can_earn: boolean };
  memberships: { type: string; program_code: string | null; points: number; expires_at: string | null }[];
}

export async function getPrograms(userId: string): Promise<ProgramsData> {
  const supabase = createAdminClient();
  const [user, mems] = await Promise.all([
    supabase.from("users").select("has_position, can_distribute, can_earn").eq("id", userId).maybeSingle(),
    supabase.from("memberships").select("type, program_code, points, expires_at").eq("user_id", userId).eq("status", "active"),
  ]);

  return {
    keys: {
      has_position: user.data?.has_position ?? false,
      can_distribute: user.data?.can_distribute ?? false,
      can_earn: user.data?.can_earn ?? false,
    },
    memberships: (mems.data ?? []).map((m) => ({
      type: m.type,
      program_code: m.program_code,
      points: m.points,
      expires_at: m.expires_at,
    })),
  };
}

// ----------------------------------------------------------------------------
// TIENDA
// ----------------------------------------------------------------------------
export async function getCatalog() {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("products")
    .select("code, name, type, enrollment_price, points, duration_months, ai_credits_monthly, tier_level")
    .eq("is_active", true)
    .order("enrollment_price", { ascending: true });
  return (data ?? []).map((p) => ({
    code: p.code,
    name: p.name,
    type: p.type as string,
    price: Number(p.enrollment_price),
    points: p.points,
    durationMonths: p.duration_months,
    credits: Number(p.ai_credits_monthly ?? 0),
  }));
}

// ----------------------------------------------------------------------------
// PENDIENTES (por activar / por colocar) — los referidos directos del usuario
// ----------------------------------------------------------------------------
export interface PendingMember {
  id: string;
  code: string;
  name: string;
}
export interface PendingData {
  toPlace: PendingMember[]; // referidos sin posición en el árbol
  toActivate: PendingMember[]; // referidos que aún no pueden cobrar (sin activar)
}

export async function getPendingReferrals(sponsorId: string): Promise<PendingData> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("users")
    .select("id, full_name, referral_code, placement_id, can_earn")
    .eq("sponsor_id", sponsorId);

  const toPlace: PendingMember[] = [];
  const toActivate: PendingMember[] = [];
  for (const u of data ?? []) {
    const m = { id: u.id, code: u.referral_code, name: u.full_name ?? "—" };
    if (!u.placement_id) toPlace.push(m);
    if (!u.can_earn) toActivate.push(m);
  }
  return { toPlace, toActivate };
}

// ----------------------------------------------------------------------------
// MI RED — árbol navegable (carga por niveles)
// ----------------------------------------------------------------------------
export interface NetworkNode {
  id: string;
  name: string;
  rank: number;
  status: string;
  leftVolume: number;
  rightVolume: number;
  hasChildren: boolean;
}

/**
 * Verifica que `nodeId` esté dentro del subárbol de `rootId` (o sea el mismo).
 * Sirve para que un distribuidor solo pueda navegar SU propia red.
 */
export async function isWithinSubtree(
  rootId: string,
  nodeId: string,
  tree: "binary" | "sponsor"
): Promise<boolean> {
  if (rootId === nodeId) return true;
  const supabase = createAdminClient();
  const field = tree === "binary" ? "placement_id" : "sponsor_id";

  let currentId: string | null = nodeId;
  for (let d = 0; d < 100000 && currentId; d++) {
    const res = await supabase.from("users").select(field).eq("id", currentId).maybeSingle();
    const row = res.data as Record<string, string | null> | null;
    const parentId: string | null = row?.[field] ?? null;
    if (parentId === rootId) return true;
    currentId = parentId;
  }
  return false;
}

/** Nodo raíz del árbol = el propio usuario (con sus volúmenes del período). */
export async function getNetworkRoot(userId: string): Promise<NetworkNode> {
  const supabase = createAdminClient();
  const period = currentPeriod();

  const [user, ledger, binCount, sponCount] = await Promise.all([
    supabase.from("users").select("full_name, current_rank, status").eq("id", userId).maybeSingle(),
    supabase.from("volume_ledger").select("left_volume, right_volume").eq("user_id", userId).eq("period", period).maybeSingle(),
    supabase.from("users").select("*", { count: "exact", head: true }).eq("placement_id", userId),
    supabase.from("users").select("*", { count: "exact", head: true }).eq("sponsor_id", userId),
  ]);

  return {
    id: userId,
    name: user.data?.full_name ?? "Yo",
    rank: user.data?.current_rank ?? 0,
    status: user.data?.status ?? "active",
    leftVolume: ledger.data?.left_volume ?? 0,
    rightVolume: ledger.data?.right_volume ?? 0,
    hasChildren: (binCount.count ?? 0) > 0 || (sponCount.count ?? 0) > 0,
  };
}

/** Devuelve los hijos directos de `parentId` en el árbol elegido. */
export async function getNetworkChildren(
  parentId: string,
  tree: "binary" | "sponsor"
): Promise<NetworkNode[]> {
  const supabase = createAdminClient();
  const period = currentPeriod();

  const filter =
    tree === "binary"
      ? supabase.from("users").select("id, full_name, current_rank, status").eq("placement_id", parentId)
      : supabase.from("users").select("id, full_name, current_rank, status").eq("sponsor_id", parentId);

  const { data: children } = await filter;

  const nodes: NetworkNode[] = [];
  for (const c of children ?? []) {
    const { data: ledger } = await supabase
      .from("volume_ledger")
      .select("left_volume, right_volume")
      .eq("user_id", c.id)
      .eq("period", period)
      .maybeSingle();

    const { count: childCount } = await supabase
      .from("users")
      .select("*", { count: "exact", head: true })
      .eq(tree === "binary" ? "placement_id" : "sponsor_id", c.id);

    nodes.push({
      id: c.id,
      name: c.full_name ?? "—",
      rank: c.current_rank ?? 0,
      status: c.status,
      leftVolume: ledger?.left_volume ?? 0,
      rightVolume: ledger?.right_volume ?? 0,
      hasChildren: (childCount ?? 0) > 0,
    });
  }
  return nodes;
}
```

### 39. `lib/data/structure.ts`
Adaptador de la red real al formato del visor Mi Red (binario y unilevel, 5 niveles, foto, código de referido, rango).
```ts
/**
 * Adaptador entre la red real de TRIADA y el visor interactivo `MlmStructure`.
 *
 * El visor espera listas planas de miembros con enlace al padre; nuestra fuente
 * de verdad son `users.placement_id`/`binary_leg` (binario) y `users.sponsor_id`
 * (patrocinio / unilevel). Aquí se recorre desde el usuario hacia abajo y se
 * traduce al formato del componente.
 * ⚠️ SOLO SERVIDOR.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { currentPeriod } from "@/lib/engine/period";
import { rankNameFor } from "@/lib/ranks";
import type { BinaryLeg } from "@/types/database";
import type { BinaryMemberData, UnilevelMemberData, RankType } from "@/components/tree/MlmStructure";

interface RawUser {
  id: string;
  full_name: string | null;
  email: string | null;
  referral_code: string;
  current_rank: number;
  status: string;
  placement_id: string | null;
  binary_leg: BinaryLeg | null;
  sponsor_id: string | null;
  country: string | null;
  avatar_url: string | null;
  created_at: string;
}

/** Iniciales para el avatar del nodo. */
function initials(name: string): string {
  const parts = name.trim().split(/\s+/).filter(Boolean);
  if (parts.length === 0) return "??";
  if (parts.length === 1) return parts[0].slice(0, 2).toUpperCase();
  return (parts[0][0] + parts[parts.length - 1][0]).toUpperCase();
}

function rankOf(level: number): RankType {
  return (level > 0 ? rankNameFor(level) : "Sin rango") as RankType;
}

export interface StructureData {
  binary: BinaryMemberData[];
  unilevel: UnilevelMemberData[];
}

/**
 * Red del usuario hasta `maxDepth` niveles, en los dos formatos que consume el
 * visor. Se carga el universo de usuarios una vez (eficiente a esta escala) y
 * se recorta el subárbol en memoria.
 */
export async function getStructureData(rootId: string, maxDepth = 5): Promise<StructureData> {
  const supabase = createAdminClient();
  const period = currentPeriod();

  const [usersRes, ledgersRes, memsRes] = await Promise.all([
    supabase
      .from("users")
      .select(
        "id, full_name, email, referral_code, current_rank, status, placement_id, binary_leg, sponsor_id, country, avatar_url, created_at"
      ),
    supabase.from("volume_ledger").select("user_id, left_volume, right_volume").eq("period", period),
    supabase.from("memberships").select("user_id, points, type, status"),
  ]);

  const users = (usersRes.data ?? []) as RawUser[];
  const byId = new Map(users.map((u) => [u.id, u]));

  const binChildren = new Map<string, RawUser[]>();
  const sponChildren = new Map<string, RawUser[]>();
  for (const u of users) {
    if (u.placement_id && u.binary_leg) {
      const arr = binChildren.get(u.placement_id) ?? [];
      arr.push(u);
      binChildren.set(u.placement_id, arr);
    }
    if (u.sponsor_id) {
      const arr = sponChildren.get(u.sponsor_id) ?? [];
      arr.push(u);
      sponChildren.set(u.sponsor_id, arr);
    }
  }

  const ledgerMap = new Map((ledgersRes.data ?? []).map((l) => [l.user_id, l]));
  const pvMap = new Map<string, number>();
  for (const m of memsRes.data ?? []) {
    if (m.type === "service" && m.status === "active") {
      pvMap.set(m.user_id, Math.max(pvMap.get(m.user_id) ?? 0, m.points));
    }
  }

  /** Volumen agregado de un subárbol binario (memoizado). */
  const volMemo = new Map<string, number>();
  function subtreeVolume(id: string): number {
    const cached = volMemo.get(id);
    if (cached !== undefined) return cached;
    const kids = binChildren.get(id) ?? [];
    const own = pvMap.get(id) ?? 0;
    const total = own + kids.reduce((a, k) => a + subtreeVolume(k.id), 0);
    volMemo.set(id, total);
    return total;
  }

  const base = (u: RawUser, level: number, uplineName?: string) => ({
    id: u.id,
    code: u.referral_code,
    avatarUrl: u.avatar_url,
    name: u.full_name ?? u.referral_code,
    rank: rankOf(u.current_rank ?? 0),
    status: (u.status === "active" ? "Activo" : "Inactivo") as "Activo" | "Inactivo",
    level,
    uplineName,
    joinDate: (u.created_at ?? "").slice(0, 10),
    email: u.email ?? "—",
    country: u.country ?? "—",
    avatarInitials: initials(u.full_name ?? u.referral_code),
  });

  // ---- Binario ----
  const binary: BinaryMemberData[] = [];
  function walkBinary(id: string, level: number, parentId: string | null, uplineName?: string) {
    const u = byId.get(id);
    if (!u) return;
    const led = ledgerMap.get(id);
    const kids = binChildren.get(id) ?? [];
    const left = kids.find((k) => k.binary_leg === "left");
    const right = kids.find((k) => k.binary_leg === "right");
    const leftPts = Number(led?.left_volume ?? 0) || (left ? subtreeVolume(left.id) : 0);
    const rightPts = Number(led?.right_volume ?? 0) || (right ? subtreeVolume(right.id) : 0);

    binary.push({
      ...base(u, level, uplineName),
      parentId,
      leg: u.binary_leg === "left" ? "izquierda" : u.binary_leg === "right" ? "derecha" : null,
      positionLabel:
        level === 0 ? "Raíz" : u.binary_leg === "left" ? "Pierna izquierda" : "Pierna derecha",
      puntos_izquierda: leftPts,
      puntos_derecha: rightPts,
      pierna_pago: leftPts <= rightPts ? "izquierda" : "derecha",
      ciclos_binarios: Math.floor(Math.min(leftPts, rightPts) / 100),
      treeType: "binary",
    });

    if (level >= maxDepth) return;
    if (left) walkBinary(left.id, level + 1, id, u.full_name ?? undefined);
    if (right) walkBinary(right.id, level + 1, id, u.full_name ?? undefined);
  }
  walkBinary(rootId, 0, null);

  // ---- Unilevel (árbol de patrocinio) ----
  const unilevel: UnilevelMemberData[] = [];
  function walkUnilevel(id: string, level: number, parentId: string | null, uplineName?: string) {
    const u = byId.get(id);
    if (!u) return;
    const kids = sponChildren.get(id) ?? [];
    const personal = pvMap.get(id) ?? 0;

    /** Volumen del grupo de patrocinio bajo este nodo. */
    const groupVolume = (function group(nid: string): number {
      const cs = sponChildren.get(nid) ?? [];
      return (pvMap.get(nid) ?? 0) + cs.reduce((a, c) => a + group(c.id), 0);
    })(id);

    unilevel.push({
      ...base(u, level, uplineName),
      parentId,
      positionLabel: level === 0 ? "Raíz" : `Nivel ${level}`,
      volumen_grupal_puntos: groupVolume,
      volumen_personal_puntos: personal,
      lineas_directas: kids.length,
      treeType: "unilevel",
    });

    if (level >= maxDepth) return;
    for (const k of kids) walkUnilevel(k.id, level + 1, id, u.full_name ?? undefined);
  }
  walkUnilevel(rootId, 0, null);

  return { binary, unilevel };
}
```

### 40. `lib/data/tree.ts`
Subárbol binario o de patrocinio hasta N niveles.
```ts
/**
 * DATOS DEL ÁRBOL GENEALÓGICO (binario y de patrocinio).
 *
 * ⚠️ SOLO SERVIDOR. Carga el universo de usuarios UNA vez (eficiente para ~300)
 * y arma el subárbol en memoria desde `rootId` hasta `maxDepth`. Solo se envía
 * al cliente el subárbol renderizado (no toda la base).
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { currentPeriod } from "@/lib/engine/period";
import type { BinaryLeg } from "@/types/database";

export interface TreeNode {
  id: string;
  code: string; // código de referido (etiqueta del nodo)
  name: string;
  rank: number;
  status: string;
  personalPV: number; // puntos del servicio activo propio
  leftVolume: number; // volumen pierna izq del período
  rightVolume: number; // volumen pierna der del período
  leftCount: number; // total de miembros en la pierna izquierda
  rightCount: number; // total de miembros en la pierna derecha
  childrenCount: number; // referidos directos (árbol de patrocinio)
  left: TreeNode | null; // hijo binario izquierdo (dentro de maxDepth)
  right: TreeNode | null; // hijo binario derecho
  children: TreeNode[]; // hijos de patrocinio (dentro de maxDepth)
  hasMore: boolean; // tiene hijos más allá de maxDepth
}

interface RawUser {
  id: string;
  full_name: string | null;
  referral_code: string;
  current_rank: number;
  status: string;
  placement_id: string | null;
  binary_leg: BinaryLeg | null;
  sponsor_id: string | null;
}

export async function getTreeData(
  rootId: string,
  tree: "binary" | "sponsor",
  maxDepth = 3
): Promise<TreeNode | null> {
  const supabase = createAdminClient();
  const period = currentPeriod();

  const [usersRes, ledgersRes, memsRes] = await Promise.all([
    supabase
      .from("users")
      .select("id, full_name, referral_code, current_rank, status, placement_id, binary_leg, sponsor_id"),
    supabase.from("volume_ledger").select("user_id, left_volume, right_volume").eq("period", period),
    supabase.from("memberships").select("user_id, points, type, status"),
  ]);

  const users = (usersRes.data ?? []) as RawUser[];
  const byId = new Map(users.map((u) => [u.id, u]));

  const binChildren = new Map<string, { left?: RawUser; right?: RawUser }>();
  const sponChildren = new Map<string, RawUser[]>();
  for (const u of users) {
    if (u.placement_id && u.binary_leg) {
      const e = binChildren.get(u.placement_id) ?? {};
      if (u.binary_leg === "left") e.left = u;
      else e.right = u;
      binChildren.set(u.placement_id, e);
    }
    if (u.sponsor_id) {
      const arr = sponChildren.get(u.sponsor_id) ?? [];
      arr.push(u);
      sponChildren.set(u.sponsor_id, arr);
    }
  }

  const ledgerMap = new Map(
    (ledgersRes.data ?? []).map((l) => [l.user_id, l])
  );
  const pvMap = new Map<string, number>();
  for (const m of memsRes.data ?? []) {
    if (m.type === "service" && m.status === "active") {
      pvMap.set(m.user_id, Math.max(pvMap.get(m.user_id) ?? 0, m.points));
    }
  }

  // Tamaño del subárbol binario (memoizado).
  const sizeMemo = new Map<string, number>();
  function binSize(id: string): number {
    const cached = sizeMemo.get(id);
    if (cached !== undefined) return cached;
    const c = binChildren.get(id) ?? {};
    let s = 0;
    if (c.left) s += 1 + binSize(c.left.id);
    if (c.right) s += 1 + binSize(c.right.id);
    sizeMemo.set(id, s);
    return s;
  }

  function build(id: string, depth: number): TreeNode | null {
    const u = byId.get(id);
    if (!u) return null;
    const c = binChildren.get(id) ?? {};
    const spon = sponChildren.get(id) ?? [];
    const led = ledgerMap.get(id);

    const node: TreeNode = {
      id: u.id,
      code: u.referral_code,
      name: u.full_name ?? "—",
      rank: u.current_rank ?? 0,
      status: u.status,
      personalPV: pvMap.get(u.id) ?? 0,
      leftVolume: led?.left_volume ?? 0,
      rightVolume: led?.right_volume ?? 0,
      leftCount: c.left ? 1 + binSize(c.left.id) : 0,
      rightCount: c.right ? 1 + binSize(c.right.id) : 0,
      childrenCount: spon.length,
      left: null,
      right: null,
      children: [],
      hasMore: false,
    };

    if (depth < maxDepth) {
      if (tree === "binary") {
        node.left = c.left ? build(c.left.id, depth + 1) : null;
        node.right = c.right ? build(c.right.id, depth + 1) : null;
      } else {
        node.children = spon
          .map((s) => build(s.id, depth + 1))
          .filter((n): n is TreeNode => n !== null);
      }
    } else {
      node.hasMore =
        tree === "binary" ? !!(c.left || c.right) : spon.length > 0;
    }

    return node;
  }

  return build(rootId, 0);
}
```

### 41. `lib/data/store.ts`
Catálogo de tienda, pedidos del usuario y cupones.
```ts
/**
 * Catálogo y pedidos de la tienda.
 *
 * `store_orders` es el pedido con su carrito. Cuando se marca pagado se
 * "explota" en una fila de `orders` por artículo, que es lo que lee el motor de
 * comisiones — así el checkout no altera el cálculo de puntos ni del binario.
 * ⚠️ SOLO SERVIDOR.
 */
import { createAdminClient } from "@/lib/supabase/admin";

export interface StoreProduct {
  id: string;
  code: string;
  slug: string;
  name: string;
  type: string;
  description: string;
  price: number;
  compareAtPrice: number | null;
  points: number;
  durationMonths: number;
  aiCreditsMonthly: number;
  imageUrl: string | null;
  badge: string | null;
  accent: string;
  isActive: boolean;
  sort: number;
}

interface ProductRow {
  id: string;
  code: string;
  slug: string | null;
  name: string;
  type: string;
  description: string | null;
  enrollment_price: number | string;
  compare_at_price: number | string | null;
  points: number;
  duration_months: number;
  ai_credits_monthly: number | null;
  image_url: string | null;
  badge: string | null;
  accent: string | null;
  is_active: boolean;
  sort: number | null;
}

const PRODUCT_FIELDS =
  "id, code, slug, name, type, description, enrollment_price, compare_at_price, points, duration_months, ai_credits_monthly, image_url, badge, accent, is_active, sort";

function mapProduct(p: ProductRow): StoreProduct {
  return {
    id: p.id,
    code: p.code,
    slug: p.slug ?? p.code.toLowerCase(),
    name: p.name,
    type: p.type,
    description: p.description ?? "",
    price: Number(p.enrollment_price),
    compareAtPrice: p.compare_at_price === null ? null : Number(p.compare_at_price),
    points: p.points,
    durationMonths: p.duration_months,
    aiCreditsMonthly: Number(p.ai_credits_monthly ?? 0),
    imageUrl: p.image_url,
    badge: p.badge,
    accent: p.accent ?? "#C2B280",
    isActive: p.is_active,
    sort: Number(p.sort ?? 0),
  };
}

/** Catálogo visible en la tienda. */
export async function getStoreCatalog(includeHidden = false): Promise<StoreProduct[]> {
  const supabase = createAdminClient();
  let q = supabase.from("products").select(PRODUCT_FIELDS).order("sort", { ascending: true });
  if (!includeHidden) q = q.eq("is_active", true);
  const { data } = await q;
  return ((data ?? []) as ProductRow[]).map(mapProduct);
}

export async function getProductBySlug(slug: string): Promise<StoreProduct | null> {
  const supabase = createAdminClient();
  const { data } = await supabase.from("products").select(PRODUCT_FIELDS).eq("slug", slug).maybeSingle();
  return data ? mapProduct(data as ProductRow) : null;
}

export interface StoreOrderItem {
  product_id: string;
  code: string;
  name: string;
  price: number;
  points: number;
  qty: number;
  image_url: string | null;
}

export interface StoreOrder {
  id: string;
  orderNumber: string;
  items: StoreOrderItem[];
  subtotal: number;
  discount: number;
  shipping: number;
  total: number;
  couponCode: string | null;
  status: string;
  paymentStatus: string;
  paymentMethod: string | null;
  createdAt: string;
  statusHistory: { status: string; at: string; note?: string }[];
}

interface StoreOrderRow {
  id: string;
  order_number: string;
  items: StoreOrderItem[];
  subtotal: number | string;
  discount: number | string;
  shipping: number | string;
  total: number | string;
  coupon_code: string | null;
  status: string;
  payment_status: string;
  payment_method: string | null;
  created_at: string;
  status_history: { status: string; at: string; note?: string }[] | null;
}

function mapOrder(o: StoreOrderRow): StoreOrder {
  return {
    id: o.id,
    orderNumber: o.order_number,
    items: Array.isArray(o.items) ? o.items : [],
    subtotal: Number(o.subtotal),
    discount: Number(o.discount),
    shipping: Number(o.shipping),
    total: Number(o.total),
    couponCode: o.coupon_code,
    status: o.status,
    paymentStatus: o.payment_status,
    paymentMethod: o.payment_method,
    createdAt: o.created_at,
    statusHistory: Array.isArray(o.status_history) ? o.status_history : [],
  };
}

const ORDER_FIELDS =
  "id, order_number, items, subtotal, discount, shipping, total, coupon_code, status, payment_status, payment_method, created_at, status_history";

/** Pedidos del usuario, más recientes primero. */
export async function getMyStoreOrders(userId: string): Promise<StoreOrder[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("store_orders")
    .select(ORDER_FIELDS)
    .eq("user_id", userId)
    .order("created_at", { ascending: false })
    .limit(50);
  return ((data ?? []) as StoreOrderRow[]).map(mapOrder);
}

export async function getStoreOrder(userId: string, orderNumber: string): Promise<StoreOrder | null> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("store_orders")
    .select(ORDER_FIELDS)
    .eq("order_number", orderNumber)
    .eq("user_id", userId)
    .maybeSingle();
  return data ? mapOrder(data as StoreOrderRow) : null;
}

export interface AdminStoreOrder extends StoreOrder {
  userName: string;
  userEmail: string;
}

/** Todos los pedidos (panel admin). */
export async function getAllStoreOrders(limit = 100): Promise<AdminStoreOrder[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("store_orders")
    .select(`${ORDER_FIELDS}, users(full_name, email)`)
    .order("created_at", { ascending: false })
    .limit(limit);

  return (data ?? []).map((o) => {
    const rel = (o as { users?: { full_name: string | null; email: string | null } | { full_name: string | null; email: string | null }[] }).users;
    const u = Array.isArray(rel) ? rel[0] : rel;
    return {
      ...mapOrder(o as unknown as StoreOrderRow),
      userName: u?.full_name ?? "—",
      userEmail: u?.email ?? "—",
    };
  });
}

export interface StoreCoupon {
  code: string;
  type: "percent" | "fixed";
  value: number;
  minSubtotal: number;
  maxUses: number;
  uses: number;
  active: boolean;
}

export async function getCoupons(): Promise<StoreCoupon[]> {
  const supabase = createAdminClient();
  const { data } = await supabase.from("store_coupons").select("*").order("code");
  return (data ?? []).map((c) => ({
    code: c.code,
    type: c.type === "fixed" ? "fixed" : "percent",
    value: Number(c.value),
    minSubtotal: Number(c.min_subtotal),
    maxUses: c.max_uses,
    uses: c.uses,
    active: c.active,
  }));
}
```

### 42. `lib/data/countries.ts`
Lista de países.
```ts
/**
 * Países de Latinoamérica de habla hispana, con bandera, código ISO y teléfono.
 * Datos estáticos (sin servidor): se usan en el registro, perfil y dashboard.
 */
export interface Country {
  name: string;
  flag: string; // emoji (se ve en móviles; en Windows muestra el código)
  iso: string; // ISO-2 minúscula, para banderas SVG (flag-icons)
  dial: string; // código telefónico
}

export const COUNTRIES: Country[] = [
  { name: "México", flag: "🇲🇽", iso: "mx", dial: "+52" },
  { name: "Guatemala", flag: "🇬🇹", iso: "gt", dial: "+502" },
  { name: "El Salvador", flag: "🇸🇻", iso: "sv", dial: "+503" },
  { name: "Honduras", flag: "🇭🇳", iso: "hn", dial: "+504" },
  { name: "Nicaragua", flag: "🇳🇮", iso: "ni", dial: "+505" },
  { name: "Costa Rica", flag: "🇨🇷", iso: "cr", dial: "+506" },
  { name: "Panamá", flag: "🇵🇦", iso: "pa", dial: "+507" },
  { name: "Cuba", flag: "🇨🇺", iso: "cu", dial: "+53" },
  { name: "República Dominicana", flag: "🇩🇴", iso: "do", dial: "+1" },
  { name: "Puerto Rico", flag: "🇵🇷", iso: "pr", dial: "+1" },
  { name: "Colombia", flag: "🇨🇴", iso: "co", dial: "+57" },
  { name: "Venezuela", flag: "🇻🇪", iso: "ve", dial: "+58" },
  { name: "Ecuador", flag: "🇪🇨", iso: "ec", dial: "+593" },
  { name: "Perú", flag: "🇵🇪", iso: "pe", dial: "+51" },
  { name: "Bolivia", flag: "🇧🇴", iso: "bo", dial: "+591" },
  { name: "Chile", flag: "🇨🇱", iso: "cl", dial: "+56" },
  { name: "Paraguay", flag: "🇵🇾", iso: "py", dial: "+595" },
  { name: "Uruguay", flag: "🇺🇾", iso: "uy", dial: "+598" },
  { name: "Argentina", flag: "🇦🇷", iso: "ar", dial: "+54" },
];

/** Bandera emoji por nombre (fallback). */
export function countryFlag(name: string | null | undefined): string {
  if (!name) return "🌐";
  return COUNTRIES.find((c) => c.name === name)?.flag ?? "🌐";
}

/** Código ISO-2 por nombre (para banderas SVG). */
export function countryIso(name: string | null | undefined): string | null {
  if (!name) return null;
  return COUNTRIES.find((c) => c.name === name)?.iso ?? null;
}
```


---

# PARTE 5 — ESCRITURA: SERVER ACTIONS
Operaciones que modifican datos desde el panel.

### 43. `app/(distribuidor)/actions.ts`
Compra simulada, solicitud de retiro (valida KYC, mínimo y saldo), recarga de créditos y direcciones USDT.
```ts
"use server";

import { revalidatePath } from "next/cache";
import { requireUser } from "@/lib/auth/session";
import { simulateActivation } from "@/lib/domain/activation";
import { createAdminClient } from "@/lib/supabase/admin";
import { loadEngineConfig } from "@/lib/engine/config";
import { grantCredits, AFFILIATION_CREDIT_PCT } from "@/lib/ai/credits";

export interface ActivationFormState {
  error?: string;
  ok?: string;
}

/**
 * Simula la compra de un producto/programa para el usuario actual (Fase B).
 * Sin pasarela real — sirve para ver las 3 llaves activarse.
 */
export async function simulatePurchaseAction(
  _prev: ActivationFormState,
  formData: FormData
): Promise<ActivationFormState> {
  const user = await requireUser();
  const productCode = String(formData.get("productCode") ?? "");
  if (!productCode) return { error: "Falta el producto." };

  try {
    const keys = await simulateActivation(user.id, productCode);
    revalidatePath("/dashboard");
    revalidatePath("/programas");
    revalidatePath("/tienda");
    revalidatePath("/bienvenida");
    return {
      ok: `Compra de ${productCode} procesada (pago simulado). Llaves → posición:${keys.has_position ? "✓" : "✗"} distribuir:${keys.can_distribute ? "✓" : "✗"} cobrar:${keys.can_earn ? "✓" : "✗"}`,
    };
  } catch (err) {
    return { error: err instanceof Error ? err.message : "Error al procesar la compra." };
  }
}

export interface WithdrawFormState {
  error?: string;
  ok?: string;
}

/**
 * Solicita un retiro en USDT. Valida KYC, mínimo y saldo disponible.
 * Reserva (descuenta) el saldo y crea la solicitud en estado 'pending'
 * (el admin la aprueba/paga en la Fase E).
 */
export async function requestWithdrawalAction(
  _prev: WithdrawFormState,
  formData: FormData
): Promise<WithdrawFormState> {
  const user = await requireUser();
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();

  const amount = Number(formData.get("amount"));
  const address = String(formData.get("address") ?? "").trim();

  if (!address) return { error: "Ingresa tu dirección USDT." };
  if (!Number.isFinite(amount) || amount <= 0) return { error: "Monto inválido." };
  if (amount < cfg.MIN_WITHDRAWAL) {
    return { error: `El mínimo de retiro es $${cfg.MIN_WITHDRAWAL}.` };
  }

  // KYC obligatorio.
  const { data: profile } = await supabase
    .from("users")
    .select("kyc_verified")
    .eq("id", user.id)
    .maybeSingle();
  if (!profile?.kyc_verified) {
    return { error: "Debes verificar tu identidad (KYC) antes de retirar." };
  }

  // Saldo disponible.
  const { data: wallet } = await supabase
    .from("wallet")
    .select("id, balance")
    .eq("user_id", user.id)
    .maybeSingle();
  const balance = Number(wallet?.balance ?? 0);
  if (amount > balance) return { error: "El monto supera tu saldo disponible." };

  // Crear la solicitud y reservar el saldo.
  await supabase.from("withdrawals").insert({
    user_id: user.id,
    amount,
    wallet_address: address,
    status: "pending",
  });
  await supabase
    .from("wallet")
    .update({ balance: Math.round((balance - amount) * 100) / 100 })
    .eq("id", wallet!.id);

  revalidatePath("/billetera");
  return { ok: `Solicitud de retiro por $${amount} enviada. Queda pendiente de aprobación.` };
}

// ---- Recarga de créditos IA (Tienda, Fase F4) ----

export interface RechargeState {
  ok?: string;
  error?: string;
}

/**
 * Compra (simulada) de un paquete de recarga de créditos. Otorga los créditos
 * al usuario (no expiran) y un bono de afiliación en créditos al patrocinador.
 */
export async function buyCreditsAction(packId: string): Promise<RechargeState> {
  const user = await requireUser();
  const supabase = createAdminClient();

  const { data: pack } = await supabase
    .from("ia_credit_packs")
    .select("id, credits, price_usd")
    .eq("id", packId)
    .eq("is_active", true)
    .maybeSingle();
  if (!pack) return { error: "Paquete de recarga no válido." };

  const credits = Number(pack.credits);
  await grantCredits(user.id, credits, "recarga", { pack: pack.id, price_usd: pack.price_usd });

  // Bono de afiliación al patrocinador (10% de la recarga, en créditos).
  const { data: buyer } = await supabase
    .from("users")
    .select("sponsor_id")
    .eq("id", user.id)
    .maybeSingle();
  if (buyer?.sponsor_id) {
    const bonus = Math.round(credits * AFFILIATION_CREDIT_PCT);
    if (bonus > 0) {
      await grantCredits(buyer.sponsor_id, bonus, "afiliacion", {
        from_user: user.id,
        motivo: "recarga",
        pack: pack.id,
      });
    }
  }

  revalidatePath("/tienda");
  return { ok: `Recarga aplicada: +${credits.toLocaleString()} créditos.` };
}

// ---- Lista de carteras (direcciones USDT) ----

/** Crea o edita una dirección de cartera del usuario. */
export async function saveWalletAddressAction(formData: FormData): Promise<void> {
  const user = await requireUser();
  const supabase = createAdminClient();
  const id = String(formData.get("id") ?? "").trim();
  const address = String(formData.get("address") ?? "").trim();
  const currency = String(formData.get("currency") ?? "").trim();
  if (!address || !currency) return;

  if (id) {
    await supabase
      .from("wallet_addresses")
      .update({ address, currency })
      .eq("id", id)
      .eq("user_id", user.id); // solo las suyas
  } else {
    // Si es la primera, queda como predeterminada.
    const { count } = await supabase
      .from("wallet_addresses")
      .select("*", { count: "exact", head: true })
      .eq("user_id", user.id);
    await supabase.from("wallet_addresses").insert({
      user_id: user.id,
      address,
      currency,
      is_default: (count ?? 0) === 0,
    });
  }
  revalidatePath("/billetera");
}

/** Marca una cartera como la "más usada" (predeterminada). */
export async function setDefaultWalletAddressAction(id: string): Promise<void> {
  const user = await requireUser();
  const supabase = createAdminClient();
  await supabase.from("wallet_addresses").update({ is_default: false }).eq("user_id", user.id);
  await supabase.from("wallet_addresses").update({ is_default: true }).eq("id", id).eq("user_id", user.id);
  revalidatePath("/billetera");
}
```

### 44. `app/(distribuidor)/tienda/actions.ts`
Checkout del carrito (relee precios en servidor, aplica cupón, dispara el motor) y validación de cupones.
```ts
"use server";

import { revalidatePath } from "next/cache";
import { requireUser } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import { simulateActivation } from "@/lib/domain/activation";
import type { StoreOrderItem } from "@/lib/data/store";

export interface CheckoutInput {
  items: { productId: string; qty: number }[];
  couponCode?: string | null;
}

export interface CheckoutResult {
  ok?: string;
  error?: string;
  orderNumber?: string;
}

/** Número de pedido legible: TR-AAAAMMDD-XXXX. */
function makeOrderNumber(): string {
  const d = new Date();
  const ymd = `${d.getFullYear()}${String(d.getMonth() + 1).padStart(2, "0")}${String(d.getDate()).padStart(2, "0")}`;
  const rand = Math.random().toString(36).slice(2, 6).toUpperCase();
  return `TR-${ymd}-${rand}`;
}

/** Valida un cupón contra la tabla (no confía en lo que mande el navegador). */
export async function validateCouponAction(
  code: string,
  subtotal: number
): Promise<{ code: string; type: "percent" | "fixed"; value: number } | { error: string }> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("store_coupons")
    .select("code, type, value, min_subtotal, max_uses, uses, active, expires_at")
    .eq("code", code.trim().toUpperCase())
    .maybeSingle();

  if (!data || !data.active) return { error: "Cupón no válido." };
  if (data.expires_at && new Date(data.expires_at) < new Date()) return { error: "El cupón expiró." };
  if (data.uses >= data.max_uses) return { error: "El cupón alcanzó su límite de usos." };
  if (subtotal < Number(data.min_subtotal)) {
    return { error: `El cupón requiere un subtotal mínimo de $${Number(data.min_subtotal)}.` };
  }
  return {
    code: data.code,
    type: data.type === "fixed" ? "fixed" : "percent",
    value: Number(data.value),
  };
}

/**
 * Cierra la compra. Los precios y puntos se releen de la base: el carrito del
 * navegador solo aporta qué producto y cuántos.
 *
 * El pago es simulado (igual que el resto del prototipo). Al confirmarse, cada
 * artículo pasa por `simulateActivation`, que es el flujo existente: crea la
 * fila en `orders`, otorga puntos y llaves, créditos de IA y el bono al
 * patrocinador. Así la tienda nueva no duplica ninguna regla de negocio.
 */
export async function checkoutAction(input: CheckoutInput): Promise<CheckoutResult> {
  const user = await requireUser();
  const supabase = createAdminClient();

  const wanted = (input.items ?? []).filter((i) => i.qty > 0).slice(0, 50);
  if (wanted.length === 0) return { error: "Tu carrito está vacío." };

  const { data: products } = await supabase
    .from("products")
    .select("id, code, name, enrollment_price, points, image_url, is_active")
    .in(
      "id",
      wanted.map((i) => i.productId)
    );

  const byId = new Map((products ?? []).map((p) => [p.id, p]));
  const items: StoreOrderItem[] = [];
  for (const w of wanted) {
    const p = byId.get(w.productId);
    if (!p || !p.is_active) return { error: `Un producto de tu carrito ya no está disponible.` };
    items.push({
      product_id: p.id,
      code: p.code,
      name: p.name,
      price: Number(p.enrollment_price),
      points: p.points,
      qty: Math.min(99, Math.max(1, Math.floor(w.qty))),
      image_url: p.image_url,
    });
  }

  const subtotal = items.reduce((s, i) => s + i.price * i.qty, 0);

  let discount = 0;
  let couponCode: string | null = null;
  if (input.couponCode) {
    const c = await validateCouponAction(input.couponCode, subtotal);
    if (!("error" in c)) {
      discount = c.type === "percent" ? (subtotal * c.value) / 100 : c.value;
      discount = Math.min(subtotal, Math.round(discount * 100) / 100);
      couponCode = c.code;
    }
  }

  // El envío se recalcula en servidor con el tipo real del producto.
  const { data: typed } = await supabase
    .from("products")
    .select("id, type")
    .in(
      "id",
      items.map((i) => i.product_id)
    );
  const physical = (typed ?? []).some((p) => p.type === "physical");
  const shipping = physical && subtotal - discount < 120 ? 6 : 0;

  const total = Math.max(0, subtotal - discount + shipping);
  const orderNumber = makeOrderNumber();
  const now = new Date().toISOString();

  const { error: insErr } = await supabase.from("store_orders").insert({
    order_number: orderNumber,
    user_id: user.id,
    items,
    subtotal,
    discount,
    shipping,
    total,
    coupon_code: couponCode,
    status: "paid",
    payment_status: "paid",
    payment_method: "USDT (simulado)",
    status_history: [
      { status: "received", at: now },
      { status: "paid", at: now, note: "Pago simulado del prototipo" },
    ],
  });
  if (insErr) return { error: `No se pudo crear el pedido: ${insErr.message}` };

  // Disparar el motor de comisiones: una activación por unidad comprada.
  try {
    for (const item of items) {
      for (let n = 0; n < item.qty; n++) {
        await simulateActivation(user.id, item.code);
      }
    }
    await supabase
      .from("store_orders")
      .update({ commissions_applied: true, updated_at: new Date().toISOString() })
      .eq("order_number", orderNumber);
  } catch (e) {
    const msg = e instanceof Error ? e.message : "error desconocido";
    await supabase
      .from("store_orders")
      .update({ notes: `Comisiones pendientes: ${msg}`, updated_at: new Date().toISOString() })
      .eq("order_number", orderNumber);
    return { error: `El pedido se registró pero falló la activación: ${msg}` };
  }

  if (couponCode) {
    const { data: c } = await supabase
      .from("store_coupons")
      .select("uses")
      .eq("code", couponCode)
      .maybeSingle();
    if (c) await supabase.from("store_coupons").update({ uses: c.uses + 1 }).eq("code", couponCode);
  }

  revalidatePath("/tienda");
  revalidatePath("/dashboard");
  revalidatePath("/pedidos");
  return { ok: "Compra confirmada.", orderNumber };
}
```

### 45. `app/(distribuidor)/red/actions.ts`
Carga de subárbol y colocación de un referido pendiente (solo tus referidos, solo dentro de tu red).
```ts
"use server";

import { revalidatePath } from "next/cache";
import { requireUser } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import { isWithinSubtree } from "@/lib/data/distributor";
import { getTreeData, type TreeNode } from "@/lib/data/tree";
import { placeMember } from "@/lib/domain/placement";
import type { BinaryLeg } from "@/types/database";

/**
 * Carga el subárbol enraizado en `nodeId`, verificando que pertenezca a la red
 * del usuario autenticado (no puede ver redes ajenas).
 */
export async function loadTreeAction(
  nodeId: string,
  tree: "binary" | "sponsor"
): Promise<TreeNode | null> {
  const user = await requireUser();
  const allowed = await isWithinSubtree(user.id, nodeId, tree);
  if (!allowed) return null;
  return getTreeData(nodeId, tree);
}

export interface PlaceResult {
  error?: string;
}

/**
 * Coloca a un referido pendiente del distribuidor en una posición de SU árbol.
 * Reglas: el miembro debe ser su referido directo y la posición destino debe
 * ser el propio distribuidor o estar dentro de su subárbol binario.
 */
export async function placeMemberAction(
  memberId: string,
  placementId: string,
  leg: BinaryLeg
): Promise<PlaceResult> {
  const user = await requireUser();
  const supabase = createAdminClient();

  // El miembro debe ser referido directo del distribuidor.
  const { data: member } = await supabase
    .from("users")
    .select("sponsor_id")
    .eq("id", memberId)
    .maybeSingle();
  if (!member || member.sponsor_id !== user.id) {
    return { error: "Solo puedes colocar a tus propios referidos." };
  }

  // La posición destino debe ser el propio distribuidor o estar en su subárbol.
  const destOk =
    placementId === user.id ||
    (await isWithinSubtree(user.id, placementId, "binary"));
  if (!destOk) {
    return { error: "Solo puedes colocar dentro de tu propia red." };
  }

  try {
    await placeMember(memberId, placementId, leg);
  } catch (err) {
    return { error: err instanceof Error ? err.message : "No se pudo colocar." };
  }

  revalidatePath("/red");
  revalidatePath("/dashboard");
  return {};
}
```

### 46. `app/(distribuidor)/perfil/actions.ts`
Actualizar perfil y cambiar contraseña.
```ts
"use server";

import { revalidatePath } from "next/cache";
import { requireUser } from "@/lib/auth/session";
import { createClient } from "@/lib/supabase/server";
import { createAdminClient } from "@/lib/supabase/admin";
import { isPasswordValid } from "@/lib/auth/password";

export interface ProfileState {
  ok?: string;
  error?: string;
}

/** Actualiza los datos personales y de contacto del usuario. */
export async function updateProfileAction(
  _prev: ProfileState,
  formData: FormData
): Promise<ProfileState> {
  const user = await requireUser();
  const supabase = createAdminClient();

  const firstName = String(formData.get("firstName") ?? "").trim();
  const lastName = String(formData.get("lastName") ?? "").trim();
  if (!firstName || !lastName) return { error: "Nombre y apellido son obligatorios." };

  const patch: Record<string, string | null> = {
    full_name: `${firstName} ${lastName}`.trim(),
    gender: String(formData.get("gender") ?? "") || null,
    birth_date: String(formData.get("birthDate") ?? "") || null,
    country: String(formData.get("country") ?? "") || null,
    phone: String(formData.get("phone") ?? "") || null,
    address: String(formData.get("address") ?? "") || null,
    city: String(formData.get("city") ?? "") || null,
    postal_code: String(formData.get("postalCode") ?? "") || null,
  };

  // Foto de perfil (opcional): se sube a Supabase Storage al guardar.
  const avatar = formData.get("avatar");
  if (avatar instanceof File && avatar.size > 0) {
    try {
      // Asegurar el bucket público "avatars" (idempotente).
      await supabase.storage.createBucket("avatars", { public: true });
    } catch {
      /* ya existe: ignorar */
    }
    const ext = (avatar.name.split(".").pop() || "jpg").toLowerCase();
    const path = `${user.id}.${ext}`;
    const buffer = Buffer.from(await avatar.arrayBuffer());
    const { error: upErr } = await supabase.storage
      .from("avatars")
      .upload(path, buffer, { contentType: avatar.type || "image/jpeg", upsert: true });
    if (upErr) return { error: `No se pudo subir la foto: ${upErr.message}` };
    const { data: pub } = supabase.storage.from("avatars").getPublicUrl(path);
    patch.avatar_url = `${pub.publicUrl}?v=${Date.now()}`;
  }

  const { error } = await supabase.from("users").update(patch).eq("id", user.id);
  if (error) return { error: error.message };

  revalidatePath("/perfil");
  revalidatePath("/dashboard");
  return { ok: "Cambios guardados." };
}

/** Cambia la contraseña del usuario autenticado (Supabase Auth). */
export async function changePasswordAction(
  _prev: ProfileState,
  formData: FormData
): Promise<ProfileState> {
  await requireUser();
  const newPassword = String(formData.get("newPassword") ?? "");
  const confirm = String(formData.get("confirmPassword") ?? "");

  if (newPassword !== confirm) return { error: "Las contraseñas no coinciden." };
  if (!isPasswordValid(newPassword)) return { error: "La contraseña no cumple los requisitos." };

  const supabase = createClient();
  const { error } = await supabase.auth.updateUser({ password: newPassword });
  if (error) return { error: error.message };

  return { ok: "Contraseña actualizada." };
}
```

### 47. `app/(distribuidor)/ia/actions.ts`
Conversaciones del chat (listar, abrir, renombrar, borrar) y publicar/despublicar una creación.
```ts
"use server";

import { requireUser } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import {
  getConversations,
  getConversationMessages,
  type AiConversationSummary,
  type AiChatMessage,
} from "@/lib/ai/hub";

/** Lista de conversaciones del usuario. */
export async function listConversationsAction(): Promise<AiConversationSummary[]> {
  const user = await requireUser();
  return getConversations(user.id);
}

/** Mensajes de una conversación (verifica propiedad). */
export async function loadConversationAction(id: string): Promise<AiChatMessage[]> {
  const user = await requireUser();
  return getConversationMessages(user.id, id);
}

/** Renombra una conversación (solo la del usuario). */
export async function renameConversationAction(id: string, title: string): Promise<void> {
  const user = await requireUser();
  const supabase = createAdminClient();
  const clean = title.trim().slice(0, 80);
  await supabase
    .from("ai_conversations")
    .update({ title: clean || null })
    .eq("id", id)
    .eq("user_id", user.id);
}

/** Borra una conversación (y sus mensajes por cascade). */
export async function deleteConversationAction(id: string): Promise<void> {
  const user = await requireUser();
  const supabase = createAdminClient();
  await supabase.from("ai_conversations").delete().eq("id", id).eq("user_id", user.id);
}

/** Marca/desmarca una creación como pública (aparece en Tendencias). */
export async function toggleCreationPublicAction(id: string, isPublic: boolean): Promise<void> {
  const user = await requireUser();
  const supabase = createAdminClient();
  await supabase
    .from("ai_creations")
    .update({ is_public: isPublic })
    .eq("id", id)
    .eq("user_id", user.id);
}

/** Suma una vista a una creación pública (al abrir su detalle en Tendencias). */
export async function bumpCreationViewAction(id: string): Promise<void> {
  const supabase = createAdminClient();
  const { data } = await supabase.from("ai_creations").select("views").eq("id", id).maybeSingle();
  if (data) {
    await supabase
      .from("ai_creations")
      .update({ views: Number(data.views ?? 0) + 1 })
      .eq("id", id);
  }
}
```


---

# PARTE 6 — PANTALLAS DEL DISTRIBUIDOR
Cada archivo es una página Server Component.

### 48. `app/(distribuidor)/dashboard/page.tsx`
**Dashboard.** Decide por IBO: vista de distribuidor (perfil, métricas, rango con insignia, gráfico de ingresos, equipo, ganancias por vía, renovación, link de invitación, retiros, nuevos miembros) o de estudiante.
```tsx
import Link from "next/link";
import { requireUser } from "@/lib/auth/session";
import { getDashboardData, getPendingReferrals, getStudentDashboard } from "@/lib/data/distributor";
import { Card } from "@/components/ui/Card";
import { ReferralLinks } from "@/components/dashboard/ReferralLinks";
import { ProfileHeader } from "@/components/dashboard/ProfileHeader";
import { CountdownTimer } from "@/components/dashboard/CountdownTimer";
import { ThreeFreeWidget } from "@/components/dashboard/ThreeFreeWidget";
import { IncomeChart } from "@/components/dashboard/IncomeChart";
import { formatUSD } from "@/lib/utils";
import { Wallet, Coins, TrendingUp, Clock, Users } from "lucide-react";
import { RankBadge } from "@/components/ui/RankBadge";
import { getT } from "@/lib/i18n/server";
import type { User } from "@/types/database";

export const metadata = { title: "Dashboard — TRIADA" };

const VIAS = [
  { key: "direct", viaKey: "via.direct" },
  { key: "binary", viaKey: "via.binary" },
  { key: "rank", viaKey: "via.rank" },
  { key: "matching", viaKey: "via.matching" },
  { key: "lifestyle", viaKey: "via.lifestyle" },
] as const;

function greeting(gender: string | null, fullName: string | null, t: (k: string) => string): string {
  const first = fullName?.split(" ")[0] ?? "";
  const word = gender === "F" ? t("dash.greetingF") : gender === "M" ? t("dash.greetingM") : t("dash.greetingX");
  return `${word}, ${first}`;
}

export default async function DashboardPage() {
  const user = await requireUser();
  // El sistema decide qué dashboard mostrar según el IBO (can_distribute).
  if (user.can_distribute) return <DistributorView user={user} />;
  return <StudentView user={user} />;
}

// ----------------------------------------------------------------------------
// DASHBOARD DEL ESTUDIANTE (sin IBO): simple
// ----------------------------------------------------------------------------
async function StudentView({ user }: { user: User }) {
  const s = await getStudentDashboard(user.id);
  const t = getT();

  return (
    <div className="mx-auto max-w-2xl space-y-6">
      <div>
        <p className="label">{t("nav.dashboard")}</p>
        <h1 className="mt-1 text-2xl font-extrabold text-cream">
          {greeting(user.gender, user.full_name, t)}
        </h1>
        <p className="mt-1 text-sm text-muted">
          {t("dash.backofficeId")}: <span className="font-semibold text-gold">{user.referral_code}</span>
        </p>
      </div>

      <CountdownTimer targetIso={s.nextPurchaseDate} />

      <ThreeFreeWidget
        qualified={s.qualifiedReferrals}
        referralCode={user.referral_code}
        nextRenewal={s.nextPurchaseDate}
      />

      <p className="text-center text-xs text-muted">
        <Link href="/tienda" className="text-gold">{t("dash.unlockTip")}</Link>
      </p>
    </div>
  );
}

// ----------------------------------------------------------------------------
// DASHBOARD DEL DISTRIBUIDOR (con IBO): panel completo
// ----------------------------------------------------------------------------
async function DistributorView({ user }: { user: User }) {
  const [d, pending] = await Promise.all([
    getDashboardData(user.id),
    getPendingReferrals(user.id),
  ]);
  const t = getT();

  return (
    <div className="space-y-6">
      <div>
        <p className="label">{t("group.business")} · {d.period}</p>
        <h1 className="mt-1 text-2xl font-extrabold text-cream">
          {greeting(user.gender, user.full_name, t)}
        </h1>
      </div>

      {/* Encabezado de perfil */}
      <ProfileHeader
        name={user.full_name ?? "Distribuidor"}
        code={user.referral_code}
        country={user.country}
        registeredAt={user.created_at}
        statusActive={d.hasActivePackage}
        binaryActive={user.can_earn}
        packageName={d.packageName}
        nextPurchaseDate={d.nextPurchaseDate}
        avatarUrl={user.avatar_url}
      />

      {pending.toPlace.length > 0 && (
        <div className="flex items-center justify-between rounded-card border border-cobalt/40 bg-cobalt/10 px-4 py-3">
          <p className="text-sm text-cream">
            {t("dash.youHave")} <span className="font-bold text-cobalt">{pending.toPlace.length}</span>{" "}
            {t("dash.pendingToPlace")}
          </p>
          <Link href="/red" className="rounded-md border border-gold px-3 py-1.5 text-xs font-semibold text-gold hover:bg-gold/10">
            {t("dash.placeNow")}
          </Link>
        </div>
      )}

      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <MiniStat icon={<Wallet size={18} />} label={t("dash.wallet")} value={formatUSD(d.walletBalance)} />
        <MiniStat icon={<Coins size={18} />} label={t("dash.commissionMonth")} value={formatUSD(d.check.total)} accent />
        <MiniStat icon={<TrendingUp size={18} />} label={t("dash.totalEarned")} value={formatUSD(d.totalEarned)} />
        <MiniStat icon={<Clock size={18} />} label={t("dash.toCollect")} value={formatUSD(d.pendingToCollect)} />
      </div>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-3">
        <div className="space-y-6 lg:col-span-2">
          <Card premium>
            <div className="flex flex-col items-center text-center">
              <RankBadge level={d.currentRank} size="lg" />
              <p className="label mt-3">{t("dash.classification")}</p>
              <p className="font-display text-2xl font-extrabold text-cream">
                {d.currentRankName ?? t("dash.noRank")}
              </p>
              <Link href="/rango" className="mt-1 text-sm text-gold hover:text-goldLight">
                {t("dash.climb")}
              </Link>
            </div>
            {d.nextRank && (
              <>
                <div className="mt-4 h-2 w-full overflow-hidden rounded-full bg-border">
                  <div className="h-full rounded-full bg-gold" style={{ width: `${d.nextRankProgressPct}%` }} />
                </div>
                <p className="mt-2 text-center text-xs text-muted">
                  {d.nextRankProgressPct}% {t("dash.nextRankTo")} {d.nextRank.name}
                </p>
              </>
            )}
          </Card>

          <div className="grid grid-cols-1 gap-6 sm:grid-cols-2">
            <Card>
              <h2 className="mb-3 text-base font-bold text-cream">{t("dash.teamPerformance")}</h2>
              {d.topEarners.length === 0 ? (
                <p className="text-sm text-muted">{t("dash.noEarners")}</p>
              ) : (
                <ul className="space-y-3">
                  {d.topEarners.map((e, i) => (
                    <li key={i} className="flex items-center justify-between">
                      <div className="flex items-center gap-2">
                        <span className="flex h-7 w-7 items-center justify-center rounded-full bg-navyDeep">
                          <Users size={13} className="text-muted" />
                        </span>
                        <div>
                          <p className="text-sm text-cream">{e.name}</p>
                          <p className="text-[11px] text-muted">{e.code}</p>
                        </div>
                      </div>
                      <span className="text-sm font-semibold text-success">{formatUSD(e.amount)}</span>
                    </li>
                  ))}
                </ul>
              )}
            </Card>

            <Card>
              <h2 className="mb-3 text-base font-bold text-cream">{t("dash.earningsByVia")}</h2>
              <ul className="space-y-2">
                {VIAS.map(({ key, viaKey }) => (
                  <li key={key} className="flex items-center justify-between text-sm">
                    <span className="text-muted">{t(viaKey)}</span>
                    <span className="font-medium text-cream">{formatUSD(d.check[key])}</span>
                  </li>
                ))}
                <li className="mt-1 flex items-center justify-between border-t border-border pt-2 text-sm">
                  <span className="font-semibold text-cream">{t("common.total")}</span>
                  <span className="font-semibold text-gold">{formatUSD(d.check.total)}</span>
                </li>
              </ul>
            </Card>
          </div>

          <Card>
            <IncomeChart daily={d.incomeDaily} emptyLabel={t("dash.noIncome")} />
          </Card>
        </div>

        <div className="space-y-6">
          <Card premium>
            <div className="mt-1 grid grid-cols-2 gap-3 text-sm">
              <Mini label={t("net.pvPersonal")} value={String(d.pvPersonal)} />
              <Mini label="PV grupal" value={String(d.pvGroup)} />
              <Mini label={t("net.left")} value={String(d.leftTotal)} />
              <Mini label={t("net.right")} value={String(d.rightTotal)} />
            </div>
            <p className="mt-3 text-xs text-muted">
              {t("dash.sponsor")}: <span className="text-cream">{d.sponsorCode ?? "—"}</span>
            </p>
          </Card>

          <Card>
            <ReferralLinks referralCode={user.referral_code} />
          </Card>

          <Card>
            <h2 className="mb-3 text-base font-bold text-cream">{t("dash.withdrawal")}</h2>
            <ul className="space-y-2 text-sm">
              <RetiroRow label={t("dash.wdRequested")} value={d.withdrawals.solicitado} color="text-gold" />
              <RetiroRow label={t("dash.wdApproved")} value={d.withdrawals.aprobado} color="text-cobalt" />
              <RetiroRow label={t("dash.wdPaid")} value={d.withdrawals.pagado} color="text-success" />
              <RetiroRow label={t("dash.wdRejected")} value={d.withdrawals.rechazado} color="text-danger" />
            </ul>
          </Card>

          <Card>
            <h2 className="mb-3 text-base font-bold text-cream">{t("dash.newMembers")}</h2>
            {d.newMembers.length === 0 ? (
              <p className="text-sm text-muted">{t("dash.noMembers")}</p>
            ) : (
              <ul className="space-y-2">
                {d.newMembers.map((m, i) => (
                  <li key={i} className="flex items-center justify-between text-sm">
                    <div>
                      <p className="text-cream">{m.name}</p>
                      <p className="text-[11px] text-muted">{m.code}</p>
                    </div>
                    <span className="text-[11px] text-muted">
                      {new Date(m.date).toLocaleDateString("es")}
                    </span>
                  </li>
                ))}
              </ul>
            )}
          </Card>
        </div>
      </div>

      {/* Temporizador de renovación (al final) */}
      <CountdownTimer targetIso={d.nextPurchaseDate} />
    </div>
  );
}

function MiniStat({ icon, label, value, accent = false }: { icon: React.ReactNode; label: string; value: string; accent?: boolean }) {
  return (
    <Card>
      <div className="flex items-center justify-between">
        <p className="label">{label}</p>
        <span className={`flex h-8 w-8 items-center justify-center rounded-lg ${accent ? "bg-gold/15 text-gold" : "bg-navyDeep text-cobalt"}`}>
          {icon}
        </span>
      </div>
      <p className={`mt-2 font-display text-2xl font-extrabold ${accent ? "text-gold" : "text-cream"}`}>{value}</p>
    </Card>
  );
}

function Mini({ label, value }: { label: string; value: string }) {
  return (
    <div className="rounded-lg bg-navyDeep p-2">
      <p className="text-[10px] uppercase tracking-wide text-muted">{label}</p>
      <p className="font-display text-lg font-extrabold text-gold">{value}</p>
    </div>
  );
}

function RetiroRow({ label, value, color }: { label: string; value: number; color: string }) {
  return (
    <li className="flex items-center justify-between">
      <span className="text-muted">{label}</span>
      <span className={`font-medium ${color}`}>{formatUSD(value)}</span>
    </li>
  );
}
```

### 49. `app/(distribuidor)/programas/page.tsx`
**Mis Programas.** Programas activos y vigencia.
```tsx
import Link from "next/link";
import { requireUser } from "@/lib/auth/session";
import { getPrograms } from "@/lib/data/distributor";
import { getT } from "@/lib/i18n/server";
import { Card } from "@/components/ui/Card";
import { Button } from "@/components/ui/Button";

export const metadata = { title: "Mis Programas — TRIADA" };

function KeyState({ on, title, desc, onLabel, offLabel }: { on: boolean; title: string; desc: string; onLabel: string; offLabel: string }) {
  return (
    <div className={`rounded-card border p-4 ${on ? "border-gold/40 bg-gold/5" : "border-border bg-navyDeep"}`}>
      <div className="flex items-center justify-between">
        <span className="font-display text-sm font-semibold text-cream">{title}</span>
        <span
          className={`rounded-full px-2 py-0.5 text-[10px] font-semibold uppercase tracking-wide ${
            on ? "bg-success/15 text-success" : "bg-muted/15 text-muted"
          }`}
        >
          {on ? onLabel : offLabel}
        </span>
      </div>
      <p className="mt-1 text-xs text-muted">{desc}</p>
    </div>
  );
}

export default async function ProgramasPage() {
  const user = await requireUser();
  const p = await getPrograms(user.id);
  const t = getT();
  const onL = t("prog.active");
  const offL = t("prog.inactive");

  return (
    <div className="space-y-8">
      <div>
        <p className="label">{t("nav.programs")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("prog.title")}</h1>
      </div>

      <section>
        <h2 className="mb-3 text-lg font-bold text-cream">{t("prog.keys")}</h2>
        <div className="grid grid-cols-1 gap-3 sm:grid-cols-3">
          <KeyState on={p.keys.has_position} title={t("prog.keyPosition")} desc={t("prog.keyPositionDesc")} onLabel={onL} offLabel={offL} />
          <KeyState on={p.keys.can_distribute} title={t("prog.keyDistribute")} desc={t("prog.keyDistributeDesc")} onLabel={onL} offLabel={offL} />
          <KeyState on={p.keys.can_earn} title={t("prog.keyEarn")} desc={t("prog.keyEarnDesc")} onLabel={onL} offLabel={offL} />
        </div>
      </section>

      <Card>
        <div className="mb-3 flex items-center justify-between">
          <h2 className="text-lg font-bold text-cream">{t("prog.activePrograms")}</h2>
          <Link href="/tienda">
            <Button variant="secondary" className="px-5 py-2 text-xs">
              {t("prog.renew")}
            </Button>
          </Link>
        </div>
        {p.memberships.length === 0 ? (
          <p className="text-sm text-muted">{t("prog.none")}</p>
        ) : (
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">{t("prog.program")}</th>
                <th className="pb-2 font-semibold">{t("prog.type")}</th>
                <th className="pb-2 font-semibold">{t("prog.points")}</th>
                <th className="pb-2 font-semibold">{t("prog.expires")}</th>
              </tr>
            </thead>
            <tbody>
              {p.memberships.map((m, i) => (
                <tr key={i} className="border-b border-border/50">
                  <td className="py-2.5 text-cream">{m.program_code ?? "—"}</td>
                  <td className="py-2.5 text-muted">{m.type === "ibo" ? t("prog.typeIbo") : t("prog.typeService")}</td>
                  <td className="py-2.5 text-cream">{m.points}</td>
                  <td className="py-2.5 text-muted">
                    {m.expires_at ? new Date(m.expires_at).toLocaleDateString("es") : "—"}
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        )}
      </Card>
    </div>
  );
}
```

### 50. `app/(distribuidor)/tienda/page.tsx`
**Tienda.** Catálogo e-commerce más la tienda de recarga de créditos de IA.
```tsx
import { getStoreCatalog } from "@/lib/data/store";
import { getCreditPacks, getAiBalance } from "@/lib/ai/hub";
import { requireUser } from "@/lib/auth/session";
import { StoreClient } from "@/components/tienda/StoreClient";
import { CreditStore } from "@/components/dashboard/CreditStore";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Tienda — TRIADA" };

export default async function TiendaPage() {
  const user = await requireUser();
  const [catalog, packs, balance] = await Promise.all([
    getStoreCatalog(),
    getCreditPacks(),
    getAiBalance(user.id),
  ]);
  const t = getT();

  return (
    <div className="space-y-10">
      <div>
        <p className="label">{t("nav.store")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("store.title")}</h1>
        <p className="mt-1 text-sm text-muted">{t("store.subtitle")}</p>
      </div>

      <StoreClient products={catalog} />

      <CreditStore packs={packs} balance={balance} />
    </div>
  );
}
```

### 51. `app/(distribuidor)/pedidos/page.tsx`
**Mis Pedidos.** Historial con ítems, descuento, cupón y estado.
```tsx
import { requireUser } from "@/lib/auth/session";
import { getMyStoreOrders } from "@/lib/data/store";
import { Card } from "@/components/ui/Card";
import { formatUSD } from "@/lib/utils";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Mis Pedidos — TRIADA" };

const STATUS_LABEL: Record<string, string> = {
  received: "Recibido",
  paid: "Pagado",
  preparing: "Preparando",
  shipped: "Enviado",
  delivered: "Entregado",
  cancelled: "Cancelado",
};

const STATUS_STYLE: Record<string, string> = {
  received: "bg-muted/15 text-muted",
  paid: "bg-success/15 text-success",
  preparing: "bg-gold/15 text-gold",
  shipped: "bg-cobalt/20 text-cream",
  delivered: "bg-success/20 text-success",
  cancelled: "bg-danger/15 text-danger",
};

export default async function PedidosPage() {
  const user = await requireUser();
  const t = getT();
  const orders = await getMyStoreOrders(user.id);

  return (
    <div className="space-y-6">
      <div>
        <p className="label">{t("nav.store")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("store.orders")}</h1>
      </div>

      {orders.length === 0 ? (
        <Card>
          <p className="py-8 text-center text-sm text-muted">Aún no has hecho ningún pedido.</p>
        </Card>
      ) : (
        <div className="space-y-3">
          {orders.map((o) => (
            <Card key={o.id}>
              <div className="flex flex-wrap items-center justify-between gap-2">
                <div>
                  <p className="font-mono text-sm text-cream">{o.orderNumber}</p>
                  <p className="text-xs text-muted">{new Date(o.createdAt).toLocaleString()}</p>
                </div>
                <div className="flex items-center gap-2">
                  <span className="text-lg font-bold text-cream">{formatUSD(o.total)}</span>
                  <span
                    className={`rounded-full px-2.5 py-0.5 text-[10px] font-semibold ${
                      STATUS_STYLE[o.status] ?? "bg-muted/15 text-muted"
                    }`}
                  >
                    {STATUS_LABEL[o.status] ?? o.status}
                  </span>
                </div>
              </div>

              <ul className="mt-3 space-y-2 border-t border-border pt-3">
                {o.items.map((i, idx) => (
                  <li key={idx} className="flex items-center gap-3">
                    <div className="h-12 w-12 shrink-0 overflow-hidden rounded-lg bg-navyDeep">
                      {i.image_url && (
                        // eslint-disable-next-line @next/next/no-img-element
                        <img src={i.image_url} alt={i.name} className="h-full w-full object-cover" />
                      )}
                    </div>
                    <div className="min-w-0 flex-1">
                      <p className="truncate text-sm text-cream">{i.name}</p>
                      <p className="text-[11px] text-muted">
                        {i.qty}× {formatUSD(i.price)} · {i.points * i.qty} pts
                      </p>
                    </div>
                  </li>
                ))}
              </ul>

              {(o.discount > 0 || o.shipping > 0) && (
                <p className="mt-2 text-[11px] text-muted">
                  {t("store.subtotal")} {formatUSD(o.subtotal)}
                  {o.discount > 0 && ` · ${t("store.discount")} −${formatUSD(o.discount)}`}
                  {o.couponCode && ` (${o.couponCode})`}
                  {o.shipping > 0 && ` · ${t("store.shipping")} ${formatUSD(o.shipping)}`}
                </p>
              )}
            </Card>
          ))}
        </div>
      )}
    </div>
  );
}
```

### 52. `app/(distribuidor)/proyectos/page.tsx`
**Mis Proyectos.** Galería de lo generado con IA.
```tsx
import { requireUser } from "@/lib/auth/session";
import { getCreations } from "@/lib/ai/creative";
import { ProjectsGrid } from "@/components/dashboard/ai/ProjectsGrid";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Mis Proyectos — TRIADA" };

export default async function ProyectosPage() {
  const user = await requireUser();
  const t = getT();
  const creations = await getCreations(user.id, 120);

  return (
    <div className="space-y-6">
      <div>
        <p className="label">{t("group.center")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("nav.projects")}</h1>
        <p className="mt-1 text-sm text-muted">{t("projects.subtitle")}</p>
      </div>

      <ProjectsGrid creations={creations} />
    </div>
  );
}
```

### 53. `app/(distribuidor)/ia/page.tsx`
**Hub Triada.** Carga catálogo de modelos por categoría, agentes, saldo, conversaciones, creaciones, capacidades de adjunto y el resumen de actividad, y los pasa al orquestador.
```tsx
import { requireUser } from "@/lib/auth/session";
import {
  loadAiModels,
  loadAssistants,
  getAiBalance,
  getConversations,
  getUserTier,
} from "@/lib/ai/hub";
import { getCreations, getPublicCreations } from "@/lib/ai/creative";
import { getAutoModeCapabilities } from "@/lib/ai/router";
import { getActivity } from "@/lib/ai/activity";
import { AiHub } from "@/components/dashboard/ai/AiHub";

export const metadata = { title: "Hub Triada — TRIADA" };

export default async function IaPage() {
  const user = await requireUser();
  const [
    chatModels,
    codeModels,
    imageModels,
    audioModels,
    assistants,
    balance,
    conversations,
    creations,
    publicCreations,
    autoCaps,
    codeCaps,
    activity,
    tier,
  ] = await Promise.all([
    loadAiModels("chat"),
    loadAiModels("code"),
    loadAiModels("image"),
    loadAiModels("audio"),
    loadAssistants(),
    getAiBalance(user.id),
    getConversations(user.id),
    getCreations(user.id),
    getPublicCreations(),
    getAutoModeCapabilities("chat"),
    getAutoModeCapabilities("code"),
    getActivity(user.id),
    getUserTier(user.id),
  ]);

  return (
    <AiHub
      userName={user.full_name ?? user.email}
      userAvatarUrl={user.avatar_url}
      userTier={tier}
      chatModels={chatModels}
      codeModels={codeModels}
      imageModels={imageModels}
      audioModels={audioModels}
      assistants={assistants}
      autoCaps={autoCaps}
      codeCaps={codeCaps}
      activity={activity}
      conversations={conversations}
      creations={creations}
      publicCreations={publicCreations}
      initialBalance={balance}
    />
  );
}
```

### 54. `app/(distribuidor)/red/page.tsx`
**Mi Red.** Visor interactivo con pendientes por colocar.
```tsx
import { requireUser } from "@/lib/auth/session";
import { getStructureData } from "@/lib/data/structure";
import { getPendingReferrals } from "@/lib/data/distributor";
import { Card } from "@/components/ui/Card";
import { MlmStructureClient } from "@/components/tree/MlmStructureClient";
import { ReferralLinks } from "@/components/dashboard/ReferralLinks";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Mi Red — TRIADA" };

export default async function RedPage() {
  const user = await requireUser();
  const t = getT();
  const [structure, pending] = await Promise.all([
    getStructureData(user.id),
    getPendingReferrals(user.id),
  ]);

  return (
    <div className="space-y-8">
      <div>
        <p className="label">{t("nav.network")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("net.title")}</h1>
        <p className="mt-1 text-sm text-muted">{t("net.subtitle")}</p>
      </div>

      {/* Pendientes por colocar */}
      {pending.toPlace.length > 0 && (
        <Card>
          <h2 className="text-sm font-bold text-cream">
            {t("dash.youHave")} {pending.toPlace.length} {t("net.pendingCount")}
          </h2>
          <p className="mt-1 text-xs text-muted">
            {pending.toPlace.map((m) => m.code).join(", ")}. {t("net.pendingTip")}
          </p>
        </Card>
      )}

      <MlmStructureClient
        binary={structure.binary}
        unilevel={structure.unilevel}
        rootId={user.id}
        pending={pending.toPlace.map((m) => ({ id: m.id, code: m.code, name: m.name }))}
      />

      {user.can_distribute && (
        <Card>
          <ReferralLinks referralCode={user.referral_code} />
        </Card>
      )}
    </div>
  );
}
```

### 55. `app/(distribuidor)/comisiones/page.tsx`
**Mis Comisiones.** Ganancias por vía.
```tsx
import { requireUser } from "@/lib/auth/session";
import { getEarningsDetailed, type EarningRow } from "@/lib/data/distributor";
import { getT } from "@/lib/i18n/server";
import { Card } from "@/components/ui/Card";
import { formatUSD } from "@/lib/utils";

export const metadata = { title: "Mis Comisiones — TRIADA" };

const VIA_KEY: Record<string, string> = {
  direct_l1: "com.directL1",
  direct_l2: "com.directL2",
  binary: "com.binary",
  rank: "com.rank",
  matching: "com.matching",
  lifestyle: "com.lifestyle",
};
const STATUS_STYLE: Record<string, string> = {
  pending: "bg-gold/15 text-gold",
  validated: "bg-cobalt/15 text-cobalt",
  paid: "bg-success/15 text-success",
  rejected: "bg-danger/15 text-danger",
};
const STATUS_KEY: Record<string, string> = {
  pending: "st.pending",
  validated: "st.validated",
  paid: "st.paid",
  rejected: "st.rejected",
};

export default async function ComisionesPage() {
  const user = await requireUser();
  const rows = await getEarningsDetailed(user.id);
  const t = getT();

  const byPeriod = new Map<string, EarningRow[]>();
  for (const r of rows) {
    if (!byPeriod.has(r.period)) byPeriod.set(r.period, []);
    byPeriod.get(r.period)!.push(r);
  }

  return (
    <div className="space-y-8">
      <div>
        <p className="label">{t("nav.commissions")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("com.title")}</h1>
        <p className="mt-1 text-sm text-muted">{t("com.subtitle")}</p>
      </div>

      {byPeriod.size === 0 && (
        <Card>
          <p className="text-sm text-muted">{t("com.none")}</p>
        </Card>
      )}

      {[...byPeriod.entries()]
        .sort((a, b) => b[0].localeCompare(a[0]))
        .map(([period, items]) => {
          const total = items.reduce((s, i) => s + i.amount, 0);
          return (
            <Card key={period}>
              <div className="mb-3 flex items-center justify-between">
                <h2 className="text-lg font-bold text-cream">{period}</h2>
                <span className="text-sm text-gold">{formatUSD(total)}</span>
              </div>
              <div className="overflow-x-auto">
                <table className="w-full text-sm">
                  <thead>
                    <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                      <th className="pb-2 font-semibold">{t("com.colVia")}</th>
                      <th className="pb-2 font-semibold">{t("com.colOrigin")}</th>
                      <th className="pb-2 font-semibold">{t("com.colPackage")}</th>
                      <th className="pb-2 font-semibold">{t("com.colAmount")}</th>
                      <th className="pb-2 font-semibold">{t("com.colStatus")}</th>
                    </tr>
                  </thead>
                  <tbody>
                    {items.map((r, i) => (
                      <tr key={i} className="border-b border-border/50">
                        <td className="py-2.5 text-cream">{t(VIA_KEY[r.type] ?? r.type)}</td>
                        <td className="py-2.5 text-muted">
                          {r.sourceName ? (
                            <>
                              <span className="text-gold">{r.sourceCode}</span> · {r.sourceName}
                            </>
                          ) : (
                            <span className="text-muted/60">{t("com.networkVolume")}</span>
                          )}
                        </td>
                        <td className="py-2.5 text-muted">{r.sourcePackage ?? "—"}</td>
                        <td className="py-2.5 font-medium text-cream">{formatUSD(r.amount)}</td>
                        <td className="py-2.5">
                          <span className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${STATUS_STYLE[r.status] ?? ""}`}>
                            {t(STATUS_KEY[r.status] ?? r.status)}
                          </span>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </Card>
          );
        })}
    </div>
  );
}
```

### 56. `app/(distribuidor)/rango/page.tsx`
**Mi Rango.** Progreso hacia el siguiente rango.
```tsx
import { requireUser } from "@/lib/auth/session";
import { getRankData } from "@/lib/data/distributor";
import { getT } from "@/lib/i18n/server";
import { RankExplorer } from "@/components/dashboard/RankExplorer";
import { RankBadge } from "@/components/ui/RankBadge";

export const metadata = { title: "Mi Rango — TRIADA" };

export default async function RangoPage() {
  const user = await requireUser();
  const d = await getRankData(user.id);
  const t = getT();

  return (
    <div className="space-y-6">
      <div className="flex items-center gap-4">
        <RankBadge level={d.currentRank} size="lg" />
        <div>
          <p className="label">{t("rank.title")}</p>
          <h1 className="mt-1 text-3xl font-extrabold text-cream">
            {d.currentRankName ?? t("rank.noRank")}
          </h1>
          <p className="mt-1 text-sm text-muted">
            {d.currentRank > 0
              ? t("rank.rankOf").replace("{n}", String(d.currentRank))
              : t("rank.startTip")}
          </p>
        </div>
      </div>

      <RankExplorer
        ranks={d.ranks}
        currentRank={d.currentRank}
        leftTotal={d.leftTotal}
        rightTotal={d.rightTotal}
        directsLeft={d.directsLeft}
        directsRight={d.directsRight}
        canEarn={d.canEarn}
      />

      <p className="text-xs uppercase tracking-label text-muted">
        IA · Trading · Finanzas Digitales
      </p>
    </div>
  );
}
```

### 57. `app/(distribuidor)/billetera/page.tsx`
**Billetera.** Saldo, direcciones USDT, retiros e historial.
```tsx
import { requireUser } from "@/lib/auth/session";
import {
  getWalletData,
  getWalletAddresses,
  getEarningsDetailed,
} from "@/lib/data/distributor";
import { Card, StatCard } from "@/components/ui/Card";
import { WithdrawForm } from "@/components/dashboard/WithdrawForm";
import { WalletAddresses } from "@/components/dashboard/WalletAddresses";
import { EarningsList } from "@/components/dashboard/EarningsList";
import { getT } from "@/lib/i18n/server";
import { formatUSD } from "@/lib/utils";
import { ShieldAlert } from "lucide-react";

export const metadata = { title: "Billetera — TRIADA" };

const STATUS_KEY: Record<string, string> = {
  pending: "st.pending",
  approved: "st.approved",
  paid: "st.paid",
  rejected: "st.rejected",
};

export default async function BilleteraPage() {
  const user = await requireUser();
  const t = getT();
  const [w, addresses, earnings] = await Promise.all([
    getWalletData(user.id),
    getWalletAddresses(user.id),
    getEarningsDetailed(user.id),
  ]);
  const paidEarnings = earnings.filter((e) => e.status === "validated" || e.status === "paid");

  return (
    <div className="space-y-8">
      <div>
        <p className="label">{t("nav.wallet")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("wal.title")}</h1>
        <p className="mt-1 text-xs text-muted">{t("wal.note")}</p>
      </div>

      <div className="grid grid-cols-1 gap-4 sm:grid-cols-3">
        <StatCard label={t("wal.balance")} value={formatUSD(w.balance)} />
        <StatCard label={t("wal.earned")} value={formatUSD(w.totalEarned)} />
        <StatCard label={t("wal.withdrawn")} value={formatUSD(w.totalWithdrawn)} />
      </div>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        {/* Retiro */}
        <Card>
          <h2 className="mb-3 text-lg font-bold text-cream">{t("wal.requestTitle")}</h2>
          {w.kycVerified ? (
            <WithdrawForm balance={w.balance} minWithdrawal={w.minWithdrawal} wallets={addresses} />
          ) : (
            <div className="flex items-start gap-3 rounded-md border border-gold/30 bg-gold/10 p-4">
              <ShieldAlert size={20} className="mt-0.5 shrink-0 text-gold" />
              <div>
                <p className="text-sm font-semibold text-gold">{t("wal.kycTitle")}</p>
                <p className="mt-1 text-xs text-muted">{t("wal.kycDesc")}</p>
              </div>
            </div>
          )}
        </Card>

        {/* Historial */}
        <Card>
          <h2 className="mb-3 text-lg font-bold text-cream">{t("wal.historyTitle")}</h2>
          {w.withdrawals.length === 0 ? (
            <p className="text-sm text-muted">{t("wal.noWithdrawals")}</p>
          ) : (
            <table className="w-full text-sm">
              <thead>
                <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                  <th className="pb-2 font-semibold">{t("wal.amount")}</th>
                  <th className="pb-2 font-semibold">{t("wal.statusCol")}</th>
                  <th className="pb-2 font-semibold">{t("wal.dateCol")}</th>
                </tr>
              </thead>
              <tbody>
                {w.withdrawals.map((wd, i) => (
                  <tr key={i} className="border-b border-border/50">
                    <td className="py-2.5 font-medium text-cream">{formatUSD(wd.amount)}</td>
                    <td className="py-2.5 text-muted">{t(STATUS_KEY[wd.status] ?? wd.status)}</td>
                    <td className="py-2.5 text-muted">
                      {new Date(wd.requested_at).toLocaleDateString("es")}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          )}
        </Card>
      </div>

      {/* Lista de carteras (direcciones USDT) */}
      <Card>
        <WalletAddresses addresses={addresses} />
      </Card>

      {/* Ganancias (pagos aprobados / validados con detalle) */}
      <Card>
        <div className="mb-3 flex items-center justify-between">
          <h2 className="text-lg font-bold text-cream">{t("wal.earnings")}</h2>
          <span className="text-sm text-muted">
            {t("common.total")}:{" "}
            <span className="text-gold">
              {formatUSD(paidEarnings.reduce((s, e) => s + e.amount, 0))}
            </span>
          </span>
        </div>
        <EarningsList
          earnings={paidEarnings}
          ownerName={user.full_name ?? "—"}
          ownerCode={user.referral_code}
        />
      </Card>
    </div>
  );
}
```

### 58. `app/(distribuidor)/herramientas/page.tsx`
**Herramientas.** Calculadoras con las constantes reales del plan.
```tsx
import { requireUser } from "@/lib/auth/session";
import { loadEngineConfig } from "@/lib/engine/config";
import { Calculators } from "@/components/dashboard/Calculators";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Herramientas — TRIADA" };

export default async function HerramientasPage() {
  await requireUser();
  const cfg = await loadEngineConfig();
  const t = getT();

  return (
    <div className="space-y-6">
      <div>
        <p className="label">{t("group.business")} · {t("nav.tools")}</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">{t("tool.title")}</h1>
        <p className="mt-1 text-sm text-muted">{t("tool.subtitle")}</p>
      </div>

      <Calculators
        cfg={{
          pointValue: cfg.POINT_VALUE,
          binaryPct: cfg.BINARY_PCT,
          rankPct: cfg.RANK_PCT,
          directL1Pct: cfg.DIRECT_L1_PCT,
          directL2Pct: cfg.DIRECT_L2_PCT,
        }}
      />
    </div>
  );
}
```

### 59. `app/(distribuidor)/academia/nova/page.tsx`
**Academia Nova** (requiere servicio ≥ 50 pts).
```tsx
import { requireService } from "@/lib/auth/guards";
import { AcademiaView } from "@/components/academia/AcademiaView";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Academia Nova — TRIADA" };

export default async function AcademiaNovaPage() {
  await requireService(50); // requiere Nova o superior
  const t = getT();

  return (
    <AcademiaView
      name="Nova"
      tagline={t("aca.novaTag")}
      modules={[
        { title: "Bienvenida y mentalidad", lessons: 4 },
        { title: "Introducción a la IA", lessons: 6 },
        { title: "Primeros pasos en trading", lessons: 5 },
        { title: "Finanzas digitales básicas", lessons: 5 },
        { title: "Tu marca personal", lessons: 4, locked: true },
        { title: "Construye tu red", lessons: 6, locked: true },
      ]}
    />
  );
}
```

### 60. `app/(distribuidor)/academia/quantum/page.tsx`
**Academia Quantum** (requiere servicio ≥ 100 pts).
```tsx
import { requireService } from "@/lib/auth/guards";
import { AcademiaView } from "@/components/academia/AcademiaView";
import { getT } from "@/lib/i18n/server";

export const metadata = { title: "Academia Quantum — TRIADA" };

export default async function AcademiaQuantumPage() {
  await requireService(100); // requiere Quantum o superior
  const t = getT();

  return (
    <AcademiaView
      name="Quantum"
      tagline={t("aca.quantumTag")}
      modules={[
        { title: "IA aplicada a negocios", lessons: 8 },
        { title: "Trading intermedio", lessons: 7 },
        { title: "Gestión de portafolio", lessons: 6 },
        { title: "Automatización con IA", lessons: 6 },
        { title: "Liderazgo y duplicación", lessons: 5, locked: true },
        { title: "Escala a rangos altos", lessons: 7, locked: true },
      ]}
    />
  );
}
```

### 61. `app/(distribuidor)/perfil/page.tsx`
**Mi Perfil.** Datos, foto y contraseña.
```tsx
import Link from "next/link";
import { requireUser } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import { getT } from "@/lib/i18n/server";
import { Card } from "@/components/ui/Card";
import { ProfileForm } from "@/components/profile/ProfileForm";
import { PasswordForm } from "@/components/profile/PasswordForm";
import { User, BadgeCheck, ShieldAlert } from "lucide-react";

export const metadata = { title: "Mi Perfil — TRIADA" };

type Tab = "personal" | "password" | "security";

export default async function PerfilPage({
  searchParams,
}: {
  searchParams: { tab?: string };
}) {
  const user = await requireUser();
  const t = getT();
  const tab: Tab =
    searchParams.tab === "password" ? "password" : searchParams.tab === "security" ? "security" : "personal";

  const supabase = createAdminClient();
  const [{ data: sponsor }, { data: mems }] = await Promise.all([
    user.sponsor_id
      ? supabase.from("users").select("referral_code").eq("id", user.sponsor_id).maybeSingle()
      : Promise.resolve({ data: null }),
    supabase.from("memberships").select("program_code, points").eq("user_id", user.id).eq("type", "service").eq("status", "active"),
  ]);
  const pkg = (mems ?? []).sort((a, b) => (b.points ?? 0) - (a.points ?? 0))[0]?.program_code ?? "—";

  const parts = (user.full_name ?? "").split(" ");
  const firstName = parts[0] ?? "";
  const lastName = parts.slice(1).join(" ");

  const tabs: { key: Tab; href: string; label: string }[] = [
    { key: "personal", href: "/perfil", label: t("profile.personal") },
    { key: "password", href: "/perfil?tab=password", label: t("profile.password") },
    { key: "security", href: "/perfil?tab=security", label: t("profile.security") },
  ];

  return (
    <div className="space-y-6">
      {/* Cabecera */}
      <Card>
        <div className="flex flex-wrap items-center gap-5">
          <span className="flex h-20 w-20 items-center justify-center overflow-hidden rounded-xl border-2 border-gold bg-navyDeep">
            {user.avatar_url ? (
              // eslint-disable-next-line @next/next/no-img-element
              <img src={user.avatar_url} alt="" className="h-full w-full object-cover" />
            ) : (
              <User size={36} className="text-muted" />
            )}
          </span>
          <div className="min-w-0">
            <p className="flex items-center gap-2 font-display text-xl font-extrabold text-cream">
              {user.full_name ?? "—"}
              {user.kyc_verified ? (
                <span className="inline-flex items-center gap-1 rounded-full bg-success/15 px-2 py-0.5 text-[10px] font-semibold text-success">
                  <BadgeCheck size={12} /> {t("profile.kycVerifiedBadge")}
                </span>
              ) : (
                <span className="inline-flex items-center gap-1 rounded-full bg-gold/15 px-2 py-0.5 text-[10px] font-semibold text-gold">
                  <ShieldAlert size={12} /> {t("profile.kycPendingBadge")}
                </span>
              )}
            </p>
            <p className="text-xs uppercase tracking-wide text-muted">{user.referral_code}</p>
            <div className="mt-2 flex flex-wrap gap-x-6 gap-y-1 text-xs text-muted">
              <span>{t("profile.email")}: <span className="text-cream">{user.email}</span></span>
              <span>{t("profile.sponsor")}: <span className="text-cream">{sponsor?.referral_code ?? "—"}</span></span>
              <span>{t("profile.package")}: <span className="text-cream">{pkg}</span></span>
            </div>
          </div>
        </div>
      </Card>

      {/* Pestañas */}
      <div className="flex flex-wrap gap-2 border-b border-border">
        {tabs.map((tb) => (
          <Link
            key={tb.key}
            href={tb.href}
            className={`-mb-px border-b-2 px-4 py-2 text-sm ${
              tab === tb.key ? "border-gold font-medium text-cream" : "border-transparent text-muted hover:text-cream"
            }`}
          >
            {tb.label}
          </Link>
        ))}
      </div>

      <Card>
        {tab === "personal" && (
          <ProfileForm
            values={{
              firstName,
              lastName,
              gender: user.gender,
              birthDate: user.birth_date,
              country: user.country,
              phone: user.phone,
              address: user.address,
              city: user.city,
              postalCode: user.postal_code,
              avatarUrl: user.avatar_url,
            }}
          />
        )}
        {tab === "password" && <PasswordForm />}
        {tab === "security" && (
          <div className="max-w-lg space-y-3">
            <h2 className="text-lg font-bold text-cream">{t("profile.kycTitle")}</h2>
            {user.kyc_verified ? (
              <p className="rounded-md border border-success/30 bg-success/10 px-4 py-3 text-sm text-success">
                {t("profile.kycVerified")}
              </p>
            ) : (
              <p className="rounded-md border border-gold/30 bg-gold/10 px-4 py-3 text-sm text-gold">
                {t("profile.kycPending")}
              </p>
            )}
          </div>
        )}
      </Card>
    </div>
  );
}
```


---

# PARTE 7 — COMPONENTES DEL DASHBOARD Y NEGOCIO
Piezas visuales y con interactividad de las pantallas anteriores.

### 62. `components/dashboard/ProfileHeader.tsx`
Cabecera del perfil con foto, ID, estado y llaves.
```tsx
"use client";

import { useEffect, useState } from "react";
import { User, BadgeCheck, Copy, Store, Calendar } from "lucide-react";
import { Flag } from "@/components/ui/Flag";

interface Props {
  name: string;
  code: string;
  country: string | null;
  registeredAt: string;
  statusActive: boolean; // tiene algún paquete activo
  binaryActive: boolean; // can_earn (participa del binario)
  packageName: string | null; // NOVA / QUANTUM…
  nextPurchaseDate: string | null;
  avatarUrl?: string | null;
}

export function ProfileHeader({
  name,
  code,
  country,
  registeredAt,
  statusActive,
  binaryActive,
  packageName,
  nextPurchaseDate,
  avatarUrl,
}: Props) {
  const [origin, setOrigin] = useState("");
  const [copied, setCopied] = useState<string | null>(null);
  useEffect(() => setOrigin(window.location.origin), []);

  function copy(kind: "ref" | "store") {
    const url =
      kind === "ref"
        ? `${origin}/registro?ref=${code}`
        : `${origin}/tienda?ref=${code}`;
    navigator.clipboard.writeText(url);
    setCopied(kind);
    setTimeout(() => setCopied(null), 1500);
  }

  const fmt = (d: string) => new Date(d).toLocaleDateString("es", { day: "2-digit", month: "long", year: "numeric" });

  return (
    <div className="rounded-card border border-border bg-card p-6 shadow-card">
      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        {/* Identidad */}
        <div className="flex gap-5">
          <div className="flex flex-col items-center">
            <span className="flex h-24 w-24 items-center justify-center overflow-hidden rounded-xl border-2 border-gold bg-navyDeep">
              {avatarUrl ? (
                // eslint-disable-next-line @next/next/no-img-element
                <img src={avatarUrl} alt="" className="h-full w-full object-cover" />
              ) : (
                <User size={40} className="text-muted" />
              )}
            </span>
            {packageName && (
              <span className="mt-2 rounded-full bg-gold/15 px-3 py-0.5 text-[11px] font-semibold text-gold">
                {packageName}
              </span>
            )}
          </div>

          <div className="min-w-0">
            <p className="flex items-center gap-1.5 font-display text-xl font-extrabold text-cream">
              {name}
              {statusActive && <BadgeCheck size={18} className="text-success" />}
            </p>
            <p className="text-xs uppercase tracking-wide text-muted">IBO id: {code}</p>

            <div className="mt-3 flex flex-wrap gap-2">
              <span className="inline-flex items-center gap-1.5 rounded-md border border-border bg-navyDeep px-3 py-1.5 text-xs text-cream">
                <Flag country={country} />
                {country ?? "—"}
              </span>
              <span className="rounded-md border border-border bg-navyDeep px-3 py-1.5 text-xs text-cream">
                Registro: <span className="font-semibold">{new Date(registeredAt).toLocaleDateString("es")}</span>
              </span>
            </div>

            <div className="mt-2 inline-flex flex-wrap items-center gap-2 rounded-md border border-border bg-navyDeep px-3 py-1.5 text-xs">
              <span className={statusActive ? "text-success" : "text-muted"}>
                {statusActive ? "Activo" : "Inactivo"}
              </span>
              <span className="text-border">·</span>
              <span className={binaryActive ? "text-success" : "text-muted"}>
                Binario {binaryActive ? "Activo" : "Inactivo"}
              </span>
            </div>
          </div>
        </div>

        {/* Próxima compra + ligas */}
        <div className="flex flex-col justify-between gap-4">
          <div className="rounded-card border border-border p-4">
            <p className="flex items-center gap-2 text-sm font-semibold text-cream">
              <Calendar size={16} className="text-gold" />
              {nextPurchaseDate ? fmt(nextPurchaseDate) : "Sin renovación programada"}
            </p>
            <p className="text-xs text-muted">Fecha de próxima compra (renovación)</p>
          </div>

          <div className="flex flex-wrap gap-3">
            <button
              onClick={() => copy("ref")}
              className="inline-flex items-center gap-2 rounded-md border border-border px-4 py-2 text-xs font-semibold text-cream hover:border-gold"
            >
              <Copy size={14} /> {copied === "ref" ? "¡Copiado!" : "Liga de referido"}
            </button>
            <button
              onClick={() => copy("store")}
              className="inline-flex items-center gap-2 rounded-md border border-border px-4 py-2 text-xs font-semibold text-cream hover:border-gold"
            >
              <Store size={14} /> {copied === "store" ? "¡Copiado!" : "Liga de tienda en línea"}
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

### 63. `components/dashboard/IncomeChart.tsx`
Gráfico de área SVG con selector Año/Mes/Día y curva suavizada.
```tsx
"use client";

import { useMemo, useState } from "react";
import { useT } from "@/components/i18n/LanguageProvider";

/**
 * Gráfica de ingresos del dashboard: área suave con selector Año / Mes / Día.
 *
 * Recibe una serie DIARIA real (YYYY-MM-DD → monto) y la reagrupa en el cliente
 * según la vista elegida. El eje vertical es el volumen de dinero (auto-escala a
 * un máximo "redondo"); el eje horizontal son años, meses o días.
 */
type View = "year" | "month" | "day";

interface Props {
  daily: { date: string; amount: number }[];
  emptyLabel: string;
}

// --- Geometría del lienzo (coordenadas internas; el SVG escala al ancho) ---
const W = 520;
const H = 240;
const PAD_L = 48;
const PAD_R = 14;
const PAD_T = 16;
const PAD_B = 28;
const PLOT_W = W - PAD_L - PAD_R;
const PLOT_H = H - PAD_T - PAD_B;
const BASELINE = H - PAD_B;

function monthShort(d: Date): string {
  return d.toLocaleDateString(undefined, { month: "short" }).replace(".", "");
}

function fmtMoney(n: number): string {
  if (n >= 1000) {
    const k = n / 1000;
    return `$${k.toLocaleString(undefined, { maximumFractionDigits: k < 10 ? 1 : 0 })}k`;
  }
  return `$${Math.round(n).toLocaleString()}`;
}

/** Redondea el máximo a un valor "bonito" para el eje (1/2/2.5/5/10 × 10^n). */
function niceMax(v: number): number {
  if (v <= 0) return 100;
  const pow = Math.pow(10, Math.floor(Math.log10(v)));
  const f = v / pow;
  const nice = f <= 1 ? 1 : f <= 2 ? 2 : f <= 2.5 ? 2.5 : f <= 5 ? 5 : 10;
  return nice * pow;
}

/** Catmull-Rom → curva de Bézier suave que pasa por todos los puntos. */
function smoothPath(pts: { x: number; y: number }[]): string {
  if (pts.length === 0) return "";
  if (pts.length === 1) return `M ${pts[0].x} ${pts[0].y}`;
  const d = [`M ${pts[0].x} ${pts[0].y}`];
  for (let i = 0; i < pts.length - 1; i++) {
    const p0 = pts[i - 1] ?? pts[i];
    const p1 = pts[i];
    const p2 = pts[i + 1];
    const p3 = pts[i + 2] ?? p2;
    const cp1x = p1.x + (p2.x - p0.x) / 6;
    const cp1y = p1.y + (p2.y - p0.y) / 6;
    const cp2x = p2.x - (p3.x - p1.x) / 6;
    const cp2y = p2.y - (p3.y - p1.y) / 6;
    d.push(`C ${cp1x} ${cp1y} ${cp2x} ${cp2y} ${p2.x} ${p2.y}`);
  }
  return d.join(" ");
}

export function IncomeChart({ daily, emptyLabel }: Props) {
  const { t } = useT();
  const [view, setView] = useState<View>("month");

  // Mapas de agregación por día / mes / año (a partir de la serie diaria).
  const { byDay, byMonth, byYear, minYear } = useMemo(() => {
    const byDay = new Map<string, number>();
    const byMonth = new Map<string, number>();
    const byYear = new Map<string, number>();
    let minYear = new Date().getFullYear();
    for (const { date, amount } of daily) {
      byDay.set(date, (byDay.get(date) ?? 0) + amount);
      const ym = date.slice(0, 7);
      byMonth.set(ym, (byMonth.get(ym) ?? 0) + amount);
      const y = date.slice(0, 4);
      byYear.set(y, (byYear.get(y) ?? 0) + amount);
      const yn = Number(y);
      if (yn && yn < minYear) minYear = yn;
    }
    return { byDay, byMonth, byYear, minYear };
  }, [daily]);

  // Serie de la vista actual: [{ label, value }].
  const series = useMemo(() => {
    const out: { label: string; value: number }[] = [];
    const now = new Date();
    if (view === "day") {
      for (let i = 13; i >= 0; i--) {
        const d = new Date();
        d.setDate(now.getDate() - i);
        const key = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
        out.push({ label: `${d.getDate()} ${monthShort(d)}`, value: byDay.get(key) ?? 0 });
      }
    } else if (view === "month") {
      for (let i = 11; i >= 0; i--) {
        const d = new Date(now.getFullYear(), now.getMonth() - i, 1);
        const key = `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}`;
        out.push({ label: `${monthShort(d)} ${String(d.getFullYear()).slice(2)}`, value: byMonth.get(key) ?? 0 });
      }
    } else {
      const start = Math.max(minYear, now.getFullYear() - 5);
      for (let y = start; y <= now.getFullYear(); y++) {
        out.push({ label: String(y), value: byYear.get(String(y)) ?? 0 });
      }
    }
    return out;
  }, [view, byDay, byMonth, byYear, minYear]);

  const max = niceMax(Math.max(...series.map((s) => s.value), 0));
  const ticks = [0, 0.25, 0.5, 0.75, 1].map((f) => f * max);

  const pts = series.map((s, i) => ({
    x: series.length === 1 ? PAD_L + PLOT_W / 2 : PAD_L + (i / (series.length - 1)) * PLOT_W,
    y: BASELINE - (s.value / max) * PLOT_H,
  }));
  const line = smoothPath(pts);
  const area = pts.length
    ? `${line} L ${pts[pts.length - 1].x} ${BASELINE} L ${pts[0].x} ${BASELINE} Z`
    : "";

  // Mostrar solo algunas etiquetas del eje X si hay muchas.
  const labelStep = Math.max(1, Math.ceil(series.length / 8));

  const tabs: { key: View; label: string }[] = [
    { key: "year", label: t("chart.year") },
    { key: "month", label: t("chart.month") },
    { key: "day", label: t("chart.day") },
  ];

  return (
    <div>
      {/* Encabezado: título + selector Año/Mes/Día */}
      <div className="mb-3 flex items-center justify-between">
        <h2 className="text-lg font-bold text-cream">{t("dash.income")}</h2>
        <div className="inline-flex items-center gap-1 text-xs">
          {tabs.map((tab) => (
            <button
              key={tab.key}
              onClick={() => setView(tab.key)}
              className={`rounded-md px-2.5 py-1 font-medium transition-colors ${
                view === tab.key ? "bg-cobalt/15 text-cobalt" : "text-muted hover:text-cream"
              }`}
            >
              {tab.label}
            </button>
          ))}
        </div>
      </div>

      {daily.length === 0 ? (
        <p className="text-sm text-muted">{emptyLabel}</p>
      ) : (
        <svg viewBox={`0 0 ${W} ${H}`} width="100%" className="block" role="img" aria-label={t("dash.income")}>
          <defs>
            <linearGradient id="incomeFill" x1="0" y1="0" x2="0" y2="1">
              <stop offset="0%" stopColor="rgb(3 83 164)" stopOpacity="0.45" />
              <stop offset="100%" stopColor="rgb(3 83 164)" stopOpacity="0" />
            </linearGradient>
          </defs>

          {/* Rejilla + etiquetas del eje Y (dinero) */}
          {ticks.map((tk, i) => {
            const y = BASELINE - (tk / max) * PLOT_H;
            return (
              <g key={i}>
                <line x1={PAD_L} y1={y} x2={W - PAD_R} y2={y} className="stroke-border" strokeWidth={0.5} opacity={0.5} />
                <text x={PAD_L - 8} y={y + 3} textAnchor="end" className="fill-muted" style={{ fontSize: 10 }}>
                  {fmtMoney(tk)}
                </text>
              </g>
            );
          })}

          {/* Área + línea */}
          <path d={area} fill="url(#incomeFill)" />
          <path d={line} fill="none" className="stroke-cobalt" strokeWidth={2} strokeLinejoin="round" strokeLinecap="round" />

          {/* Puntos */}
          {pts.map((p, i) => (
            <circle key={i} cx={p.x} cy={p.y} r={3.5} className="fill-cobalt stroke-card" strokeWidth={2} />
          ))}

          {/* Etiquetas del eje X (años / meses / días) */}
          {series.map((s, i) =>
            i % labelStep === 0 ? (
              <text key={i} x={pts[i].x} y={H - 8} textAnchor="middle" className="fill-muted" style={{ fontSize: 10 }}>
                {s.label}
              </text>
            ) : null
          )}
        </svg>
      )}
    </div>
  );
}
```

### 64. `components/dashboard/CountdownTimer.tsx`
Cuenta regresiva a la próxima renovación.
```tsx
"use client";

import { useEffect, useState } from "react";
import { Clock } from "lucide-react";

/** Cuenta regresiva hasta la fecha de renovación (próxima compra). */
export function CountdownTimer({ targetIso }: { targetIso: string | null }) {
  // `now` arranca en null para que el servidor y la primera renderización del
  // cliente coincidan (evita el mismatch de hidratación por los segundos).
  const [now, setNow] = useState<number | null>(null);

  useEffect(() => {
    setNow(Date.now());
    const id = setInterval(() => setNow(Date.now()), 1000);
    return () => clearInterval(id);
  }, []);

  if (!targetIso) {
    return (
      <div className="rounded-card border border-border bg-navyDeep/60 px-5 py-4 text-sm text-muted">
        Sin renovación programada. Activa un programa en la Tienda.
      </div>
    );
  }

  const diff = now === null ? null : Math.max(0, new Date(targetIso).getTime() - now);
  const days = diff === null ? null : Math.floor(diff / 86400000);
  const hrs = diff === null ? null : Math.floor((diff % 86400000) / 3600000);
  const mins = diff === null ? null : Math.floor((diff % 3600000) / 60000);
  const secs = diff === null ? null : Math.floor((diff % 60000) / 1000);
  const expired = diff === 0;

  return (
    <div className="rounded-card border border-gold/30 bg-navyDeep/60 px-5 py-4">
      <div className="flex items-center justify-between">
        <p className="flex items-center gap-2 text-xs uppercase tracking-label text-muted">
          <Clock size={14} className="text-gold" />
          {expired ? "Renovación vencida" : "Próxima renovación en"}
        </p>
        <p className="text-[11px] text-muted">
          {new Date(targetIso).toLocaleDateString("es", { day: "2-digit", month: "short", year: "numeric" })}
        </p>
      </div>
      <div className="mt-3 grid grid-cols-4 gap-2 text-center">
        {[
          { v: days, l: "Días" },
          { v: hrs, l: "Hrs" },
          { v: mins, l: "Min" },
          { v: secs, l: "Seg" },
        ].map((b) => (
          <div key={b.l} className="rounded-lg bg-card py-2">
            <p className="font-mono text-2xl font-extrabold text-gold">
              {b.v === null ? "--" : String(b.v).padStart(2, "0")}
            </p>
            <p className="text-[10px] uppercase tracking-wide text-muted">{b.l}</p>
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 65. `components/dashboard/ThreeFreeWidget.tsx`
Widget del programa de estudiante.
```tsx
"use client";

import { useEffect, useState } from "react";
import { Gift, Share2 } from "lucide-react";

/**
 * Widget "3 & Free": refiere 3 clientes activos y tu próxima suscripción es gratis.
 * Medidor circular de progreso (estilo TRIADA).
 */
export function ThreeFreeWidget({
  qualified,
  referralCode,
  nextRenewal,
}: {
  qualified: number;
  referralCode: string;
  nextRenewal: string | null;
}) {
  const goal = 3;
  const value = Math.min(qualified, goal);
  const remaining = Math.max(0, goal - value);
  const [origin, setOrigin] = useState("");
  const [copied, setCopied] = useState(false);
  useEffect(() => setOrigin(window.location.origin), []);

  // Semicírculo: 180° de arco. radio 80, circunferencia media = π*r.
  const r = 80;
  const semi = Math.PI * r; // longitud del semicírculo
  const pct = value / goal;
  const dash = semi * pct;

  function share() {
    navigator.clipboard.writeText(`${origin}/registro?ref=${referralCode}`);
    setCopied(true);
    setTimeout(() => setCopied(false), 1500);
  }

  return (
    <div className="rounded-card border border-border bg-card p-6 shadow-card">
      <div className="flex items-center gap-2 border-b border-border pb-3">
        <Gift size={18} className="text-gold" />
        <h2 className="font-display text-lg font-bold text-cream">3 &amp; Gratis</h2>
      </div>

      <p className="mt-4 text-center text-sm text-muted">
        {remaining > 0 ? (
          <>
            Estás a <span className="font-bold text-gold">{remaining}</span> referido(s) de tu
            próxima suscripción <span className="font-bold text-gold">gratis</span>. Refiere 3
            clientes activos.
          </>
        ) : (
          <span className="font-semibold text-success">
            ¡Lo lograste! Tu próxima suscripción es gratis. 🎉
          </span>
        )}
      </p>

      {/* Medidor semicircular */}
      <div className="mt-4 flex justify-center">
        <svg viewBox="0 0 200 110" className="w-56">
          <path d="M20 100 A80 80 0 0 1 180 100" fill="none" stroke="#1A3570" strokeWidth="14" strokeLinecap="round" />
          <path
            d="M20 100 A80 80 0 0 1 180 100"
            fill="none"
            stroke="#C2B280"
            strokeWidth="14"
            strokeLinecap="round"
            strokeDasharray={`${dash} ${semi}`}
          />
          <text x="100" y="86" textAnchor="middle" className="fill-cream" style={{ fontSize: 28, fontWeight: 800 }}>
            {value} de {goal}
          </text>
          <text x="100" y="104" textAnchor="middle" className="fill-current text-muted" style={{ fontSize: 10, letterSpacing: 1 }}>
            CALIFICADOS
          </text>
        </svg>
      </div>

      {/* Tarjetas */}
      <div className="mt-4 grid grid-cols-2 gap-3">
        <div className="rounded-lg bg-navyDeep p-3 text-center">
          <p className="font-display text-2xl font-extrabold text-cream">{qualified}</p>
          <p className="text-[10px] uppercase tracking-wide text-muted">Referidos calificados</p>
        </div>
        <div className="rounded-lg bg-navyDeep p-3 text-center">
          <p className="font-display text-lg font-extrabold text-gold">
            {nextRenewal ? new Date(nextRenewal).toLocaleDateString("es", { day: "2-digit", month: "short" }) : "—"}
          </p>
          <p className="text-[10px] uppercase tracking-wide text-muted">Próxima renovación</p>
        </div>
      </div>

      <button
        onClick={share}
        className="mt-4 flex w-full items-center justify-center gap-2 rounded-[4px] bg-gold px-4 py-2.5 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight"
      >
        <Share2 size={14} /> {copied ? "¡Enlace copiado!" : "Compartir tu enlace de referido"}
      </button>
    </div>
  );
}
```

### 66. `components/dashboard/ReferralLinks.tsx`
Link de invitación con copiar.
```tsx
"use client";

import { useEffect, useState } from "react";

/**
 * Link de referido ÚNICO. Los nuevos miembros entran como pendientes de
 * colocación; el distribuidor los ubica luego en su árbol.
 * Solo se muestra si el distribuidor tiene can_distribute activo.
 */
export function ReferralLinks({ referralCode }: { referralCode: string }) {
  const [origin, setOrigin] = useState("");
  const [copied, setCopied] = useState(false);

  useEffect(() => {
    setOrigin(window.location.origin);
  }, []);

  const url = `${origin}/registro?ref=${referralCode}`;

  async function copy() {
    await navigator.clipboard.writeText(url);
    setCopied(true);
    setTimeout(() => setCopied(false), 1500);
  }

  return (
    <div>
      <p className="mb-1 text-xs font-medium uppercase tracking-[0.08em] text-muted">
        Tu link de invitación
      </p>
      <div className="flex items-center gap-2">
        <input
          readOnly
          value={url}
          className="w-full truncate rounded-md border border-border bg-navyDeep px-3 py-2 text-xs text-cream"
        />
        <button
          onClick={copy}
          className="shrink-0 rounded-md border border-gold px-3 py-2 text-xs font-semibold text-gold transition-colors hover:bg-gold/10"
        >
          {copied ? "¡Copiado!" : "Copiar"}
        </button>
      </div>
      <p className="mt-2 text-xs text-muted">
        Quien se registre con tu link quedará pendiente de colocación en tu red.
      </p>
    </div>
  );
}
```

### 67. `components/dashboard/EarningsList.tsx`
Lista de ganancias por vía.
```tsx
"use client";

import { Fragment, useState } from "react";
import { Search } from "lucide-react";
import { formatUSD } from "@/lib/utils";
import { useT } from "@/components/i18n/LanguageProvider";
import type { EarningRow } from "@/lib/data/distributor";

const TYPE_KEY: Record<string, string> = {
  direct_l1: "com.directL1",
  direct_l2: "com.directL2",
  binary: "com.binary",
  rank: "com.rank",
  matching: "com.matching",
  lifestyle: "com.lifestyle",
};
const STATUS_KEY: Record<string, string> = {
  pending: "st.pending",
  validated: "st.validated",
  paid: "st.paid",
  rejected: "st.rejected",
};

interface Group {
  period: string;
  date: string;
  total: number;
  rows: EarningRow[];
}

export function EarningsList({
  earnings,
  ownerName,
  ownerCode,
}: {
  earnings: EarningRow[];
  ownerName: string;
  ownerCode: string;
}) {
  const { t } = useT();
  const [open, setOpen] = useState<string | null>(null);

  // Agrupar por período.
  const groups = new Map<string, Group>();
  for (const e of earnings) {
    const g = groups.get(e.period) ?? { period: e.period, date: e.created_at, total: 0, rows: [] };
    g.total += e.amount;
    g.rows.push(e);
    if (e.created_at > g.date) g.date = e.created_at;
    groups.set(e.period, g);
  }
  const list = [...groups.values()].sort((a, b) => b.period.localeCompare(a.period));

  if (list.length === 0) {
    return <p className="text-sm text-muted">{t("wal.noEarnings")}</p>;
  }

  return (
    <div className="space-y-3">
      <table className="w-full text-sm">
        <thead>
          <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
            <th className="pb-2 font-semibold">{t("common.name")}</th>
            <th className="pb-2 font-semibold">{t("common.period")}</th>
            <th className="pb-2 font-semibold">{t("wal.dateCol")}</th>
            <th className="pb-2 font-semibold text-right">{t("wal.amount")}</th>
            <th className="pb-2 font-semibold text-right">{t("common.detail")}</th>
          </tr>
        </thead>
        <tbody>
          {list.map((g) => (
            <Fragment key={g.period}>
              <tr className="border-b border-border/50">
                <td className="py-3 text-success">
                  {ownerName} <span className="text-muted">({ownerCode})</span>
                </td>
                <td className="py-3 text-muted">{g.period}</td>
                <td className="py-3 text-muted">{new Date(g.date).toLocaleDateString("es")}</td>
                <td className="py-3 text-right font-medium text-cream">{formatUSD(g.total)}</td>
                <td className="py-3 text-right">
                  <button
                    onClick={() => setOpen(open === g.period ? null : g.period)}
                    className="inline-flex h-8 w-8 items-center justify-center rounded-md bg-gold text-goldInk hover:bg-goldLight"
                    title="Ver detalle"
                  >
                    <Search size={14} />
                  </button>
                </td>
              </tr>
              {open === g.period && (
                <tr key={g.period + "-d"}>
                  <td colSpan={5} className="bg-navyDeep/40 px-4 py-3">
                    <p className="mb-2 text-[11px] uppercase tracking-wide text-muted">
                      {t("com.title")} · {g.period}
                    </p>
                    <table className="w-full text-xs">
                      <thead>
                        <tr className="text-left text-[10px] uppercase tracking-wide text-muted">
                          <th className="pb-1 font-semibold">{t("com.colVia")}</th>
                          <th className="pb-1 font-semibold">{t("com.colOrigin")}</th>
                          <th className="pb-1 font-semibold">{t("com.colPackage")}</th>
                          <th className="pb-1 font-semibold">{t("com.colStatus")}</th>
                          <th className="pb-1 font-semibold text-right">{t("wal.amount")}</th>
                        </tr>
                      </thead>
                      <tbody>
                        {g.rows.map((r, i) => (
                          <tr key={i} className="border-t border-border/40">
                            <td className="py-1.5 text-cream">{t(TYPE_KEY[r.type] ?? r.type)}</td>
                            <td className="py-1.5 text-muted">
                              {r.sourceName ? (
                                <>
                                  <span className="text-cream">{r.sourceCode}</span> · {r.sourceName}
                                </>
                              ) : (
                                t("com.networkVolume")
                              )}
                            </td>
                            <td className="py-1.5 text-muted">{r.sourcePackage ?? "—"}</td>
                            <td className="py-1.5 text-muted">{t(STATUS_KEY[r.status] ?? r.status)}</td>
                            <td className="py-1.5 text-right text-cream">{formatUSD(r.amount)}</td>
                          </tr>
                        ))}
                      </tbody>
                    </table>
                  </td>
                </tr>
              )}
            </Fragment>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

### 68. `components/dashboard/CoinBadge.tsx`
Insignia de monto.
```tsx
/** Insignia visual de la moneda (USDT con su red). */
export function CoinBadge({ currency }: { currency: string }) {
  // currency: "USDT.TRC-20" | "USDT.BEP-20"
  const network = currency.includes(".") ? currency.split(".")[1] : currency;
  return (
    <span className="inline-flex items-center gap-2">
      <span className="flex h-6 w-6 items-center justify-center rounded-full bg-[#26A17B] text-sm font-bold text-white">
        ₮
      </span>
      <span className="text-sm text-cream">
        USDT <span className="text-muted">{network}</span>
      </span>
    </span>
  );
}
```

### 69. `components/dashboard/RankExplorer.tsx`
Explorador de los 13 rangos y requisitos.
```tsx
"use client";

import { useState } from "react";
import { Check, X } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import { RankBadge } from "@/components/ui/RankBadge";
import type { RankRule } from "@/lib/engine/config";

interface Props {
  ranks: RankRule[];
  currentRank: number;
  leftTotal: number;
  rightTotal: number;
  directsLeft: number;
  directsRight: number;
  canEarn: boolean;
}

export function RankExplorer({
  ranks,
  currentRank,
  leftTotal,
  rightTotal,
  directsLeft,
  directsRight,
  canEarn,
}: Props) {
  const { t } = useT();
  // Por defecto, el siguiente rango (o el actual si es el máximo).
  const initial = Math.min(currentRank + 1, 13) || 1;
  const [selected, setSelected] = useState(initial);

  const rank = ranks.find((r) => r.level === selected);

  return (
    <div className="space-y-6">
      {/* Detalle del rango seleccionado */}
      {rank && (
        <div className="rounded-card border border-gold/40 bg-gradient-to-br from-card to-navy p-5">
          <div className="flex items-center justify-between">
            <div className="flex items-center gap-3">
              <RankBadge level={rank.level} size="md" />
              <div>
                <p className="label">{t("rank.reqFor")}</p>
                <p className="font-display text-2xl font-extrabold text-gold">{rank.name}</p>
              </div>
            </div>
            <span className="rounded-full bg-navyDeep px-3 py-1 text-xs text-muted">
              #{rank.level} · {rank.points_per_leg.toLocaleString()} {t("rank.ptsPerLeg")}
            </span>
          </div>

          <ul className="mt-4 space-y-2.5">
            <ConditionRow
              label={t("rank.ptsLeft")}
              current={leftTotal}
              required={rank.points_per_leg}
              missingLabel={t("rank.missing")}
              completedLabel={t("rank.completed")}
              unit="pts"
            />
            <ConditionRow
              label={t("rank.ptsRight")}
              current={rightTotal}
              required={rank.points_per_leg}
              missingLabel={t("rank.missing")}
              completedLabel={t("rank.completed")}
              unit="pts"
            />
            <ConditionRow
              label={`${t("rank.directs")}`}
              current={directsLeft}
              required={rank.min_directs_left}
              extraCurrent={directsRight}
              extraRequired={rank.min_directs_right}
              missingLabel={t("rank.missing")}
              completedLabel={t("rank.completed")}
              unit=""
            />
            <li className="flex items-center justify-between rounded-lg bg-navyDeep px-3 py-2 text-sm">
              <span className="flex items-center gap-2">
                {canEarn ? <Check size={14} className="text-success" /> : <X size={14} className="text-danger" />}
                <span className="text-cream">{t("rank.serviceActive")}</span>
              </span>
              <span className={canEarn ? "text-success" : "text-danger"}>
                {canEarn ? t("rank.yes") : t("rank.no")}
              </span>
            </li>
          </ul>
        </div>
      )}

      {/* Escalera de rangos (clic para explorar) */}
      <div>
        <p className="mb-2 text-xs text-muted">{t("rank.tapHint")}</p>
        <div className="grid grid-cols-1 gap-2 sm:grid-cols-2 lg:grid-cols-3">
          {ranks.map((r) => {
            const achieved = r.level <= currentRank;
            const isSel = r.level === selected;
            return (
              <button
                key={r.level}
                onClick={() => setSelected(r.level)}
                className={`flex items-center gap-3 rounded-card border p-3 text-left transition-colors ${
                  isSel ? "border-gold bg-gold/10" : "border-border bg-card hover:border-gold/50"
                }`}
              >
                <RankBadge
                  level={r.level}
                  size="sm"
                  className={achieved ? "" : "opacity-40 grayscale"}
                />
                <div className="min-w-0">
                  <p className="truncate text-sm font-semibold text-cream">{r.name}</p>
                  <p className="text-[11px] text-muted">
                    {achieved ? t("rank.achieved") : `${r.points_per_leg.toLocaleString()} ${t("rank.ptsPerLeg")}`}
                  </p>
                </div>
              </button>
            );
          })}
        </div>
      </div>
    </div>
  );
}

function ConditionRow({
  label,
  current,
  required,
  extraCurrent,
  extraRequired,
  missingLabel,
  completedLabel,
  unit,
}: {
  label: string;
  current: number;
  required: number;
  extraCurrent?: number;
  extraRequired?: number;
  missingLabel: string;
  completedLabel: string;
  unit: string;
}) {
  const okMain = current >= required;
  const okExtra = extraRequired === undefined || (extraCurrent ?? 0) >= extraRequired;
  const met = okMain && okExtra;
  const missMain = Math.max(0, required - current);
  const missExtra = extraRequired === undefined ? 0 : Math.max(0, extraRequired - (extraCurrent ?? 0));

  return (
    <li className="flex items-center justify-between rounded-lg bg-navyDeep px-3 py-2 text-sm">
      <span className="flex items-center gap-2">
        {met ? <Check size={14} className="text-success" /> : <X size={14} className="text-danger" />}
        <span className="text-cream">{label}</span>
      </span>
      <span className="text-right">
        <span className={met ? "text-success" : "text-cream"}>
          {extraRequired === undefined
            ? `${current} / ${required} ${unit}`
            : `${current}/${extraCurrent} · req ${required}/${extraRequired}`}
        </span>
        {!met && (
          <span className="ml-2 text-[11px] text-danger">
            {missingLabel} {extraRequired === undefined ? `${missMain} ${unit}` : `${missMain}/${missExtra}`}
          </span>
        )}
      </span>
    </li>
  );
}
```

### 70. `components/dashboard/Calculators.tsx`
Calculadoras del plan de compensación.
```tsx
"use client";

import { useState } from "react";
import { Card } from "@/components/ui/Card";
import { formatUSD } from "@/lib/utils";
import { useT } from "@/components/i18n/LanguageProvider";

export interface CalcConfig {
  pointValue: number;
  binaryPct: number;
  rankPct: number;
  directL1Pct: number;
  directL2Pct: number;
}

// Programas y precios de inscripción (igual que la calculadora de Excel).
const DIRECT_PROGRAMS = [
  { code: "NOVA", name: "Nova", price: 125 },
  { code: "QUANTUM", name: "Quantum", price: 200 },
  { code: "NEXUS", name: "Nexus / Q.Trim", price: 450 },
  { code: "QSEM", name: "Q. Semestral", price: 820 },
  { code: "QANU", name: "Q. Anual", price: 1397 },
];
const NET_PROGRAMS = [
  { code: "NOVA", name: "Nova", points: 50 },
  { code: "QUANTUM", name: "Quantum", points: 100 },
  { code: "NEXUS", name: "Nexus", points: 400 },
  { code: "QSEM", name: "Q. Semestral", points: 700 },
  { code: "QANU", name: "Q. Anual", points: 1200 },
];
const RANKS = [
  { pts: 200, name: "PIONERO" },
  { pts: 400, name: "IMPULSOR" },
  { pts: 1000, name: "ÉLITE" },
  { pts: 2000, name: "EXECUTIVE" },
  { pts: 4000, name: "VISIONARIO" },
  { pts: 7000, name: "CONQUISTADOR" },
  { pts: 10000, name: "SOBERANO" },
  { pts: 20000, name: "DIAMANTE" },
  { pts: 40000, name: "DIAMANTE IMPERIAL" },
  { pts: 70000, name: "DIAMANTE MONARCA" },
  { pts: 120000, name: "LEYENDA" },
  { pts: 240000, name: "LEYENDA TRIADA" },
  { pts: 500000, name: "LEYENDA CORONA" },
];

const inputCls =
  "w-20 rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold";

export function Calculators({ cfg }: { cfg: CalcConfig }) {
  const [n1, setN1] = useState<number[]>(Array(5).fill(0));
  const [n2, setN2] = useState<number[]>(Array(5).fill(0));
  const [left, setLeft] = useState<number[]>(Array(5).fill(0));
  const [right, setRight] = useState<number[]>(Array(5).fill(0));
  const { t } = useT();

  const set = (arr: number[], setter: (a: number[]) => void, i: number, v: number) => {
    const copy = [...arr];
    copy[i] = v;
    setter(copy);
  };

  // Distribución directa.
  const directTotal = DIRECT_PROGRAMS.reduce(
    (s, p, i) => s + n1[i] * p.price * cfg.directL1Pct + n2[i] * p.price * cfg.directL2Pct,
    0
  );

  // Tu red.
  const leftPoints = NET_PROGRAMS.reduce((s, p, i) => s + left[i] * p.points, 0);
  const rightPoints = NET_PROGRAMS.reduce((s, p, i) => s + right[i] * p.points, 0);
  const lesser = Math.min(leftPoints, rightPoints);

  const matched = [...RANKS].reverse().find((r) => lesser >= r.pts) ?? null;
  const binary = lesser * cfg.pointValue * cfg.binaryPct;
  const bonoRango = matched ? matched.pts * cfg.pointValue * cfg.rankPct : 0;
  const chequeTotal = directTotal + binary + bonoRango;

  return (
    <div className="space-y-6">
      {/* DISTRIBUCIÓN DIRECTA */}
      <Card>
        <h2 className="text-lg font-bold text-cream">{t("tool.directTitle")}</h2>
        <p className="mb-4 text-xs text-muted">{t("tool.directDesc")}</p>
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">{t("tool.colProgram")}</th>
                <th className="pb-2 font-semibold">{t("tool.colEnroll")}</th>
                <th className="pb-2 font-semibold">{t("tool.colN1")}</th>
                <th className="pb-2 font-semibold">{t("tool.colN2")}</th>
                <th className="pb-2 font-semibold text-right">{t("tool.colBonus")}</th>
              </tr>
            </thead>
            <tbody>
              {DIRECT_PROGRAMS.map((p, i) => {
                const bono = n1[i] * p.price * cfg.directL1Pct + n2[i] * p.price * cfg.directL2Pct;
                return (
                  <tr key={p.code} className="border-b border-border/50">
                    <td className="py-2 text-cream">{p.name}</td>
                    <td className="py-2 text-muted">{formatUSD(p.price)}</td>
                    <td className="py-2">
                      <input type="number" min={0} value={n1[i] || ""} onChange={(e) => set(n1, setN1, i, Number(e.target.value))} className={inputCls} />
                    </td>
                    <td className="py-2">
                      <input type="number" min={0} value={n2[i] || ""} onChange={(e) => set(n2, setN2, i, Number(e.target.value))} className={inputCls} />
                    </td>
                    <td className="py-2 text-right font-medium text-cream">{formatUSD(bono)}</td>
                  </tr>
                );
              })}
            </tbody>
          </table>
        </div>
        <div className="mt-3 flex justify-end gap-2 text-sm">
          <span className="text-muted">{t("tool.directTotal")}</span>
          <span className="font-semibold text-gold">{formatUSD(directTotal)}</span>
        </div>
      </Card>

      {/* TU RED */}
      <Card>
        <h2 className="text-lg font-bold text-cream">{t("tool.netTitle")}</h2>
        <p className="mb-4 text-xs text-muted">{t("tool.netDesc")}</p>
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">{t("tool.colProgram")}</th>
                <th className="pb-2 font-semibold">{t("tool.colPts")}</th>
                <th className="pb-2 font-semibold">{t("tool.colLeft")}</th>
                <th className="pb-2 font-semibold">{t("tool.colRight")}</th>
              </tr>
            </thead>
            <tbody>
              {NET_PROGRAMS.map((p, i) => (
                <tr key={p.code} className="border-b border-border/50">
                  <td className="py-2 text-cream">{p.name}</td>
                  <td className="py-2 text-muted">{p.points}</td>
                  <td className="py-2">
                    <input type="number" min={0} value={left[i] || ""} onChange={(e) => set(left, setLeft, i, Number(e.target.value))} className={inputCls} />
                  </td>
                  <td className="py-2">
                    <input type="number" min={0} value={right[i] || ""} onChange={(e) => set(right, setRight, i, Number(e.target.value))} className={inputCls} />
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
        <div className="mt-4 grid grid-cols-2 gap-3 sm:grid-cols-4">
          <Result label={t("tool.ptsLeft")} value={`${leftPoints}`} />
          <Result label={t("tool.ptsRight")} value={`${rightPoints}`} />
          <Result label={t("tool.lesser")} value={`${lesser} pts`} />
          <Result label={t("tool.yourRank")} value={matched?.name ?? t("tool.noRank")} />
        </div>
      </Card>

      {/* RESULTADO */}
      <Card premium>
        <h2 className="mb-3 text-lg font-bold text-cream">{t("tool.checkTitle")}</h2>
        <ul className="space-y-2 text-sm">
          <li className="flex justify-between"><span className="text-muted">{t("tool.binary")}</span><span className="text-cream">{formatUSD(binary)}</span></li>
          <li className="flex justify-between"><span className="text-muted">{t("tool.rankBonus")}</span><span className="text-cream">{formatUSD(bonoRango)}</span></li>
          <li className="flex justify-between"><span className="text-muted">{t("tool.direct")}</span><span className="text-cream">{formatUSD(directTotal)}</span></li>
        </ul>
        <div className="mt-3 border-t border-border pt-3 text-center">
          <p className="font-display text-4xl font-extrabold text-gold">{formatUSD(chequeTotal)}</p>
          <p className="mt-1 text-[11px] text-muted">{t("tool.disclaimer")}</p>
        </div>
      </Card>
    </div>
  );
}

function Result({ label, value }: { label: string; value: string }) {
  return (
    <div className="rounded-lg bg-navyDeep p-3 text-center">
      <p className="text-[10px] uppercase tracking-wide text-muted">{label}</p>
      <p className="mt-1 font-display text-base font-extrabold text-cream">{value}</p>
    </div>
  );
}
```

### 71. `components/dashboard/WalletAddresses.tsx`
Gestión de direcciones USDT.
```tsx
"use client";

import { useState } from "react";
import { Pencil, Plus } from "lucide-react";
import {
  saveWalletAddressAction,
  setDefaultWalletAddressAction,
} from "@/app/(distribuidor)/actions";
import { CoinBadge } from "@/components/dashboard/CoinBadge";
import { useT } from "@/components/i18n/LanguageProvider";
import type { WalletAddress } from "@/lib/data/distributor";

const CURRENCIES = ["USDT.TRC-20", "USDT.BEP-20", "BTC.Bitcoin"];

export function WalletAddresses({ addresses }: { addresses: WalletAddress[] }) {
  const { t } = useT();
  // null = lista; "new" = alta; id = edición.
  const [mode, setMode] = useState<null | "new" | string>(null);
  const editing = typeof mode === "string" && mode !== "new"
    ? addresses.find((a) => a.id === mode)
    : undefined;

  if (mode !== null) {
    return (
      <div>
        <div className="mb-4 flex items-center justify-between">
          <h2 className="text-lg font-bold text-cream">
            {editing ? t("wal.editWallet") : t("wal.newWallet")}
          </h2>
          <button onClick={() => setMode(null)} className="rounded-md border border-border px-3 py-1.5 text-xs text-cream hover:border-gold">
            « {t("common.back")}
          </button>
        </div>
        <form action={saveWalletAddressAction} className="space-y-4">
          {editing && <input type="hidden" name="id" value={editing.id} />}
          <label className="block">
            <span className="mb-1.5 block text-xs font-medium uppercase tracking-[0.08em] text-muted">
              {t("wal.crypto")} <span className="text-danger">*</span>
            </span>
            <select
              name="currency"
              defaultValue={editing?.currency ?? ""}
              required
              className="w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream outline-none focus:border-gold"
            >
              <option value="" disabled>{t("reg.select")}</option>
              {CURRENCIES.map((c) => (
                <option key={c} value={c}>{c}</option>
              ))}
            </select>
          </label>
          <label className="block">
            <span className="mb-1.5 block text-xs font-medium uppercase tracking-[0.08em] text-muted">
              {t("wal.walletAddress")} <span className="text-danger">*</span>
            </span>
            <input
              name="address"
              defaultValue={editing?.address ?? ""}
              required
              placeholder="T... / 0x..."
              className="w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream placeholder:text-muted/60 outline-none focus:border-gold"
            />
          </label>
          <button
            type="submit"
            onClick={() => setTimeout(() => setMode(null), 100)}
            className="rounded-[4px] bg-gold px-6 py-2.5 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight"
          >
            {t("common.save")}
          </button>
        </form>
      </div>
    );
  }

  return (
    <div>
      <div className="mb-4 flex items-center justify-between">
        <h2 className="text-lg font-bold text-cream">{t("wal.walletsTitle")}</h2>
        <button
          onClick={() => setMode("new")}
          className="inline-flex items-center gap-1.5 rounded-[4px] bg-gold px-4 py-2 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight"
        >
          <Plus size={14} /> {t("wal.newWallet")}
        </button>
      </div>

      {addresses.length === 0 ? (
        <p className="text-sm text-muted">{t("wal.noWallets")}</p>
      ) : (
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">{t("wal.colAddress")}</th>
                <th className="pb-2 font-semibold">{t("wal.colCurrency")}</th>
                <th className="pb-2 font-semibold text-center">{t("wal.colDefault")}</th>
                <th className="pb-2 font-semibold text-right">{t("wal.colActions")}</th>
              </tr>
            </thead>
            <tbody>
              {addresses.map((a) => (
                <tr key={a.id} className="border-b border-border/50">
                  <td className="max-w-[220px] truncate py-3 text-cobalt">{a.address}</td>
                  <td className="py-3"><CoinBadge currency={a.currency} /></td>
                  <td className="py-3 text-center">
                    <form action={setDefaultWalletAddressAction.bind(null, a.id)}>
                      <button
                        type="submit"
                        title="Marcar como más usada"
                        className={`h-4 w-4 rounded-full border-2 ${a.is_default ? "border-cobalt bg-cobalt" : "border-muted"}`}
                      />
                    </form>
                  </td>
                  <td className="py-3 text-right">
                    <button
                      onClick={() => setMode(a.id)}
                      className="inline-flex h-8 w-8 items-center justify-center rounded-md bg-gold text-goldInk hover:bg-goldLight"
                      title="Editar"
                    >
                      <Pencil size={14} />
                    </button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      )}
    </div>
  );
}
```

### 72. `components/dashboard/WithdrawForm.tsx`
Formulario de retiro.
```tsx
"use client";

import Link from "next/link";
import { useState } from "react";
import { useFormState, useFormStatus } from "react-dom";
import {
  requestWithdrawalAction,
  type WithdrawFormState,
} from "@/app/(distribuidor)/actions";
import { Field } from "@/components/ui/Input";
import { Button } from "@/components/ui/Button";
import { useT } from "@/components/i18n/LanguageProvider";
import type { WalletAddress } from "@/lib/data/distributor";

const initial: WithdrawFormState = {};

function SubmitButton() {
  const { pending } = useFormStatus();
  const { t } = useT();
  return (
    <Button type="submit" className="w-full" disabled={pending}>
      {pending ? t("wal.sending") : t("wal.requestBtn")}
    </Button>
  );
}

function trunc(a: string) {
  return a.length > 16 ? `${a.slice(0, 8)}…${a.slice(-6)}` : a;
}

export function WithdrawForm({
  balance,
  minWithdrawal,
  wallets,
}: {
  balance: number;
  minWithdrawal: number;
  wallets: WalletAddress[];
}) {
  const { t } = useT();
  const [state, formAction] = useFormState(requestWithdrawalAction, initial);
  const def = wallets.find((w) => w.is_default) ?? wallets[0];
  const [selected, setSelected] = useState(def?.address ?? "");

  if (wallets.length === 0) {
    return (
      <p className="rounded-md border border-gold/30 bg-gold/10 px-4 py-3 text-sm text-gold">
        {t("wal.noWallets")}{" "}
        <Link href="#" className="underline">
          {t("wal.newWallet")}
        </Link>{" "}
        ↓
      </p>
    );
  }

  return (
    <form action={formAction} className="space-y-4">
      <Field
        label={`${t("wal.amountLabel")} $${minWithdrawal}`}
        name="amount"
        type="number"
        step="0.01"
        min={minWithdrawal}
        max={balance}
        placeholder={`${t("wal.balance")}: $${balance.toFixed(2)}`}
        required
      />

      {/* Selección de cartera guardada (incluye red/moneda) */}
      <label className="block">
        <span className="mb-1.5 block text-xs font-medium uppercase tracking-[0.08em] text-muted">
          {t("wal.address")}
        </span>
        <select
          name="address"
          value={selected}
          onChange={(e) => setSelected(e.target.value)}
          required
          className="w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream outline-none focus:border-gold"
        >
          {wallets.map((w) => (
            <option key={w.id} value={w.address}>
              {w.currency} — {trunc(w.address)}
            </option>
          ))}
        </select>
      </label>

      {state.error && (
        <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">
          {state.error}
        </p>
      )}
      {state.ok && (
        <p className="rounded-md border border-success/30 bg-success/10 px-3 py-2 text-sm text-success">
          {state.ok}
        </p>
      )}
      <SubmitButton />
    </form>
  );
}
```

### 73. `components/dashboard/Store.tsx`
Tienda simple usada en la activación inicial (/bienvenida).
```tsx
"use client";

import { useFormState, useFormStatus } from "react-dom";
import {
  simulatePurchaseAction,
  type ActivationFormState,
} from "@/app/(distribuidor)/actions";
import { Card } from "@/components/ui/Card";
import { useT } from "@/components/i18n/LanguageProvider";

const initial: ActivationFormState = {};

interface Item {
  code: string;
  name: string;
  type: string;
  price: number;
  points: number;
  durationMonths: number;
  credits: number;
}

const TYPE_GROUPS: { type: string; titleKey: string; descKey: string }[] = [
  { type: "ibo", titleKey: "store.groupIbo", descKey: "store.groupIboDesc" },
  { type: "service", titleKey: "store.groupService", descKey: "store.groupServiceDesc" },
  { type: "physical", titleKey: "store.groupPhysical", descKey: "store.groupPhysicalDesc" },
];

function BuyButton({ code }: { code: string }) {
  const { pending } = useFormStatus();
  const { t } = useT();
  return (
    <button
      type="submit"
      name="productCode"
      value={code}
      disabled={pending}
      className="w-full rounded-[4px] bg-gold px-4 py-2 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk transition-all hover:bg-goldLight disabled:opacity-50"
    >
      {pending ? t("store.buying") : t("store.buy")}
    </button>
  );
}

export function Store({ catalog }: { catalog: Item[] }) {
  const { t } = useT();
  const [state, formAction] = useFormState(simulatePurchaseAction, initial);

  return (
    <form action={formAction} className="space-y-8">
      <p className="rounded-md border border-border bg-navyDeep px-3 py-2 text-xs text-muted">
        {t("store.simNote")}
      </p>

      {state.ok && (
        <p className="rounded-md border border-success/30 bg-success/10 px-3 py-2 text-sm text-success">
          {state.ok}
        </p>
      )}
      {state.error && (
        <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">
          {state.error}
        </p>
      )}

      {TYPE_GROUPS.map((group) => {
        const items = catalog.filter((c) => c.type === group.type);
        if (items.length === 0) return null;
        return (
          <section key={group.type}>
            <h2 className="text-lg font-bold text-cream">{t(group.titleKey)}</h2>
            <p className="mb-3 text-xs text-muted">{t(group.descKey)}</p>
            <div className="grid grid-cols-1 gap-3 sm:grid-cols-2 lg:grid-cols-3">
              {items.map((item) => (
                <Card key={item.code}>
                  <p className="font-display text-base font-semibold text-cream">{item.name}</p>
                  <p className="mt-1 text-2xl font-extrabold text-gold">${item.price}</p>
                  <p className="mt-1 text-xs text-muted">
                    {item.points} pts
                    {item.type === "service" && ` · ${item.durationMonths} ${t("store.months")}`}
                  </p>
                  {item.credits > 0 && (
                    <p className="mt-2 inline-flex items-center gap-1 rounded-full border border-gold/40 bg-gold/10 px-2.5 py-1 text-xs font-semibold text-gold">
                      ✦ {item.credits.toLocaleString()} {t("store.creditsMonth")}
                    </p>
                  )}
                  <div className="mt-4">
                    <BuyButton code={item.code} />
                  </div>
                </Card>
              ))}
            </div>
          </section>
        );
      })}
    </form>
  );
}
```

### 74. `components/dashboard/CreditStore.tsx`
Tienda de recarga de créditos de IA.
```tsx
"use client";

import { useState, useTransition } from "react";
import { Sparkles } from "lucide-react";
import { buyCreditsAction } from "@/app/(distribuidor)/actions";
import { Card } from "@/components/ui/Card";
import { useT } from "@/components/i18n/LanguageProvider";
import type { CreditPack } from "@/lib/ai/hub";

export function CreditStore({ packs, balance }: { packs: CreditPack[]; balance: number }) {
  const { t } = useT();
  const [bal, setBal] = useState(balance);
  const [msg, setMsg] = useState<string | null>(null);
  const [err, setErr] = useState<string | null>(null);
  const [pendingId, setPendingId] = useState<string | null>(null);
  const [, start] = useTransition();

  function buy(pack: CreditPack) {
    setErr(null);
    setMsg(null);
    setPendingId(pack.id);
    start(async () => {
      const r = await buyCreditsAction(pack.id);
      setPendingId(null);
      if (r.error) setErr(r.error);
      else {
        setMsg(r.ok ?? "");
        setBal((b) => b + pack.credits);
      }
    });
  }

  return (
    <section>
      <div className="flex items-center justify-between">
        <div>
          <h2 className="text-lg font-bold text-cream">{t("recharge.title")}</h2>
          <p className="mb-3 text-xs text-muted">{t("recharge.desc")}</p>
        </div>
        <span className="flex items-center gap-1 rounded-full border border-gold/40 bg-gold/10 px-3 py-1.5 text-sm font-semibold text-gold">
          <Sparkles size={13} /> {bal.toLocaleString()}
        </span>
      </div>

      {msg && (
        <p className="mb-3 rounded-md border border-success/30 bg-success/10 px-3 py-2 text-sm text-success">{msg}</p>
      )}
      {err && (
        <p className="mb-3 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{err}</p>
      )}

      <div className="grid grid-cols-2 gap-3 sm:grid-cols-4">
        {packs.map((p) => (
          <Card key={p.id}>
            <p className="font-display text-2xl font-extrabold text-gold">${p.priceUsd}</p>
            <p className="mt-1 flex items-center gap-1 text-sm font-semibold text-cream">
              <Sparkles size={13} className="text-gold" /> {p.credits.toLocaleString()}
            </p>
            {p.bonusLabel && (
              <span className="mt-1 inline-block rounded-full bg-success/15 px-2 py-0.5 text-[10px] font-semibold text-success">
                {p.bonusLabel}
              </span>
            )}
            <button
              onClick={() => buy(p)}
              disabled={pendingId !== null}
              className="mt-3 w-full rounded-[4px] bg-gold px-4 py-2 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk transition-all hover:bg-goldLight disabled:opacity-50"
            >
              {pendingId === p.id ? t("recharge.buying") : t("recharge.buy")}
            </button>
          </Card>
        ))}
      </div>
    </section>
  );
}
```

### 75. `components/academia/AcademiaView.tsx`
Vista de contenido de las academias.
```tsx
import { Card } from "@/components/ui/Card";
import { BookOpen, Lock, PlayCircle } from "lucide-react";
import { getT } from "@/lib/i18n/server";

interface Module {
  title: string;
  lessons: number;
  locked?: boolean;
}

/** Vista de una academia (Centro de Estudio). Contenido de muestra del prototipo. */
export function AcademiaView({
  name,
  tagline,
  modules,
}: {
  name: string;
  tagline: string;
  modules: Module[];
}) {
  const t = getT();
  return (
    <div className="space-y-8">
      <div>
        <p className="label">{t("aca.center")}</p>
        <h1 className="mt-1 flex items-center gap-2 text-3xl font-extrabold text-cream">
          <BookOpen className="text-gold" /> {t("aca.academy")} {name}
        </h1>
        <p className="mt-1 text-sm text-muted">{tagline}</p>
      </div>

      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
        {modules.map((m, i) => (
          <Card key={i}>
            <div className="flex items-start justify-between">
              <span className="flex h-9 w-9 items-center justify-center rounded-lg bg-navyDeep">
                {m.locked ? (
                  <Lock size={18} className="text-muted" />
                ) : (
                  <PlayCircle size={18} className="text-gold" />
                )}
              </span>
              <span className="text-[10px] uppercase tracking-wide text-muted">
                {m.lessons} {t("aca.lessons")}
              </span>
            </div>
            <p className="mt-3 font-display text-sm font-semibold text-cream">{m.title}</p>
            <p className="mt-1 text-xs text-muted">
              {m.locked ? t("aca.locked") : t("aca.available")}
            </p>
          </Card>
        ))}
      </div>

      <p className="text-xs text-muted">{t("aca.sample")}</p>
    </div>
  );
}
```

### 76. `components/profile/ProfileForm.tsx`
Formulario de perfil con subida de foto.
```tsx
"use client";

import { useRef, useState } from "react";
import { useFormState, useFormStatus } from "react-dom";
import { Camera, User } from "lucide-react";
import { updateProfileAction, type ProfileState } from "@/app/(distribuidor)/perfil/actions";
import { useT } from "@/components/i18n/LanguageProvider";
import { COUNTRIES } from "@/lib/data/countries";
import { Button } from "@/components/ui/Button";

const initial: ProfileState = {};
const inputCls =
  "w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream outline-none focus:border-gold";

export interface ProfileValues {
  firstName: string;
  lastName: string;
  gender: string | null;
  birthDate: string | null;
  country: string | null;
  phone: string | null;
  address: string | null;
  city: string | null;
  postalCode: string | null;
  avatarUrl: string | null;
}

function SubmitBtn() {
  const { pending } = useFormStatus();
  const { t } = useT();
  return (
    <Button type="submit" disabled={pending}>
      {pending ? "…" : t("profile.save")}
    </Button>
  );
}

export function ProfileForm({ values }: { values: ProfileValues }) {
  const { t } = useT();
  const [state, formAction] = useFormState(updateProfileAction, initial);
  const [country, setCountry] = useState(values.country ?? "");
  const [preview, setPreview] = useState<string | null>(values.avatarUrl);
  const fileRef = useRef<HTMLInputElement>(null);

  return (
    <form action={formAction} className="space-y-4">
      {/* Foto de perfil */}
      <div className="flex items-center gap-4">
        <button
          type="button"
          onClick={() => fileRef.current?.click()}
          className="group relative flex h-20 w-20 items-center justify-center overflow-hidden rounded-xl border-2 border-gold bg-navyDeep"
          title="Cambiar foto"
        >
          {preview ? (
            // eslint-disable-next-line @next/next/no-img-element
            <img src={preview} alt="avatar" className="h-full w-full object-cover" />
          ) : (
            <User size={36} className="text-muted" />
          )}
          <span className="absolute inset-0 flex items-center justify-center bg-navy/60 opacity-0 transition-opacity group-hover:opacity-100">
            <Camera size={18} className="text-gold" />
          </span>
        </button>
        <div className="text-xs text-muted">
          <p className="text-cream">{t("menu.profile")}</p>
          <p>Toca la foto para cambiarla. Recuerda guardar los cambios.</p>
        </div>
        <input
          ref={fileRef}
          type="file"
          name="avatar"
          accept="image/*"
          className="hidden"
          onChange={(e) => {
            const f = e.target.files?.[0];
            if (f) setPreview(URL.createObjectURL(f));
          }}
        />
      </div>

      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2">
        <Labeled label={t("profile.firstName")}>
          <input name="firstName" defaultValue={values.firstName} required className={inputCls} />
        </Labeled>
        <Labeled label={t("profile.lastName")}>
          <input name="lastName" defaultValue={values.lastName} required className={inputCls} />
        </Labeled>
        <Labeled label={t("profile.gender")}>
          <select name="gender" defaultValue={values.gender ?? ""} className={inputCls}>
            <option value="">—</option>
            <option value="M">{t("reg.genderM")}</option>
            <option value="F">{t("reg.genderF")}</option>
            <option value="X">{t("reg.genderX")}</option>
          </select>
        </Labeled>
        <Labeled label={t("profile.birthDate")}>
          <input type="date" name="birthDate" defaultValue={values.birthDate ?? ""} className={inputCls} />
        </Labeled>
        <Labeled label={t("profile.country")}>
          <select name="country" value={country} onChange={(e) => setCountry(e.target.value)} className={inputCls}>
            <option value="">—</option>
            {COUNTRIES.map((c) => (
              <option key={c.name} value={c.name}>{c.flag} {c.name}</option>
            ))}
          </select>
        </Labeled>
        <Labeled label={t("profile.phone")}>
          <input name="phone" defaultValue={values.phone ?? ""} className={inputCls} />
        </Labeled>
        <Labeled label={t("profile.address")}>
          <input name="address" defaultValue={values.address ?? ""} className={inputCls} />
        </Labeled>
        <Labeled label={t("profile.city")}>
          <input name="city" defaultValue={values.city ?? ""} className={inputCls} />
        </Labeled>
        <Labeled label={t("profile.postalCode")}>
          <input name="postalCode" defaultValue={values.postalCode ?? ""} className={inputCls} />
        </Labeled>
      </div>

      {state.error && <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{state.error}</p>}
      {state.ok && <p className="rounded-md border border-success/30 bg-success/10 px-3 py-2 text-sm text-success">{t("profile.saved")}</p>}
      <SubmitBtn />
    </form>
  );
}

function Labeled({ label, children }: { label: string; children: React.ReactNode }) {
  return (
    <label className="block">
      <span className="mb-1.5 block text-xs font-medium uppercase tracking-[0.08em] text-muted">{label}</span>
      {children}
    </label>
  );
}
```

### 77. `components/profile/PasswordForm.tsx`
Formulario de cambio de contraseña.
```tsx
"use client";

import { useState } from "react";
import { useFormState, useFormStatus } from "react-dom";
import { Check, X } from "lucide-react";
import { changePasswordAction, type ProfileState } from "@/app/(distribuidor)/perfil/actions";
import { useT } from "@/components/i18n/LanguageProvider";
import { checkPassword } from "@/lib/auth/password";
import { Button } from "@/components/ui/Button";

const initial: ProfileState = {};
const inputCls =
  "w-full rounded-md border border-border bg-navyDeep px-3 py-2.5 text-sm text-cream outline-none focus:border-gold";

const PWD_KEYS = ["pwd.min9", "pwd.max16", "pwd.upper", "pwd.lower", "pwd.number", "pwd.special"];

function SubmitBtn() {
  const { pending } = useFormStatus();
  const { t } = useT();
  return (
    <Button type="submit" disabled={pending}>
      {pending ? "…" : t("profile.changeBtn")}
    </Button>
  );
}

export function PasswordForm() {
  const { t } = useT();
  const [state, formAction] = useFormState(changePasswordAction, initial);
  const [pwd, setPwd] = useState("");
  const rules = checkPassword(pwd);

  return (
    <form action={formAction} className="max-w-md space-y-4">
      <label className="block">
        <span className="mb-1.5 block text-xs font-medium uppercase tracking-[0.08em] text-muted">{t("profile.newPassword")}</span>
        <input type="password" name="newPassword" value={pwd} onChange={(e) => setPwd(e.target.value)} required className={inputCls} />
        {pwd.length > 0 && (
          <ul className="mt-2 space-y-1 rounded-md border border-border bg-navyDeep p-2">
            {rules.map((r, i) => (
              <li key={i} className="flex items-center gap-2 text-[11px]">
                {r.ok ? <Check size={12} className="text-success" /> : <X size={12} className="text-danger" />}
                <span className={r.ok ? "text-success" : "text-muted"}>{t(PWD_KEYS[i])}</span>
              </li>
            ))}
          </ul>
        )}
      </label>
      <label className="block">
        <span className="mb-1.5 block text-xs font-medium uppercase tracking-[0.08em] text-muted">{t("profile.confirmPassword")}</span>
        <input type="password" name="confirmPassword" required className={inputCls} />
      </label>

      {state.error && <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{state.error}</p>}
      {state.ok && <p className="rounded-md border border-success/30 bg-success/10 px-3 py-2 text-sm text-success">{t("profile.pwdChanged")}</p>}
      <SubmitBtn />
    </form>
  );
}
```


---

# PARTE 8 — TIENDA E-COMMERCE
Carrito persistente, cupones y checkout. Vive ENCIMA del motor de comisiones sin alterarlo.

### 78. `components/tienda/CartContext.tsx`
Estado del carrito (localStorage), cupón, subtotal, descuento, envío (solo con producto físico) y puntos.
```tsx
"use client";

import {
  createContext,
  useCallback,
  useContext,
  useEffect,
  useMemo,
  useState,
  type ReactNode,
} from "react";

export interface CartItem {
  productId: string;
  code: string;
  slug: string;
  name: string;
  price: number;
  points: number;
  qty: number;
  imageUrl: string | null;
  /** service | physical | membership — decide si hay envío. */
  type: string;
  accent: string;
}

export interface AppliedCoupon {
  code: string;
  type: "percent" | "fixed";
  value: number;
}

interface CartState {
  items: CartItem[];
  isOpen: boolean;
  ready: boolean;
  coupon: AppliedCoupon | null;
  subtotal: number;
  discount: number;
  shipping: number;
  total: number;
  points: number;
  count: number;
  addItem: (item: Omit<CartItem, "qty">, qty?: number) => void;
  removeItem: (productId: string) => void;
  updateQty: (productId: string, qty: number) => void;
  clear: () => void;
  applyCoupon: (c: AppliedCoupon | null) => void;
  openCart: () => void;
  closeCart: () => void;
}

const CartCtx = createContext<CartState | null>(null);
const STORAGE_KEY = "triada-cart-v1";
const COUPON_KEY = "triada-cupon-v1";

/** Envío plano solo si el carrito lleva producto físico. */
export const FLAT_SHIPPING = 6;
export const FREE_SHIPPING_FROM = 120;

export function CartProvider({ children }: { children: ReactNode }) {
  const [items, setItems] = useState<CartItem[]>([]);
  const [coupon, setCoupon] = useState<AppliedCoupon | null>(null);
  const [isOpen, setIsOpen] = useState(false);
  const [ready, setReady] = useState(false);

  useEffect(() => {
    try {
      const raw = localStorage.getItem(STORAGE_KEY);
      if (raw) {
        const parsed = JSON.parse(raw) as CartItem[];
        if (Array.isArray(parsed)) setItems(parsed);
      }
      const c = localStorage.getItem(COUPON_KEY);
      if (c) setCoupon(JSON.parse(c) as AppliedCoupon);
    } catch {
      /* almacenamiento no disponible */
    }
    setReady(true);
  }, []);

  useEffect(() => {
    if (!ready) return;
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(items));
      if (coupon) localStorage.setItem(COUPON_KEY, JSON.stringify(coupon));
      else localStorage.removeItem(COUPON_KEY);
    } catch {
      /* ignorar */
    }
  }, [items, coupon, ready]);

  useEffect(() => {
    document.body.style.overflow = isOpen ? "hidden" : "";
    return () => {
      document.body.style.overflow = "";
    };
  }, [isOpen]);

  const addItem = useCallback((item: Omit<CartItem, "qty">, qty = 1) => {
    setItems((prev) => {
      const idx = prev.findIndex((p) => p.productId === item.productId);
      if (idx >= 0) {
        const next = [...prev];
        next[idx] = { ...next[idx], qty: Math.min(99, next[idx].qty + qty) };
        return next;
      }
      return [...prev, { ...item, qty }];
    });
    setIsOpen(true);
  }, []);

  const removeItem = useCallback((productId: string) => {
    setItems((prev) => prev.filter((p) => p.productId !== productId));
  }, []);

  const updateQty = useCallback((productId: string, qty: number) => {
    setItems((prev) =>
      prev.map((p) => (p.productId === productId ? { ...p, qty: Math.max(1, Math.min(99, qty)) } : p))
    );
  }, []);

  const clear = useCallback(() => {
    setItems([]);
    setCoupon(null);
  }, []);

  const subtotal = useMemo(() => items.reduce((s, i) => s + i.price * i.qty, 0), [items]);
  const count = useMemo(() => items.reduce((s, i) => s + i.qty, 0), [items]);
  const points = useMemo(() => items.reduce((s, i) => s + i.points * i.qty, 0), [items]);

  const discount = useMemo(() => {
    if (!coupon) return 0;
    const d = coupon.type === "percent" ? (subtotal * coupon.value) / 100 : coupon.value;
    return Math.min(subtotal, Math.round(d * 100) / 100);
  }, [coupon, subtotal]);

  const shipping = useMemo(() => {
    const hasPhysical = items.some((i) => i.type === "physical");
    if (!hasPhysical) return 0;
    return subtotal - discount >= FREE_SHIPPING_FROM ? 0 : FLAT_SHIPPING;
  }, [items, subtotal, discount]);

  const total = Math.max(0, subtotal - discount + shipping);

  const value: CartState = {
    items,
    isOpen,
    ready,
    coupon,
    subtotal,
    discount,
    shipping,
    total,
    points,
    count,
    addItem,
    removeItem,
    updateQty,
    clear,
    applyCoupon: setCoupon,
    openCart: () => setIsOpen(true),
    closeCart: () => setIsOpen(false),
  };

  return <CartCtx.Provider value={value}>{children}</CartCtx.Provider>;
}

export function useCart(): CartState {
  const ctx = useContext(CartCtx);
  if (!ctx) throw new Error("useCart debe usarse dentro de CartProvider");
  return ctx;
}
```

### 79. `components/tienda/CartDrawer.tsx`
Cajón lateral del carrito: cantidades, cupón, totales y confirmación de compra.
```tsx
"use client";

import { useState } from "react";
import { useRouter } from "next/navigation";
import { X, Trash2, Minus, Plus, ShoppingCart, Loader2, Tag, CheckCircle2 } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import { checkoutAction, validateCouponAction } from "@/app/(distribuidor)/tienda/actions";
import { useCart } from "./CartContext";

export function CartDrawer() {
  const { t } = useT();
  const router = useRouter();
  const {
    items,
    isOpen,
    closeCart,
    removeItem,
    updateQty,
    clear,
    subtotal,
    discount,
    shipping,
    total,
    points,
    coupon,
    applyCoupon,
  } = useCart();

  const [code, setCode] = useState("");
  const [couponError, setCouponError] = useState<string | null>(null);
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [done, setDone] = useState<string | null>(null);

  async function tryCoupon() {
    setCouponError(null);
    const res = await validateCouponAction(code, subtotal);
    if ("error" in res) {
      setCouponError(res.error);
      return;
    }
    applyCoupon(res);
    setCode("");
  }

  async function pay() {
    if (items.length === 0 || busy) return;
    setBusy(true);
    setError(null);
    try {
      const res = await checkoutAction({
        items: items.map((i) => ({ productId: i.productId, qty: i.qty })),
        couponCode: coupon?.code ?? null,
      });
      if (res.error) {
        setError(res.error);
        return;
      }
      setDone(res.orderNumber ?? "");
      clear();
      router.refresh();
    } catch {
      setError(t("store.checkoutError"));
    } finally {
      setBusy(false);
    }
  }

  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 z-50 flex justify-end" onClick={closeCart}>
      <div className="absolute inset-0 bg-black/60" />
      <aside
        className="relative flex h-full w-full max-w-md flex-col border-l border-border bg-navyDeep shadow-card"
        onClick={(e) => e.stopPropagation()}
      >
        <header className="flex items-center justify-between border-b border-border px-5 py-4">
          <h2 className="flex items-center gap-2 text-sm font-semibold text-cream">
            <ShoppingCart size={16} className="text-gold" /> {t("store.cart")}
          </h2>
          <button onClick={closeCart} className="text-muted hover:text-cream" aria-label={t("common.close")}>
            <X size={18} />
          </button>
        </header>

        {done !== null ? (
          <div className="flex flex-1 flex-col items-center justify-center gap-3 p-6 text-center">
            <CheckCircle2 size={44} className="text-success" />
            <p className="text-lg font-semibold text-cream">{t("store.thanks")}</p>
            <p className="text-sm text-muted">
              {t("store.orderNumber")}: <span className="font-mono text-cream">{done}</span>
            </p>
            <button
              onClick={() => {
                setDone(null);
                closeCart();
              }}
              className="mt-2 rounded-xl bg-gold px-5 py-2.5 text-sm font-semibold text-goldInk hover:bg-goldLight"
            >
              {t("common.close")}
            </button>
          </div>
        ) : items.length === 0 ? (
          <div className="flex flex-1 flex-col items-center justify-center gap-2 p-6 text-center">
            <ShoppingCart size={36} className="text-muted" />
            <p className="text-sm text-muted">{t("store.cartEmpty")}</p>
          </div>
        ) : (
          <>
            <div className="flex-1 space-y-3 overflow-y-auto p-4">
              {items.map((i) => (
                <div key={i.productId} className="flex gap-3 rounded-card border border-border bg-card p-3">
                  <div className="h-16 w-16 shrink-0 overflow-hidden rounded-lg bg-navyDeep">
                    {i.imageUrl && (
                      // eslint-disable-next-line @next/next/no-img-element
                      <img src={i.imageUrl} alt={i.name} className="h-full w-full object-cover" />
                    )}
                  </div>
                  <div className="min-w-0 flex-1">
                    <p className="truncate text-sm font-medium text-cream">{i.name}</p>
                    <p className="text-[11px] text-muted">
                      ${i.price} · {i.points} {t("store.points")}
                    </p>
                    <div className="mt-1.5 flex items-center gap-2">
                      <button
                        onClick={() => updateQty(i.productId, i.qty - 1)}
                        className="rounded border border-border p-1 text-muted hover:text-cream"
                        aria-label="-"
                      >
                        <Minus size={11} />
                      </button>
                      <span className="w-6 text-center text-xs text-cream">{i.qty}</span>
                      <button
                        onClick={() => updateQty(i.productId, i.qty + 1)}
                        className="rounded border border-border p-1 text-muted hover:text-cream"
                        aria-label="+"
                      >
                        <Plus size={11} />
                      </button>
                      <button
                        onClick={() => removeItem(i.productId)}
                        className="ml-auto text-muted hover:text-danger"
                        aria-label={t("common.delete")}
                      >
                        <Trash2 size={13} />
                      </button>
                    </div>
                  </div>
                </div>
              ))}
            </div>

            <div className="space-y-3 border-t border-border p-4">
              {coupon ? (
                <div className="flex items-center justify-between rounded-lg border border-gold/40 bg-gold/10 px-3 py-2 text-xs">
                  <span className="flex items-center gap-1.5 text-gold">
                    <Tag size={12} /> {coupon.code}
                  </span>
                  <button onClick={() => applyCoupon(null)} className="text-muted hover:text-danger">
                    <X size={12} />
                  </button>
                </div>
              ) : (
                <div>
                  <div className="flex gap-2">
                    <input
                      value={code}
                      onChange={(e) => setCode(e.target.value.toUpperCase())}
                      placeholder={t("store.couponPlaceholder")}
                      className="min-w-0 flex-1 rounded-lg border border-border bg-card px-3 py-2 text-xs text-cream outline-none focus:border-gold"
                    />
                    <button
                      onClick={tryCoupon}
                      disabled={!code.trim()}
                      className="rounded-lg border border-border px-3 py-2 text-xs font-medium text-cream hover:border-gold disabled:opacity-40"
                    >
                      {t("store.apply")}
                    </button>
                  </div>
                  {couponError && <p className="mt-1 text-[11px] text-danger">{couponError}</p>}
                </div>
              )}

              <dl className="space-y-1 text-xs">
                <Row label={t("store.subtotal")} value={`$${subtotal.toFixed(2)}`} />
                {discount > 0 && (
                  <Row label={t("store.discount")} value={`−$${discount.toFixed(2)}`} accent="text-success" />
                )}
                {shipping > 0 && <Row label={t("store.shipping")} value={`$${shipping.toFixed(2)}`} />}
                <Row label={t("store.pointsEarned")} value={`${points} pts`} accent="text-gold" />
                <div className="flex justify-between border-t border-border pt-2 text-sm font-bold text-cream">
                  <dt>{t("store.total")}</dt>
                  <dd>${total.toFixed(2)}</dd>
                </div>
              </dl>

              {error && (
                <p className="rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-xs text-danger">
                  {error}
                </p>
              )}

              <button
                onClick={pay}
                disabled={busy}
                className="flex w-full items-center justify-center gap-2 rounded-xl bg-gold px-4 py-3 text-sm font-semibold text-goldInk transition-opacity hover:bg-goldLight disabled:opacity-50"
              >
                {busy ? <Loader2 size={16} className="animate-spin" /> : <ShoppingCart size={16} />}
                {busy ? t("store.processing") : t("store.checkout")}
              </button>
              <p className="text-center text-[10px] text-muted">{t("store.simulatedNote")}</p>
            </div>
          </>
        )}
      </aside>
    </div>
  );
}

function Row({ label, value, accent }: { label: string; value: string; accent?: string }) {
  return (
    <div className="flex justify-between">
      <dt className="text-muted">{label}</dt>
      <dd className={accent ?? "text-cream"}>{value}</dd>
    </div>
  );
}
```

### 80. `components/tienda/ProductCard.tsx`
Tarjeta de producto con imagen, insignia, puntos, créditos de IA y precio.
```tsx
"use client";

import { ShoppingCart, Sparkles, ImageOff } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { StoreProduct } from "@/lib/data/store";
import { useCart } from "./CartContext";

export function ProductCard({ product }: { product: StoreProduct }) {
  const { t } = useT();
  const { addItem } = useCart();

  const monthly = product.durationMonths > 1 ? product.price / product.durationMonths : null;

  return (
    <article className="group flex flex-col overflow-hidden rounded-card border border-border bg-card transition-colors hover:border-gold/50">
      <div className="relative aspect-[4/3] w-full overflow-hidden bg-navyDeep">
        {product.imageUrl ? (
          // eslint-disable-next-line @next/next/no-img-element
          <img
            src={product.imageUrl}
            alt={product.name}
            className="h-full w-full object-cover transition-transform duration-300 group-hover:scale-105"
          />
        ) : (
          <div className="flex h-full w-full items-center justify-center text-muted">
            <ImageOff size={28} />
          </div>
        )}
        {product.badge && (
          <span
            className="absolute left-3 top-3 rounded-full px-2.5 py-1 text-[10px] font-bold uppercase tracking-wide text-goldInk"
            style={{ backgroundColor: product.accent }}
          >
            {product.badge}
          </span>
        )}
      </div>

      <div className="flex flex-1 flex-col p-4">
        <h3 className="text-sm font-semibold text-cream">{product.name}</h3>
        {product.description && (
          <p className="mt-1 line-clamp-2 text-[12px] leading-snug text-muted">{product.description}</p>
        )}

        <div className="mt-3 flex flex-wrap items-center gap-1.5">
          {product.points > 0 && (
            <span className="rounded-md border border-border px-2 py-0.5 text-[10px] text-muted">
              {product.points} {t("store.points")}
            </span>
          )}
          {product.durationMonths > 1 && (
            <span className="rounded-md border border-border px-2 py-0.5 text-[10px] text-muted">
              {product.durationMonths} {t("store.months")}
            </span>
          )}
          {product.aiCreditsMonthly > 0 && (
            <span className="flex items-center gap-1 rounded-md border border-gold/40 bg-gold/10 px-2 py-0.5 text-[10px] text-gold">
              <Sparkles size={9} /> {product.aiCreditsMonthly.toLocaleString()}
            </span>
          )}
        </div>

        <div className="mt-auto flex items-end justify-between gap-2 pt-4">
          <div>
            {product.compareAtPrice && product.compareAtPrice > product.price && (
              <p className="text-[11px] text-muted line-through">${product.compareAtPrice}</p>
            )}
            <p className="text-xl font-bold text-cream">${product.price}</p>
            {monthly && (
              <p className="text-[10px] text-muted">
                ${monthly.toFixed(0)}/{t("store.perMonth")}
              </p>
            )}
          </div>
          <button
            onClick={() =>
              addItem({
                productId: product.id,
                code: product.code,
                slug: product.slug,
                name: product.name,
                price: product.price,
                points: product.points,
                imageUrl: product.imageUrl,
                type: product.type,
                accent: product.accent,
              })
            }
            className="flex items-center gap-1.5 rounded-xl bg-gold px-3 py-2 text-xs font-semibold text-goldInk transition-opacity hover:bg-goldLight"
          >
            <ShoppingCart size={14} /> {t("store.add")}
          </button>
        </div>
      </div>
    </article>
  );
}
```

### 81. `components/tienda/StoreClient.tsx`
Catálogo con filtros, buscador y contador del carrito.
```tsx
"use client";

import { useMemo, useState } from "react";
import { ShoppingCart, Search } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { StoreProduct } from "@/lib/data/store";
import { CartProvider, useCart } from "./CartContext";
import { CartDrawer } from "./CartDrawer";
import { ProductCard } from "./ProductCard";

type Filter = "todo" | "service" | "physical" | "membership";

const FILTERS: Filter[] = ["todo", "service", "physical", "membership"];

export function StoreClient({ products }: { products: StoreProduct[] }) {
  return (
    <CartProvider>
      <StoreInner products={products} />
      <CartDrawer />
    </CartProvider>
  );
}

function StoreInner({ products }: { products: StoreProduct[] }) {
  const { t } = useT();
  const { count, openCart } = useCart();
  const [filter, setFilter] = useState<Filter>("todo");
  const [query, setQuery] = useState("");

  const list = useMemo(() => {
    const q = query.trim().toLowerCase();
    return products.filter((p) => {
      if (filter !== "todo" && p.type !== filter) return false;
      if (!q) return true;
      return p.name.toLowerCase().includes(q) || p.description.toLowerCase().includes(q);
    });
  }, [products, filter, query]);

  return (
    <>
      <div className="mb-5 flex flex-wrap items-center justify-between gap-3">
        <div className="flex gap-1 rounded-lg bg-navyDeep p-1">
          {FILTERS.map((f) => (
            <button
              key={f}
              onClick={() => setFilter(f)}
              className={`rounded-md px-3 py-1.5 text-sm transition-colors ${
                filter === f ? "bg-gold text-goldInk" : "text-muted hover:text-cream"
              }`}
            >
              {t(`store.filter.${f}`)}
            </button>
          ))}
        </div>

        <div className="flex items-center gap-2">
          <label className="flex items-center gap-2 rounded-lg border border-border bg-navyDeep px-3 py-2">
            <Search size={14} className="shrink-0 text-muted" />
            <input
              value={query}
              onChange={(e) => setQuery(e.target.value)}
              placeholder={t("store.search")}
              className="w-40 bg-transparent text-sm text-cream outline-none placeholder:text-muted"
            />
          </label>
          <button
            onClick={openCart}
            className="relative flex items-center gap-2 rounded-xl bg-gold px-4 py-2.5 text-sm font-semibold text-goldInk hover:bg-goldLight"
          >
            <ShoppingCart size={16} />
            {t("store.cart")}
            {count > 0 && (
              <span className="absolute -right-1.5 -top-1.5 flex h-5 min-w-5 items-center justify-center rounded-full bg-navyDeep px-1 text-[10px] font-bold text-gold">
                {count}
              </span>
            )}
          </button>
        </div>
      </div>

      {list.length === 0 ? (
        <p className="py-20 text-center text-sm text-muted">{t("store.noResults")}</p>
      ) : (
        <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4">
          {list.map((p) => (
            <ProductCard key={p.id} product={p} />
          ))}
        </div>
      )}
    </>
  );
}
```


---

# PARTE 9 — MI RED (VISOR INTERACTIVO)
Árbol binario y unilevel con ReactFlow: foto, código de referido, insignia de rango, plegar ramas y colocar pendientes.

### 82. `components/tree/MlmStructure.tsx`
El visor completo: tarjeta de nodo, expediente del distribuidor, buscador, leyenda, minimapa, layout automático, plegado de ramas y colocación de pendientes.
```tsx
"use client";

import { useState, useMemo, useCallback, useEffect } from 'react';
import ReactFlow, {
  Background,
  Controls,
  MiniMap,
  Handle,
  Position,
  useNodesState,
  useEdgesState,
  useReactFlow,
  ReactFlowProvider,
  Node,
  Edge,
  NodeProps,
} from 'reactflow';
import 'reactflow/dist/style.css';
import { RankBadge } from '@/components/ui/RankBadge';
import { rankLevelByName } from '@/lib/ranks';
import {
  Crown,
  Gem,
  Award,
  Shield,
  Sparkles,
  GitBranch,
  Network,
  Users,
  Search,
  Maximize2,
  Copy,
  Check,
  Coins,
  ChevronRight,
  X,
  Layers,
  ArrowRightLeft,
  DollarSign,
  Calendar,
  Globe,
  Mail,
  Zap,
  Plus,
  Minus,
} from 'lucide-react';

/* ==========================================================================
   TIPOS Y MODELOS DE DATOS (MULTILEVEL MARKETING)
   ========================================================================== */

/** Los 13 rangos de TRIADA, más 'Sin rango' para quien aún no califica. */
export type RankType =
  | 'Sin rango'
  | 'PIONERO'
  | 'IMPULSOR'
  | 'ÉLITE'
  | 'EXECUTIVE'
  | 'VISIONARIO'
  | 'CONQUISTADOR'
  | 'SOBERANO'
  | 'DIAMANTE'
  | 'DIAMANTE IMPERIAL'
  | 'DIAMANTE MONARCA'
  | 'LEYENDA'
  | 'LEYENDA TRIADA'
  | 'LEYENDA CORONA';

export type StatusType = 'Activo' | 'Inactivo';
export type TreeStructureType = 'binary' | 'unilevel';

export interface BaseMemberData {
  id: string;
  /** Código de referido visible (ej. DARNP8G8). Es el ID que usa el afiliado. */
  code: string;
  name: string;
  avatarUrl?: string | null;
  rank: RankType;
  status: StatusType;
  level: number;
  positionLabel: string;
  uplineName?: string;
  joinDate: string;
  email: string;
  country: string;
  avatarInitials: string;
}

export interface BinaryMemberData extends BaseMemberData {
  parentId: string | null;
  leg: 'izquierda' | 'derecha' | null;
  puntos_izquierda: number;
  puntos_derecha: number;
  pierna_pago: 'izquierda' | 'derecha';
  ciclos_binarios: number;
  treeType: 'binary';
}

export interface UnilevelMemberData extends BaseMemberData {
  parentId: string | null;
  volumen_grupal_puntos: number;
  volumen_personal_puntos: number;
  lineas_directas: number;
  treeType: 'unilevel';
}

export type MemberNodeData = (BinaryMemberData | UnilevelMemberData) & {
  isSelected?: boolean;
  onSelectNode?: (data: MemberNodeData) => void;
  /** Descendientes directos existentes (para saber si se puede plegar). */
  childCount?: number;
  isCollapsed?: boolean;
  onToggleCollapse?: (id: string) => void;
  /** Piernas binarias libres: habilitan el "+" para colocar un pendiente. */
  freeLegs?: { izquierda: boolean; derecha: boolean };
  onPlace?: (parentId: string, leg: 'izquierda' | 'derecha') => void;
};

/* ==========================================================================
   CONFIGURACIÓN DE RANGOS Y METADATOS VISUALES (LUJO & CORPORATIVO)
   ========================================================================== */

const RANK_METADATA: Record<
  RankType,
  {
    icon: typeof Crown;
    badgeBg: string;
    badgeText: string;
    badgeBorder: string;
    glowColor: string;
  }
> = {
  'LEYENDA CORONA': {
    icon: Crown,
    badgeBg: 'bg-gradient-to-r from-amber-500/25 via-yellow-400/25 to-amber-600/25',
    badgeText: 'text-[#FDE047]',
    badgeBorder: 'border-[#D4AF37]',
    glowColor: 'rgba(212, 175, 55, 0.4)',
  },
  'LEYENDA TRIADA': {
    icon: Crown,
    badgeBg: 'bg-gradient-to-r from-amber-500/20 to-yellow-500/20',
    badgeText: 'text-amber-200',
    badgeBorder: 'border-[#D4AF37]/70',
    glowColor: 'rgba(212, 175, 55, 0.35)',
  },
  LEYENDA: {
    icon: Crown,
    badgeBg: 'bg-gradient-to-r from-amber-500/15 to-yellow-500/15',
    badgeText: 'text-amber-300',
    badgeBorder: 'border-amber-400/50',
    glowColor: 'rgba(245, 158, 11, 0.3)',
  },
  'DIAMANTE MONARCA': {
    icon: Gem,
    badgeBg: 'bg-gradient-to-r from-cyan-500/20 via-sky-400/20 to-blue-500/20',
    badgeText: 'text-cyan-300',
    badgeBorder: 'border-cyan-400/60',
    glowColor: 'rgba(56, 189, 248, 0.35)',
  },
  'DIAMANTE IMPERIAL': {
    icon: Gem,
    badgeBg: 'bg-gradient-to-r from-sky-500/20 to-indigo-500/20',
    badgeText: 'text-sky-300',
    badgeBorder: 'border-sky-400/50',
    glowColor: 'rgba(14, 165, 233, 0.3)',
  },
  DIAMANTE: {
    icon: Gem,
    badgeBg: 'bg-gradient-to-r from-sky-500/15 to-indigo-500/15',
    badgeText: 'text-sky-200',
    badgeBorder: 'border-sky-400/40',
    glowColor: 'rgba(14, 165, 233, 0.25)',
  },
  SOBERANO: {
    icon: Sparkles,
    badgeBg: 'bg-gradient-to-r from-emerald-500/20 to-teal-500/20',
    badgeText: 'text-emerald-300',
    badgeBorder: 'border-emerald-500/50',
    glowColor: 'rgba(16, 185, 129, 0.3)',
  },
  CONQUISTADOR: {
    icon: Gem,
    badgeBg: 'bg-gradient-to-r from-rose-500/20 to-pink-500/20',
    badgeText: 'text-rose-300',
    badgeBorder: 'border-rose-500/50',
    glowColor: 'rgba(244, 63, 94, 0.3)',
  },
  VISIONARIO: {
    icon: Award,
    badgeBg: 'bg-gradient-to-r from-violet-500/20 to-purple-500/20',
    badgeText: 'text-violet-300',
    badgeBorder: 'border-violet-400/50',
    glowColor: 'rgba(139, 92, 246, 0.3)',
  },
  EXECUTIVE: {
    icon: Award,
    badgeBg: 'bg-gradient-to-r from-slate-400/20 to-zinc-400/20',
    badgeText: 'text-slate-200',
    badgeBorder: 'border-slate-300/50',
    glowColor: 'rgba(226, 232, 240, 0.25)',
  },
  'ÉLITE': {
    icon: Shield,
    badgeBg: 'bg-gradient-to-r from-blue-500/20 to-indigo-500/20',
    badgeText: 'text-blue-300',
    badgeBorder: 'border-blue-500/50',
    glowColor: 'rgba(59, 130, 246, 0.3)',
  },
  IMPULSOR: {
    icon: Shield,
    badgeBg: 'bg-gradient-to-r from-teal-500/15 to-cyan-500/15',
    badgeText: 'text-teal-300',
    badgeBorder: 'border-teal-500/40',
    glowColor: 'rgba(20, 184, 166, 0.25)',
  },
  PIONERO: {
    icon: Shield,
    badgeBg: 'bg-gradient-to-r from-slate-500/15 to-slate-600/15',
    badgeText: 'text-slate-300',
    badgeBorder: 'border-slate-500/40',
    glowColor: 'rgba(148, 163, 184, 0.2)',
  },
  'Sin rango': {
    icon: Shield,
    badgeBg: 'bg-slate-700/20',
    badgeText: 'text-slate-400',
    badgeBorder: 'border-slate-600/40',
    glowColor: 'rgba(100, 116, 139, 0.15)',
  },
};

/* ==========================================================================
   LAYOUT AUTOMÁTICO
   El prototipo traía posiciones fijas para 7 nodos de ejemplo. Aquí se calculan
   a partir del árbol real: el binario se reparte por profundidad y el unilevel
   agrupa cada rama bajo su padre.
   ========================================================================== */

const NODE_W = 300;
const H_GAP = 40;
const V_GAP = 220;

/** Posiciona un árbol binario: cada nivel se separa en vertical y las hojas se
 *  reparten en horizontal, centrando cada padre sobre sus hijos. */
function layoutBinary(members: BinaryMemberData[]): Record<string, { x: number; y: number }> {
  const byParent = new Map<string, BinaryMemberData[]>();
  const roots: BinaryMemberData[] = [];
  const ids = new Set(members.map((m) => m.id));
  for (const m of members) {
    if (m.parentId && ids.has(m.parentId)) {
      const arr = byParent.get(m.parentId) ?? [];
      arr.push(m);
      byParent.set(m.parentId, arr);
    } else {
      roots.push(m);
    }
  }
  for (const arr of byParent.values()) {
    arr.sort((a, b) => (a.leg === 'izquierda' ? -1 : 1) - (b.leg === 'izquierda' ? -1 : 1));
  }

  const pos: Record<string, { x: number; y: number }> = {};
  let cursor = 0;

  function walk(node: BinaryMemberData, depth: number): number {
    const kids = byParent.get(node.id) ?? [];
    if (kids.length === 0) {
      const x = cursor * (NODE_W + H_GAP);
      cursor += 1;
      pos[node.id] = { x, y: depth * V_GAP };
      return x;
    }
    const xs = kids.map((k) => walk(k, depth + 1));
    const x = (Math.min(...xs) + Math.max(...xs)) / 2;
    pos[node.id] = { x, y: depth * V_GAP };
    return x;
  }

  for (const r of roots) walk(r, 0);
  return pos;
}

/** Mismo reparto para el unilevel, que admite N hijos por nodo. */
function layoutUnilevel(members: UnilevelMemberData[]): Record<string, { x: number; y: number }> {
  const byParent = new Map<string, UnilevelMemberData[]>();
  const roots: UnilevelMemberData[] = [];
  const ids = new Set(members.map((m) => m.id));
  for (const m of members) {
    if (m.parentId && ids.has(m.parentId)) {
      const arr = byParent.get(m.parentId) ?? [];
      arr.push(m);
      byParent.set(m.parentId, arr);
    } else {
      roots.push(m);
    }
  }

  const pos: Record<string, { x: number; y: number }> = {};
  let cursor = 0;

  function walk(node: UnilevelMemberData, depth: number): number {
    const kids = byParent.get(node.id) ?? [];
    if (kids.length === 0) {
      const x = cursor * (NODE_W + H_GAP);
      cursor += 1;
      pos[node.id] = { x, y: depth * V_GAP };
      return x;
    }
    const xs = kids.map((k) => walk(k, depth + 1));
    const x = (Math.min(...xs) + Math.max(...xs)) / 2;
    pos[node.id] = { x, y: depth * V_GAP };
    return x;
  }

  for (const r of roots) walk(r, 0);
  return pos;
}

const EDGE_LABEL_COMMON = {
  labelStyle: { fill: '#F5D77F', fontWeight: 700, fontSize: 10 },
  labelBgStyle: {
    fill: '#0A1424',
    fillOpacity: 0.95,
    stroke: 'rgba(212,175,55,0.5)',
    strokeWidth: 1,
    rx: 6,
    ry: 6,
  },
  labelBgPadding: [6, 4] as [number, number],
};

function buildBinaryTreeElements(
  members: BinaryMemberData[],
  selectedId: string | null,
  onSelectNode: (data: MemberNodeData) => void
): { nodes: Node<MemberNodeData>[]; edges: Edge[] } {
  const positions = layoutBinary(members);
  const ids = new Set(members.map((m) => m.id));

  const nodes: Node<MemberNodeData>[] = members.map((member) => ({
    id: member.id,
    type: 'mlmNode',
    position: positions[member.id] ?? { x: 0, y: 0 },
    data: {
      ...member,
      isSelected: member.id === selectedId,
      onSelectNode,
    },
  }));

  const edges: Edge[] = members
    .filter((m) => m.parentId && ids.has(m.parentId))
    .map((m) => ({
      id: `e-${m.parentId}-${m.id}`,
      source: m.parentId as string,
      target: m.id,
      type: 'smoothstep',
      label: m.leg === 'izquierda' ? 'Izq' : 'Der',
      ...EDGE_LABEL_COMMON,
      style: { stroke: '#D4AF37', strokeWidth: 2 },
    }));

  return { nodes, edges };
}

function buildUnilevelTreeElements(
  members: UnilevelMemberData[],
  selectedId: string | null,
  onSelectNode: (data: MemberNodeData) => void
): { nodes: Node<MemberNodeData>[]; edges: Edge[] } {
  const positions = layoutUnilevel(members);
  const ids = new Set(members.map((m) => m.id));

  const nodes: Node<MemberNodeData>[] = members.map((member) => ({
    id: member.id,
    type: 'mlmNode',
    position: positions[member.id] ?? { x: 0, y: 0 },
    data: {
      ...member,
      isSelected: member.id === selectedId,
      onSelectNode,
    },
  }));

  const edges: Edge[] = members
    .filter((m) => m.parentId && ids.has(m.parentId))
    .map((m) => ({
      id: `e-${m.parentId}-${m.id}`,
      source: m.parentId as string,
      target: m.id,
      type: 'smoothstep',
      label: `Nivel ${m.level}`,
      ...EDGE_LABEL_COMMON,
      style: { stroke: '#D4AF37', strokeWidth: 2 },
    }));

  return { nodes, edges };
}

/* ==========================================================================
   COMPONENTE DE NODO PERSONALIZADO (MLM NODE)
   Tarjeta estilizada con estética corporativa de lujo, bordes dorados,
   azul marino profundo (#0A1424 / #0F223F) y micro-animación en hover
   ========================================================================== */

function MLMNodeCard({ data }: NodeProps<MemberNodeData>) {
  const rankMeta = RANK_METADATA[data.rank] || RANK_METADATA['Sin rango'];
  const RankIcon = rankMeta.icon;
  const isBinary = data.treeType === 'binary';
  const binaryData = isBinary ? (data as BinaryMemberData) : null;
  const unilevelData = !isBinary ? (data as UnilevelMemberData) : null;

  return (
    <div
      onClick={() => data.onSelectNode?.(data)}
      className={`group relative w-[240px] cursor-pointer select-none rounded-2xl p-3.5 transition-all duration-300 ease-out
        ${
          data.isSelected
            ? 'scale-[1.04] bg-[#0F223F] ring-2 ring-[#D4AF37] shadow-[0_0_28px_rgba(212,175,55,0.45)] border-[#D4AF37]'
            : 'bg-[#0A1424]/95 hover:bg-[#0F223F] hover:scale-105 hover:-translate-y-1 hover:border-[#D4AF37] hover:shadow-[0_0_24px_rgba(212,175,55,0.3)] border border-[#D4AF37]/35 shadow-[0_10px_25px_rgba(0,0,0,0.6)]'
        }
      `}
    >
      {/* ReactFlow Handles para conexiones de entrada y salida */}
      <Handle
        type="target"
        position={Position.Top}
        className="!w-3 !h-3 !border-2 !border-[#050B14] !bg-[#D4AF37] !rounded-full -top-1.5 transition-transform group-hover:scale-125"
      />
      <Handle
        type="source"
        position={Position.Bottom}
        className="!w-3 !h-3 !border-2 !border-[#050B14] !bg-[#D4AF37] !rounded-full -bottom-1.5 transition-transform group-hover:scale-125"
      />

      {/* Punto superior: pliega/despliega la rama (solo si tiene descendientes). */}
      {(data.childCount ?? 0) > 0 && (
        <button
          onClick={(e) => {
            e.stopPropagation();
            data.onToggleCollapse?.(data.id);
          }}
          title={data.isCollapsed ? 'Desplegar rama' : 'Plegar rama'}
          className="absolute -top-3 left-1/2 z-20 flex h-6 w-6 -translate-x-1/2 items-center justify-center rounded-full border-2 border-[#050B14] bg-[#D4AF37] text-[#050B14] shadow-[0_0_10px_rgba(212,175,55,0.6)] transition-transform hover:scale-110"
        >
          {data.isCollapsed ? <Plus className="h-3.5 w-3.5" /> : <Minus className="h-3.5 w-3.5" />}
        </button>
      )}

      {/* Puntos inferiores: colocar un referido pendiente en la pierna libre. */}
      {isBinary && data.freeLegs && (
        <div className="absolute -bottom-3.5 left-1/2 z-20 flex -translate-x-1/2 gap-6 opacity-0 transition-opacity group-hover:opacity-100">
          {(['izquierda', 'derecha'] as const).map((leg) =>
            data.freeLegs![leg] ? (
              <button
                key={leg}
                onClick={(e) => {
                  e.stopPropagation();
                  data.onPlace?.(data.id, leg);
                }}
                title={`Colocar en pierna ${leg}`}
                className="flex h-6 w-6 items-center justify-center rounded-full border-2 border-[#050B14] bg-emerald-500 text-[#050B14] shadow-[0_0_10px_rgba(16,185,129,0.6)] transition-transform hover:scale-110"
              >
                <Plus className="h-3.5 w-3.5" />
              </button>
            ) : (
              <span key={leg} className="h-6 w-6" />
            )
          )}
        </div>
      )}

      {/* Brillo ambiental dorado en la parte superior del nodo */}
      <div className="pointer-events-none absolute inset-x-0 top-0 h-10 rounded-t-2xl bg-gradient-to-b from-[#D4AF37]/10 to-transparent" />

      {/* Barra Superior: Nivel & Indicador de Estado */}
      <div className="relative mb-2.5 flex items-center justify-between">
        <span className="inline-flex items-center gap-1 rounded-md border border-[#D4AF37]/30 bg-[#050B14]/80 px-1.5 py-0.5 text-[10px] font-semibold text-[#F5D77F]">
          <Layers className="h-2.5 w-2.5 text-[#D4AF37]" />
          L{data.level}
        </span>

        {/* Indicador de Estado (Luz verde / roja brillante) */}
        <div className="inline-flex items-center gap-1.5 rounded-full border border-slate-700/60 bg-[#050B14]/80 px-2 py-0.5">
          {data.status === 'Activo' ? (
            <>
              <span className="relative flex h-2 w-2">
                <span className="absolute inline-flex h-full w-full animate-ping rounded-full bg-emerald-400 opacity-75" />
                <span className="relative inline-flex h-2 w-2 rounded-full bg-emerald-500 shadow-[0_0_8px_#10B981]" />
              </span>
              <span className="text-[10px] font-semibold text-emerald-400">Activo</span>
            </>
          ) : (
            <>
              <span className="relative flex h-2 w-2">
                <span className="relative inline-flex h-2 w-2 rounded-full bg-rose-500 shadow-[0_0_8px_#F43F5E]" />
              </span>
              <span className="text-[10px] font-semibold text-rose-400">Inactivo</span>
            </>
          )}
        </div>
      </div>

      {/* Sección Central: Avatar & Datos del Socio */}
      <div className="relative flex items-center gap-2.5">
        {/* Avatar circular con borde dorado */}
        <div className="relative flex h-10 w-10 flex-shrink-0 items-center justify-center overflow-hidden rounded-full border-2 border-[#D4AF37] bg-gradient-to-br from-[#1A365D] to-[#0A1424] shadow-[0_0_10px_rgba(212,175,55,0.25)]">
          {data.avatarUrl ? (
            // eslint-disable-next-line @next/next/no-img-element
            <img src={data.avatarUrl} alt={data.name} className="h-full w-full object-cover" />
          ) : (
            <span className="text-sm font-bold tracking-wider text-[#F5D77F]">
              {data.avatarInitials}
            </span>
          )}
          {data.level === 0 && (
            <Crown className="absolute -top-2.5 h-3.5 w-3.5 fill-[#D4AF37] text-[#D4AF37]" />
          )}
        </div>

        {/* Nombre & ID */}
        <div className="min-w-0 flex-1">
          <h4 className="truncate text-xs font-bold tracking-tight text-white group-hover:text-[#FDE047] transition-colors">
            {data.name}
          </h4>
          <p className="font-mono text-[10px] text-slate-400">#{data.code}</p>
        </div>
      </div>

      {/* Badge de Rango Metálico */}
      <div className="relative mt-2.5 flex items-center justify-between">
        <div
          className={`inline-flex items-center gap-1 rounded-full border px-2 py-0.5 text-[10px] font-medium ${rankMeta.badgeBg} ${rankMeta.badgeText} ${rankMeta.badgeBorder}`}
        >
          <RankIcon className="h-2.5 w-2.5" />
          <span className="truncate max-w-[120px]">{data.rank}</span>
        </div>
        <span className="text-[9px] text-slate-400 font-medium truncate max-w-[65px]">
          {data.country}
        </span>
      </div>

      {/* Separador Elegante */}
      <div className="my-2.5 h-[1px] w-full bg-gradient-to-r from-transparent via-[#D4AF37]/30 to-transparent" />

      {/* Métricas Específicas del Árbol */}
      {isBinary && binaryData && (
        <div className="space-y-1.5">
          <div className="grid grid-cols-2 gap-1.5 text-center">
            {/* Puntos Izquierda */}
            <div className="rounded-lg border border-slate-800/80 bg-[#050B14]/70 p-1.5">
              <span className="block text-[9px] font-medium uppercase tracking-wider text-slate-400">
                Pierna Izq
              </span>
              <span className="font-mono text-[11px] font-bold text-white">
                {binaryData.puntos_izquierda.toLocaleString('es-ES')}
              </span>
            </div>

            {/* Puntos Derecha */}
            <div className="rounded-lg border border-slate-800/80 bg-[#050B14]/70 p-1.5">
              <span className="block text-[9px] font-medium uppercase tracking-wider text-slate-400">
                Pierna Der
              </span>
              <span className="font-mono text-[11px] font-bold text-white">
                {binaryData.puntos_derecha.toLocaleString('es-ES')}
              </span>
            </div>
          </div>

          {/* Mini Barra de Balance Binario */}
          <div className="h-1.5 w-full overflow-hidden rounded-full bg-slate-800/80">
            <div
              className="h-full bg-gradient-to-r from-[#D4AF37] to-[#F5D77F] transition-all"
              style={{
                width: `${Math.round(
                  (binaryData.puntos_izquierda /
                    (binaryData.puntos_izquierda + binaryData.puntos_derecha || 1)) *
                    100
                )}%`,
              }}
            />
          </div>
        </div>
      )}

      {!isBinary && unilevelData && (
        <div className="rounded-lg border border-[#D4AF37]/20 bg-[#050B14]/70 p-2">
          <div className="flex items-center justify-between">
            <span className="text-[9px] font-semibold uppercase tracking-wider text-slate-400">
              Volumen Grupal (VGP)
            </span>
            <Coins className="h-3 w-3 text-[#D4AF37]" />
          </div>
          <div className="mt-0.5 flex items-baseline justify-between">
            <span className="font-mono text-xs font-extrabold text-[#F5D77F]">
              {unilevelData.volumen_grupal_puntos.toLocaleString('es-ES')}{' '}
              <span className="text-[9px] font-normal text-slate-400">pts</span>
            </span>
            <span className="text-[9px] text-slate-400 font-mono">
              VP: {unilevelData.volumen_personal_puntos.toLocaleString('es-ES')}
            </span>
          </div>
        </div>
      )}

      {/* Flecha indicadora flotante al pie */}
      <div className="mt-1 flex items-center justify-center text-[9px] font-medium text-[#D4AF37]/70 group-hover:text-[#D4AF37] transition-colors">
        <span>Click para detalles</span>
        <ChevronRight className="h-2.5 w-2.5 ml-0.5 transform group-hover:translate-x-0.5 transition-transform" />
      </div>
    </div>
  );
}

const nodeTypes = {
  mlmNode: MLMNodeCard,
};

/* ==========================================================================
   PANEL LATERAL DE DETALLES FLOTANTE (SIDEBAR Z-INDEX ALTO)
   Carga dinámicamente:
   - Inicial dentro de círculo con borde dorado
   - Nombre, rango (badge dorado) y ID único
   - Indicador visual de estado de cuenta (luz verde / roja)
   - Métricas específicas (binario: puntos izq/der; unilevel: volumen grupal)
   - Botón "✕" de cierre
   ========================================================================== */

interface DetailSidebarProps {
  member: MemberNodeData | null;
  onClose: () => void;
  onFocusNode: (id: string) => void;
  onToggleStatus?: (id: string) => void;
  onAddSimulatedPoints?: (id: string, leg?: 'izquierda' | 'derecha') => void;
  /** Hay referidos pendientes y el visor sabe colocarlos. */
  canPlace?: boolean;
  /** Piernas sin ocupar en este nodo. */
  freeLegs?: { izquierda: boolean; derecha: boolean };
  onPlace?: (parentId: string, leg: 'izquierda' | 'derecha') => void;
}

function DetailSidebar({
  member,
  onClose,
  onFocusNode,
  onToggleStatus,
  onAddSimulatedPoints,
  canPlace,
  freeLegs,
  onPlace,
}: DetailSidebarProps) {
  const [copied, setCopied] = useState(false);

  if (!member) return null;

  const rankMeta = RANK_METADATA[member.rank] || RANK_METADATA['Sin rango'];
  const RankIcon = rankMeta.icon;
  const isBinary = member.treeType === 'binary';
  const binaryData = isBinary ? (member as BinaryMemberData) : null;
  const unilevelData = !isBinary ? (member as UnilevelMemberData) : null;

  const handleCopyId = () => {
    navigator.clipboard.writeText(member.id);
    setCopied(true);
    setTimeout(() => setCopied(false), 2000);
  };

  // Cálculos de negocio MLM
  const binaryTotalPoints = binaryData
    ? binaryData.puntos_izquierda + binaryData.puntos_derecha
    : 0;
  const leftLegPct =
    binaryTotalPoints > 0
      ? Math.round((binaryData!.puntos_izquierda / binaryTotalPoints) * 100)
      : 50;
  const rightLegPct = 100 - leftLegPct;

  const weakLegPoints = binaryData
    ? Math.min(binaryData.puntos_izquierda, binaryData.puntos_derecha)
    : 0;
  const weakLegName =
    (binaryData?.puntos_izquierda ?? 0) <= (binaryData?.puntos_derecha ?? 0)
      ? 'Pierna Izquierda'
      : 'Pierna Derecha';
  const estimatedBinaryBonus = Math.round(weakLegPoints * 0.1); // Bono binario del 10%
  const estimatedUnilevelBonus = unilevelData
    ? Math.round(unilevelData.volumen_grupal_puntos * 0.05)
    : 0;

  return (
    <aside
      aria-label="Panel de Detalles del Socio"
      className="fixed bottom-6 right-6 top-20 z-50 flex w-[380px] max-w-[calc(100vw-3rem)] flex-col overflow-hidden rounded-3xl border border-[#D4AF37]/40 bg-[#0A1424]/95 backdrop-blur-2xl shadow-[0_20px_50px_rgba(0,0,0,0.85),0_0_35px_rgba(212,175,55,0.18)] animate-in fade-in slide-in-from-right-8 duration-300"
    >
      {/* Resplandor decorativo superior en oro */}
      <div className="pointer-events-none absolute inset-x-0 top-0 h-28 bg-gradient-to-b from-[#D4AF37]/15 via-transparent to-transparent" />

      {/* Encabezado del Panel */}
      <div className="relative flex items-center justify-between border-b border-slate-800/80 px-6 py-4">
        <div className="flex items-center gap-2">
          <div className="flex h-7 w-7 items-center justify-center rounded-lg border border-[#D4AF37]/40 bg-[#0F223F]">
            <Sparkles className="h-4 w-4 text-[#D4AF37]" />
          </div>
          <div>
            <h3 className="text-xs font-extrabold uppercase tracking-widest text-[#F5D77F]">
              Expediente del Distribuidor
            </h3>
            <p className="text-[10px] text-slate-400">
              {isBinary ? 'Red Binaria Global' : 'Red Unilevel Directa'}
            </p>
          </div>
        </div>

        {/* Botón "✕" para cerrar el panel */}
        <button
          onClick={onClose}
          aria-label="Cerrar detalles"
          className="group flex h-8 w-8 items-center justify-center rounded-xl border border-slate-700/80 bg-[#050B14]/80 text-slate-400 transition-all hover:border-[#D4AF37] hover:bg-[#0F223F] hover:text-[#F5D77F]"
        >
          <X className="h-4 w-4 transition-transform group-hover:rotate-90" />
        </button>
      </div>

      {/* Contenido con scroll elegante */}
      <div className="relative flex-1 overflow-y-auto px-6 py-5 space-y-5">
        {/* Tarjeta de Perfil Principal */}
        <div className="relative overflow-hidden rounded-2xl border border-[#D4AF37]/30 bg-gradient-to-b from-[#0F223F] to-[#0A1424] p-4 shadow-lg text-center">
          {/* Avatar circular con borde dorado prominente */}
          <div className="relative mx-auto flex h-20 w-20 items-center justify-center overflow-hidden rounded-full border-4 border-[#D4AF37] bg-gradient-to-tr from-[#050B14] via-[#1A365D] to-[#0A1424] shadow-[0_0_20px_rgba(212,175,55,0.4)]">
            {member.avatarUrl ? (
              // eslint-disable-next-line @next/next/no-img-element
              <img src={member.avatarUrl} alt={member.name} className="h-full w-full object-cover" />
            ) : (
              <span className="text-2xl font-black tracking-wider text-[#F5D77F]">
                {member.avatarInitials}
              </span>
            )}
            {member.level === 0 && (
              <div className="absolute -top-3.5 flex h-7 w-7 items-center justify-center rounded-full border-2 border-[#D4AF37] bg-[#050B14] shadow-md">
                <Crown className="h-4 w-4 fill-[#D4AF37] text-[#D4AF37]" />
              </div>
            )}
          </div>

          {/* Nombre Completo */}
          <h2 className="mt-3 text-lg font-extrabold tracking-tight text-white">
            {member.name}
          </h2>

          {/* ID con Botón de Copiar */}
          <div className="mt-1 flex items-center justify-center gap-2">
            <span className="font-mono text-xs font-semibold text-slate-400">
              ID: <span className="text-slate-200">#{member.code}</span>
            </span>
            <button
              onClick={handleCopyId}
              title="Copiar ID"
              className="inline-flex items-center gap-1 rounded-md border border-slate-700/80 bg-[#050B14] px-1.5 py-0.5 text-[10px] text-slate-300 transition-colors hover:border-[#D4AF37] hover:text-[#F5D77F]"
            >
              {copied ? (
                <>
                  <Check className="h-2.5 w-2.5 text-emerald-400" />
                  <span className="text-emerald-400 font-semibold">Copiado</span>
                </>
              ) : (
                <>
                  <Copy className="h-2.5 w-2.5" />
                  <span>Copiar</span>
                </>
              )}
            </button>
          </div>

          {/* Rango (insignia real del plan + nombre) */}
          <div className="mt-3 inline-flex items-center gap-2 rounded-full border border-[#D4AF37] bg-gradient-to-r from-[#D4AF37]/20 via-[#F5D77F]/25 to-[#AA820A]/20 py-1 pl-1.5 pr-3.5 shadow-sm">
            {rankLevelByName(member.rank) > 0 ? (
              <RankBadge level={rankLevelByName(member.rank)} size="xs" />
            ) : (
              <RankIcon className="h-3.5 w-3.5 text-[#FDE047]" />
            )}
            <span className="text-xs font-bold tracking-wide text-[#FDE047]">
              {member.rank}
            </span>
          </div>

          {/* Indicador Visual de Estado de Cuenta & Botón de Simulación */}
          <div className="mt-4 flex flex-col items-center gap-2">
            <div className="flex items-center justify-center">
              {member.status === 'Activo' ? (
                <div className="inline-flex items-center gap-2 rounded-full border border-emerald-500/40 bg-emerald-500/10 px-3 py-1 shadow-[0_0_15px_rgba(16,185,129,0.2)]">
                  <span className="relative flex h-2.5 w-2.5">
                    <span className="absolute inline-flex h-full w-full animate-ping rounded-full bg-emerald-400 opacity-75" />
                    <span className="relative inline-flex h-2.5 w-2.5 rounded-full bg-emerald-500 shadow-[0_0_10px_#10B981]" />
                  </span>
                  <span className="text-xs font-semibold text-emerald-400">
                    Cuenta Verificada • Activo
                  </span>
                </div>
              ) : (
                <div className="inline-flex items-center gap-2 rounded-full border border-rose-500/40 bg-rose-500/10 px-3 py-1 shadow-[0_0_15px_rgba(244,63,94,0.2)]">
                  <span className="relative flex h-2.5 w-2.5">
                    <span className="relative inline-flex h-2.5 w-2.5 rounded-full bg-rose-500 shadow-[0_0_10px_#F43F5E]" />
                  </span>
                  <span className="text-xs font-semibold text-rose-400">
                    Mantenimiento Inactivo
                  </span>
                </div>
              )}
            </div>

            {/* Micro-botón para simular cambio de estado en vivo */}
            {onToggleStatus && (
              <button
                onClick={() => onToggleStatus(member.id)}
                className="text-[10px] text-slate-400 underline hover:text-[#F5D77F] transition-colors"
              >
                Alternar a {member.status === 'Activo' ? 'Inactivo' : 'Activo'} (Demo)
              </button>
            )}
          </div>
        </div>

        {/* MÉTRICAS ESPECÍFICAS SEGÚN EL ÁRBOL ACTIVO */}
        {isBinary && binaryData && (
          <div className="space-y-3">
            <div className="flex items-center justify-between">
              <span className="text-[11px] font-bold uppercase tracking-wider text-[#F5D77F] flex items-center gap-1.5">
                <GitBranch className="h-3.5 w-3.5 text-[#D4AF37]" />
                Métricas de Red Binaria
              </span>
              <span className="text-[10px] text-slate-400 font-mono">
                Corte Semanal
              </span>
            </div>

            {/* Piernas Izquierda y Derecha */}
            <div className="grid grid-cols-2 gap-2.5">
              {/* Pierna Izquierda */}
              <div className="rounded-2xl border border-slate-700/70 bg-[#050B14]/80 p-3 shadow-inner">
                <div className="flex items-center justify-between">
                  <span className="text-[10px] font-bold uppercase tracking-wider text-slate-400">
                    Pierna Izquierda
                  </span>
                  <div className="h-2 w-2 rounded-full bg-sky-400 shadow-[0_0_6px_#38BDF8]" />
                </div>
                <div className="mt-1 font-mono text-base font-extrabold text-white">
                  {binaryData.puntos_izquierda.toLocaleString('es-ES')}
                </div>
                <div className="mt-0.5 text-[10px] text-slate-400">
                  {leftLegPct}% del volumen total
                </div>
              </div>

              {/* Pierna Derecha */}
              <div className="rounded-2xl border border-slate-700/70 bg-[#050B14]/80 p-3 shadow-inner">
                <div className="flex items-center justify-between">
                  <span className="text-[10px] font-bold uppercase tracking-wider text-slate-400">
                    Pierna Derecha
                  </span>
                  <div className="h-2 w-2 rounded-full bg-[#D4AF37] shadow-[0_0_6px_#D4AF37]" />
                </div>
                <div className="mt-1 font-mono text-base font-extrabold text-[#F5D77F]">
                  {binaryData.puntos_derecha.toLocaleString('es-ES')}
                </div>
                <div className="mt-0.5 text-[10px] text-slate-400">
                  {rightLegPct}% del volumen total
                </div>
              </div>
            </div>

            {/* Indicador de Pierna de Pago (Pierna Menor) */}
            <div className="rounded-2xl border border-[#D4AF37]/35 bg-gradient-to-r from-[#0F223F] to-[#0A1424] p-3.5">
              <div className="flex items-center justify-between">
                <span className="text-[11px] font-semibold text-slate-300 flex items-center gap-1.5">
                  <ArrowRightLeft className="h-3.5 w-3.5 text-[#D4AF37]" />
                  Pierna de Pago Calificada:
                </span>
                <span className="rounded-full bg-[#D4AF37]/20 border border-[#D4AF37]/50 px-2 py-0.5 text-[10px] font-bold text-[#FDE047]">
                  {weakLegName}
                </span>
              </div>
              <div className="mt-2.5 flex items-baseline justify-between border-t border-slate-800 pt-2">
                <span className="text-[10px] text-slate-400">
                  Bono Binario Estimado (10%):
                </span>
                <span className="font-mono text-sm font-black text-emerald-400">
                  ${estimatedBinaryBonus.toLocaleString('es-ES')} USD
                </span>
              </div>

              {onAddSimulatedPoints && (
                <div className="mt-2.5 pt-2 border-t border-slate-800/80 flex justify-end">
                  <button
                    onClick={() =>
                      onAddSimulatedPoints(
                        member.id,
                        weakLegName === 'Pierna Izquierda' ? 'izquierda' : 'derecha'
                      )
                    }
                    className="inline-flex items-center gap-1 text-[10px] font-semibold text-[#F5D77F] hover:text-white transition-colors"
                  >
                    <Zap className="h-3 w-3 text-[#D4AF37]" />
                    Simular +10,000 pts en pierna menor
                  </button>
                </div>
              )}
            </div>

            {/* Ciclos Binarios Completados */}
            <div className="flex items-center justify-between rounded-xl border border-slate-800 bg-[#050B14]/60 px-3.5 py-2.5">
              <span className="text-xs text-slate-300 flex items-center gap-2">
                <Zap className="h-3.5 w-3.5 text-[#D4AF37]" />
                Ciclos Binarios Acumulados
              </span>
              <span className="font-mono text-xs font-bold text-white">
                {binaryData.ciclos_binarios} Ciclos
              </span>
            </div>
          </div>
        )}

        {!isBinary && unilevelData && (
          <div className="space-y-3">
            <div className="flex items-center justify-between">
              <span className="text-[11px] font-bold uppercase tracking-wider text-[#F5D77F] flex items-center gap-1.5">
                <Network className="h-3.5 w-3.5 text-[#D4AF37]" />
                Métricas de Red Unilevel
              </span>
              <span className="text-[10px] text-slate-400 font-mono">
                Periodo Mensual
              </span>
            </div>

            {/* Tarjeta de Volumen Grupal (VGP) Destacada */}
            <div className="rounded-2xl border border-[#D4AF37]/45 bg-gradient-to-br from-[#0F223F] via-[#10274A] to-[#0A1424] p-4 shadow-lg">
              <div className="flex items-center justify-between">
                <span className="text-[10px] font-bold uppercase tracking-wider text-slate-300">
                  Volumen Grupal de Puntos (VGP)
                </span>
                <Coins className="h-4 w-4 text-[#D4AF37]" />
              </div>
              <div className="mt-1 font-mono text-2xl font-black text-[#F5D77F] tracking-tight">
                {unilevelData.volumen_grupal_puntos.toLocaleString('es-ES')}{' '}
                <span className="text-xs font-semibold text-slate-300">PTS</span>
              </div>
              <p className="mt-1 text-[10px] text-slate-400">
                Suma consolidada de toda la genealogía descendente hasta 3 niveles.
              </p>

              {onAddSimulatedPoints && (
                <div className="mt-2.5 pt-2 border-t border-slate-800/80 flex justify-end">
                  <button
                    onClick={() => onAddSimulatedPoints(member.id)}
                    className="inline-flex items-center gap-1 text-[10px] font-semibold text-[#F5D77F] hover:text-white transition-colors"
                  >
                    <Zap className="h-3 w-3 text-[#D4AF37]" />
                    Simular +25,000 pts VGP (Demo)
                  </button>
                </div>
              )}
            </div>

            {/* Volumen Personal & Líneas Directas */}
            <div className="grid grid-cols-2 gap-2.5">
              <div className="rounded-2xl border border-slate-700/70 bg-[#050B14]/80 p-3">
                <span className="block text-[10px] font-bold uppercase tracking-wider text-slate-400">
                  Volumen Personal (VP)
                </span>
                <span className="mt-1 block font-mono text-sm font-extrabold text-white">
                  {unilevelData.volumen_personal_puntos.toLocaleString('es-ES')} pts
                </span>
                <span className="text-[9px] text-emerald-400 font-semibold">
                  ✓ Recompra Calificada
                </span>
              </div>

              <div className="rounded-2xl border border-slate-700/70 bg-[#050B14]/80 p-3">
                <span className="block text-[10px] font-bold uppercase tracking-wider text-slate-400">
                  Líneas Directas
                </span>
                <span className="mt-1 block font-mono text-sm font-extrabold text-[#F5D77F]">
                  {unilevelData.lineas_directas} Frontales
                </span>
                <span className="text-[9px] text-slate-400 font-medium">
                  {member.level === 0 ? 'Líneas Activas' : 'Equipo Directo'}
                </span>
              </div>
            </div>

            {/* Comisión Unilevel Proyectada */}
            <div className="rounded-2xl border border-[#D4AF37]/30 bg-[#050B14]/70 p-3.5">
              <div className="flex items-center justify-between">
                <span className="text-xs text-slate-300 flex items-center gap-1.5">
                  <DollarSign className="h-3.5 w-3.5 text-[#D4AF37]" />
                  Comisión Unilevel Proyectada (5%):
                </span>
                <span className="font-mono text-sm font-black text-emerald-400">
                  ${estimatedUnilevelBonus.toLocaleString('es-ES')} USD
                </span>
              </div>
            </div>
          </div>
        )}

        {/* Metadatos Corporativos Adicionales */}
        <div className="space-y-2 rounded-2xl border border-slate-800 bg-[#050B14]/70 p-3.5 text-xs">
          <span className="block text-[10px] font-bold uppercase tracking-wider text-[#F5D77F] mb-1">
            Información Institucional
          </span>

          <div className="flex items-center justify-between py-1 border-b border-slate-800/60">
            <span className="text-slate-400 flex items-center gap-1.5 text-[11px]">
              <Users className="h-3 w-3 text-slate-500" /> Patrocinador (Upline):
            </span>
            <span className="font-medium text-white truncate max-w-[160px]">
              {member.uplineName}
            </span>
          </div>

          <div className="flex items-center justify-between py-1 border-b border-slate-800/60">
            <span className="text-slate-400 flex items-center gap-1.5 text-[11px]">
              <Globe className="h-3 w-3 text-slate-500" /> País / Región:
            </span>
            <span className="font-medium text-white">{member.country}</span>
          </div>

          <div className="flex items-center justify-between py-1 border-b border-slate-800/60">
            <span className="text-slate-400 flex items-center gap-1.5 text-[11px]">
              <Calendar className="h-3 w-3 text-slate-500" /> Fecha de Ingreso:
            </span>
            <span className="font-mono text-white text-[11px]">
              {member.joinDate}
            </span>
          </div>

          <div className="flex items-center justify-between py-1">
            <span className="text-slate-400 flex items-center gap-1.5 text-[11px]">
              <Mail className="h-3 w-3 text-slate-500" /> Correo:
            </span>
            <span className="font-mono text-[10px] text-slate-300 truncate max-w-[160px]">
              {member.email}
            </span>
          </div>
        </div>
      </div>

      {/* Colocar un referido pendiente bajo este nodo */}
      {canPlace && onPlace && isBinary && (freeLegs?.izquierda || freeLegs?.derecha) && (
        <div className="border-t border-slate-800/80 bg-[#050B14]/90 px-4 pt-3">
          <p className="mb-2 text-[10px] font-semibold uppercase tracking-wider text-slate-400">
            Colocar referido pendiente
          </p>
          <div className="flex gap-2">
            {(['izquierda', 'derecha'] as const).filter((leg) => freeLegs?.[leg]).map((leg) => (
              <button
                key={leg}
                onClick={() => onPlace(member.id, leg)}
                className="flex flex-1 items-center justify-center gap-1.5 rounded-xl border border-emerald-500/50 bg-emerald-500/10 px-3 py-2 text-[11px] font-semibold text-emerald-300 transition-colors hover:bg-emerald-500/20"
              >
                <Plus className="h-3.5 w-3.5" />
                Pierna {leg}
              </button>
            ))}
          </div>
        </div>
      )}

      {/* Pie del Panel con Botones de Acción */}
      <div className="border-t border-slate-800/80 bg-[#050B14]/90 p-4 flex gap-2">
        <button
          onClick={() => onFocusNode(member.id)}
          className="flex-1 flex items-center justify-center gap-1.5 rounded-xl border border-[#D4AF37]/50 bg-gradient-to-r from-[#D4AF37] via-[#F5D77F] to-[#AA820A] px-3 py-2 text-xs font-bold text-[#050B14] shadow-[0_0_15px_rgba(212,175,55,0.3)] transition-all hover:brightness-110 active:scale-95"
        >
          <Maximize2 className="h-3.5 w-3.5" />
          Centrar en Lienzo
        </button>

        <button
          onClick={onClose}
          className="flex items-center justify-center rounded-xl border border-slate-700 bg-[#0F223F] px-4 py-2 text-xs font-semibold text-slate-200 transition-colors hover:border-[#D4AF37] hover:text-[#F5D77F]"
        >
          Cerrar
        </button>
      </div>
    </aside>
  );
}

/* ==========================================================================
   COMPONENTE INTERNO DEL VISUALIZADOR (DENTRO DEL PROVIDER DE REACTFLOW)
   ========================================================================== */

export interface PendingMember {
  id: string;
  code: string;
  name: string;
}

interface MlmStructureProps {
  binary: BinaryMemberData[];
  unilevel: UnilevelMemberData[];
  /** Id del usuario dueño del panel: es el nodo seleccionado al abrir. */
  rootId: string;
  /** Referidos directos aún sin colocar en el binario. */
  pending?: PendingMember[];
  /** Coloca un pendiente bajo `parentId` en la pierna indicada. */
  onPlaceMember?: (memberId: string, parentId: string, leg: 'left' | 'right') => Promise<string | null>;
}

function MLMCanvasViewer({ binary, unilevel, rootId, pending = [], onPlaceMember }: MlmStructureProps) {
  const [activeTab, setActiveTab] = useState<TreeStructureType>('binary');
  const [binaryMembers, setBinaryMembers] = useState<BinaryMemberData[]>(binary);
  const [unilevelMembers, setUnilevelMembers] = useState<UnilevelMemberData[]>(unilevel);

  // Los datos llegan del servidor: si se recargan, refrescar el lienzo.
  useEffect(() => setBinaryMembers(binary), [binary]);
  useEffect(() => setUnilevelMembers(unilevel), [unilevel]);
  const [selectedMemberId, setSelectedMemberId] = useState<string>(rootId);
  const [searchQuery, setSearchQuery] = useState('');
  const [showSearchDropdown, setShowSearchDropdown] = useState(false);
  const [showLegend, setShowLegend] = useState(true);
  const [isSidebarOpen, setIsSidebarOpen] = useState(true);
  const [collapsedIds, setCollapsedIds] = useState<Set<string>>(new Set());
  const [placeTarget, setPlaceTarget] = useState<{ parentId: string; leg: 'izquierda' | 'derecha' } | null>(null);
  const [placeError, setPlaceError] = useState<string | null>(null);
  const [placing, setPlacing] = useState(false);

  const { fitView, setCenter, getNode } = useReactFlow();

  // Obtener miembro seleccionado actual desde el estado dinámico
  const selectedMember = useMemo<MemberNodeData | null>(() => {
    if (activeTab === 'binary') {
      return binaryMembers.find((m) => m.id === selectedMemberId) || binaryMembers[0] || null;
    } else {
      return unilevelMembers.find((m) => m.id === selectedMemberId) || unilevelMembers[0] || null;
    }
  }, [activeTab, selectedMemberId, binaryMembers, unilevelMembers]);

  /** Piernas libres del nodo abierto en el expediente. */
  const selectedFreeLegs = useMemo(() => {
    const taken = new Set(
      binaryMembers.filter((m) => m.parentId === selectedMemberId && m.leg).map((m) => m.leg as string)
    );
    return { izquierda: !taken.has('izquierda'), derecha: !taken.has('derecha') };
  }, [binaryMembers, selectedMemberId]);

  // Callback para seleccionar nodo
  const handleSelectNode = useCallback((data: MemberNodeData) => {
    setSelectedMemberId(data.id);
    setIsSidebarOpen(true);
  }, []);

  /** Pliega/despliega la rama que cuelga de un nodo. */
  const handleToggleCollapse = useCallback((id: string) => {
    setCollapsedIds((prev) => {
      const next = new Set(prev);
      if (next.has(id)) next.delete(id);
      else next.add(id);
      return next;
    });
  }, []);

  /** Abre el selector de referido pendiente para esa pierna. */
  const handlePlace = useCallback((parentId: string, leg: 'izquierda' | 'derecha') => {
    setPlaceError(null);
    setPlaceTarget({ parentId, leg });
  }, []);

  async function confirmPlace(memberId: string) {
    if (!placeTarget || !onPlaceMember) return;
    setPlacing(true);
    setPlaceError(null);
    const err = await onPlaceMember(
      memberId,
      placeTarget.parentId,
      placeTarget.leg === 'izquierda' ? 'left' : 'right'
    );
    setPlacing(false);
    if (err) setPlaceError(err);
    else setPlaceTarget(null);
  }


  // Construir elementos según pestaña activa y datos dinámicos.
  // Los descendientes de un nodo plegado se excluyen del lienzo.
  const { initialNodes, initialEdges } = useMemo(() => {
    const visible = <T extends { id: string; parentId: string | null }>(list: T[]): T[] => {
      if (collapsedIds.size === 0) return list;
      const byId = new Map(list.map((m) => [m.id, m]));
      const hidden = (m: T): boolean => {
        let p = m.parentId;
        while (p) {
          if (collapsedIds.has(p)) return true;
          p = byId.get(p)?.parentId ?? null;
        }
        return false;
      };
      return list.filter((m) => !hidden(m));
    };

    /** Datos extra de interacción que recibe cada tarjeta. */
    const decorate = <T extends { id: string }>(list: T[], counts: Map<string, number>) =>
      list.map((m) => ({
        ...m,
        childCount: counts.get(m.id) ?? 0,
        isCollapsed: collapsedIds.has(m.id),
        onToggleCollapse: handleToggleCollapse,
      }));

    if (activeTab === 'binary') {
      const counts = new Map<string, number>();
      for (const m of binaryMembers) {
        if (m.parentId) counts.set(m.parentId, (counts.get(m.parentId) ?? 0) + 1);
      }
      // Piernas ocupadas, para saber dónde ofrecer el "+".
      const taken = new Map<string, Set<string>>();
      for (const m of binaryMembers) {
        if (!m.parentId || !m.leg) continue;
        const set = taken.get(m.parentId) ?? new Set<string>();
        set.add(m.leg);
        taken.set(m.parentId, set);
      }

      const decorated = decorate(visible(binaryMembers), counts).map((m) => ({
        ...m,
        freeLegs: pending.length > 0 && onPlaceMember
          ? {
              izquierda: !(taken.get(m.id)?.has('izquierda') ?? false),
              derecha: !(taken.get(m.id)?.has('derecha') ?? false),
            }
          : undefined,
        onPlace: handlePlace,
      })) as BinaryMemberData[];

      const { nodes, edges } = buildBinaryTreeElements(decorated, selectedMemberId, handleSelectNode);
      return { initialNodes: nodes, initialEdges: edges };
    }

    const counts = new Map<string, number>();
    for (const m of unilevelMembers) {
      if (m.parentId) counts.set(m.parentId, (counts.get(m.parentId) ?? 0) + 1);
    }
    const decorated = decorate(visible(unilevelMembers), counts) as UnilevelMemberData[];
    const { nodes, edges } = buildUnilevelTreeElements(decorated, selectedMemberId, handleSelectNode);
    return { initialNodes: nodes, initialEdges: edges };
  }, [
    activeTab,
    binaryMembers,
    unilevelMembers,
    selectedMemberId,
    handleSelectNode,
    collapsedIds,
    handleToggleCollapse,
    handlePlace,
    pending.length,
    onPlaceMember,
  ]);

  const [nodes, setNodes, onNodesChange] = useNodesState(initialNodes);
  const [edges, setEdges, onEdgesChange] = useEdgesState(initialEdges);

  // Sincronizar nodos y aristas cuando cambia la estructura o la selección
  useEffect(() => {
    setNodes(initialNodes);
    setEdges(initialEdges);
  }, [initialNodes, initialEdges, setNodes, setEdges]);

  // Reajustar zoom (fitView) automáticamente solo al alternar entre pestañas o al inicio
  useEffect(() => {
    const timer = setTimeout(() => {
      fitView({ padding: 0.25, duration: 600 });
    }, 80);

    return () => clearTimeout(timer);
  }, [activeTab, fitView]);

  // Manejar cambio de pestaña
  const handleTabChange = (newTab: TreeStructureType) => {
    if (newTab === activeTab) return;
    setActiveTab(newTab);
    // Seleccionar automáticamente la raíz del nuevo árbol
    if (newTab === 'binary') {
      setSelectedMemberId('BIN-001');
    } else {
      setSelectedMemberId('UNI-001');
    }
    setIsSidebarOpen(true);
    setSearchQuery('');
    setShowSearchDropdown(false);
  };

  // Centrar y enfocar un nodo específico
  const focusOnNode = useCallback(
    (nodeId: string) => {
      const targetNode = getNode(nodeId);
      if (targetNode) {
        setCenter(
          targetNode.position.x + 120,
          targetNode.position.y + 75,
          { zoom: 1.15, duration: 600 }
        );
      }
    },
    [getNode, setCenter]
  );

  // Lista activa de miembros en base al estado mutable
  const currentMembersList =
    activeTab === 'binary' ? binaryMembers : unilevelMembers;

  const filteredMembers = useMemo(() => {
    if (!searchQuery.trim()) return [];
    const query = searchQuery.toLowerCase();
    return currentMembersList.filter(
      (m) =>
        m.name.toLowerCase().includes(query) ||
        m.code.toLowerCase().includes(query) ||
        m.rank.toLowerCase().includes(query)
    );
  }, [searchQuery, currentMembersList]);

  const handleSelectSearchResult = (member: BaseMemberData) => {
    setSelectedMemberId(member.id);
    setIsSidebarOpen(true);
    focusOnNode(member.id);
    setSearchQuery('');
    setShowSearchDropdown(false);
  };

  return (
    <div className="relative h-[calc(100vh-12rem)] min-h-[520px] w-full overflow-hidden rounded-card border border-border bg-[#050B14]">
      {/* ====================================================================
          BARRA DE ENCABEZADO SUPERIOR (LUJO CORPORATIVO & SELECTOR DE PESTAÑAS)
          ==================================================================== */}
      <header className="absolute left-0 right-0 top-0 z-30 flex h-16 flex-wrap items-center justify-between gap-2 border-b border-[#D4AF37]/30 bg-[#0A1424]/90 px-4 backdrop-blur-xl">

        {/* SELECTOR DE PESTAÑAS (TABS) ELEGANTE: BINARIO VS UNILEVEL */}
        <div className="flex items-center gap-1.5 rounded-2xl border border-[#D4AF37]/35 bg-[#050B14]/80 p-1.5 shadow-inner">
          {/* Pestaña: Estructura Binaria */}
          <button
            onClick={() => handleTabChange('binary')}
            className={`group relative flex items-center gap-2.5 rounded-xl px-5 py-2.5 text-xs font-bold transition-all duration-300 ${
              activeTab === 'binary'
                ? 'bg-gradient-to-r from-[#D4AF37] via-[#F5D77F] to-[#AA820A] text-[#050B14] shadow-[0_0_20px_rgba(212,175,55,0.45)]'
                : 'text-slate-300 hover:text-white hover:bg-[#0F223F]/80'
            }`}
          >
            <GitBranch
              className={`h-4 w-4 transition-transform group-hover:scale-110 ${
                activeTab === 'binary' ? 'text-[#050B14]' : 'text-[#D4AF37]'
              }`}
            />
            <span className="tracking-wide">Estructura Binaria</span>
            <span
              className={`rounded-full px-2 py-0.5 text-[10px] font-mono font-bold ${
                activeTab === 'binary'
                  ? 'bg-[#050B14] text-[#FDE047]'
                  : 'bg-[#0F223F] text-slate-300 border border-slate-700'
              }`}
            >
              {binaryMembers.length} Nodos
            </span>
          </button>

          {/* Pestaña: Estructura Unilevel */}
          <button
            onClick={() => handleTabChange('unilevel')}
            className={`group relative flex items-center gap-2.5 rounded-xl px-5 py-2.5 text-xs font-bold transition-all duration-300 ${
              activeTab === 'unilevel'
                ? 'bg-gradient-to-r from-[#D4AF37] via-[#F5D77F] to-[#AA820A] text-[#050B14] shadow-[0_0_20px_rgba(212,175,55,0.45)]'
                : 'text-slate-300 hover:text-white hover:bg-[#0F223F]/80'
            }`}
          >
            <Network
              className={`h-4 w-4 transition-transform group-hover:scale-110 ${
                activeTab === 'unilevel' ? 'text-[#050B14]' : 'text-[#D4AF37]'
              }`}
            />
            <span className="tracking-wide">Estructura Unilevel</span>
            <span
              className={`rounded-full px-2 py-0.5 text-[10px] font-mono font-bold ${
                activeTab === 'unilevel'
                  ? 'bg-[#050B14] text-[#FDE047]'
                  : 'bg-[#0F223F] text-slate-300 border border-slate-700'
              }`}
            >
              {unilevelMembers.length} Nodos
            </span>
          </button>
        </div>

        {/* Acciones de Navegación, Buscador & Centrar */}
        <div className="flex items-center gap-3">
          {/* Buscador de Socios */}
          <div className="relative">
            <div className="flex items-center rounded-xl border border-slate-700 bg-[#050B14] px-3 py-1.5 focus-within:border-[#D4AF37] transition-all">
              <Search className="h-3.5 w-3.5 text-slate-400" />
              <input
                type="text"
                value={searchQuery}
                onChange={(e) => {
                  setSearchQuery(e.target.value);
                  setShowSearchDropdown(true);
                }}
                onFocus={() => setShowSearchDropdown(true)}
                placeholder="Buscar por nombre o ID..."
                className="ml-2 w-48 bg-transparent text-xs text-white placeholder-slate-500 focus:outline-none"
              />
              {searchQuery && (
                <button
                  onClick={() => setSearchQuery('')}
                  className="text-slate-400 hover:text-white"
                >
                  <X className="h-3 w-3" />
                </button>
              )}
            </div>

            {/* Menú desplegable de resultados */}
            {showSearchDropdown && searchQuery && (
              <div className="absolute right-0 top-12 z-50 w-72 rounded-2xl border border-[#D4AF37]/40 bg-[#0A1424] p-2 shadow-2xl backdrop-blur-xl">
                {filteredMembers.length > 0 ? (
                  <div className="max-h-60 overflow-y-auto space-y-1">
                    {filteredMembers.map((m) => (
                      <button
                        key={m.id}
                        onClick={() => handleSelectSearchResult(m)}
                        className="flex w-full items-center justify-between rounded-xl p-2 text-left transition-colors hover:bg-[#0F223F]"
                      >
                        <div className="flex items-center gap-2">
                          <div className="flex h-7 w-7 shrink-0 items-center justify-center overflow-hidden rounded-full border border-[#D4AF37] bg-[#1A365D] text-[10px] font-bold text-[#F5D77F]">
                            {m.avatarUrl ? (
                              // eslint-disable-next-line @next/next/no-img-element
                              <img src={m.avatarUrl} alt={m.name} className="h-full w-full object-cover" />
                            ) : (
                              m.avatarInitials
                            )}
                          </div>
                          <div>
                            <p className="text-xs font-bold text-white">{m.name}</p>
                            <p className="font-mono text-[9px] text-slate-400">
                              #{m.code} • {m.rank}
                            </p>
                          </div>
                        </div>
                        <span className="text-[10px] text-[#D4AF37]">L{m.level}</span>
                      </button>
                    ))}
                  </div>
                ) : (
                  <p className="p-3 text-center text-xs text-slate-400">
                    No se encontraron distribuidores
                  </p>
                )}
              </div>
            )}
          </div>

          {/* Botón Reajustar Vista (Fit View) */}
          <button
            onClick={() => fitView({ padding: 0.25, duration: 600 })}
            title="Centrar y encuadrar árbol completo"
            className="flex items-center gap-1.5 rounded-xl border border-slate-700 bg-[#0F223F] px-3.5 py-2 text-xs font-semibold text-slate-200 transition-all hover:border-[#D4AF37] hover:text-[#F5D77F] active:scale-95"
          >
            <Maximize2 className="h-3.5 w-3.5 text-[#D4AF37]" />
            <span>Encuadrar</span>
          </button>
        </div>
      </header>


      {/* ====================================================================
          CANVAS INFINITO CON REACTFLOW
          - Fondo ultra oscuro #050B14
          - Cuadrícula con puntos dorados de opacidad baja
          - MiniMap y Controls estilizados
          - Conectores smoothstep dorados #D4AF37 de 2px
          - Arrastre fluido (Pan) y Zoom con rueda de ratón
          ==================================================================== */}
      <main className="h-full w-full pt-16">
        <ReactFlow
          nodes={nodes}
          edges={edges}
          onNodesChange={onNodesChange}
          onEdgesChange={onEdgesChange}
          nodeTypes={nodeTypes}
          onNodeClick={(_, node) => {
            if (node.data) {
              handleSelectNode(node.data as MemberNodeData);
            }
          }}
          fitView
          fitViewOptions={{ padding: 0.25 }}
          minZoom={0.3}
          maxZoom={1.8}
          panOnDrag={true}
          zoomOnScroll={true}
          zoomOnPinch={true}
          proOptions={{ hideAttribution: true }}
          defaultEdgeOptions={{
            type: 'smoothstep',
            style: { stroke: '#D4AF37', strokeWidth: 2 },
          }}
        >
          {/* Fondo con puntos dorados de opacidad baja */}
          <Background
            color="#D4AF37"
            gap={26}
            size={1.2}
            style={{ opacity: 0.16 }}
          />

          {/* MiniMap estilizado con paleta azul y dorada */}
          <MiniMap
            nodeColor={(node) => {
              const member = node.data as MemberNodeData;
              if (member?.status === 'Inactivo') return '#F43F5E';
              if (member?.level === 0) return '#FDE047';
              return '#D4AF37';
            }}
            nodeStrokeColor="#D4AF37"
            nodeBorderRadius={4}
            maskColor="rgba(5, 11, 20, 0.85)"
            position="bottom-left"
            className="!bottom-6 !left-6 !w-44 !h-28"
          />

          {/* Controles de Zoom estilizados */}
          <Controls
            showInteractive={false}
            position="bottom-left"
            className="!bottom-36 !left-6"
          />
        </ReactFlow>
      </main>

      {/* ====================================================================
          LEYENDA FLOTANTE DE LA RED (INFORMATIVA)
          ==================================================================== */}
      <div className="absolute bottom-6 left-56 z-20">
        {showLegend ? (
          <div className="rounded-2xl border border-slate-800 bg-[#0A1424]/90 p-3 backdrop-blur-md shadow-xl text-xs space-y-2">
            <div className="flex items-center justify-between gap-4">
              <span className="font-bold text-[#F5D77F] flex items-center gap-1.5 text-[11px]">
                <Shield className="h-3 w-3 text-[#D4AF37]" />
                Convenciones del Árbol
              </span>
              <button
                onClick={() => setShowLegend(false)}
                className="text-slate-400 hover:text-white"
              >
                <X className="h-3 w-3" />
              </button>
            </div>

            <div className="flex items-center gap-4 text-[10px]">
              <div className="flex items-center gap-1.5">
                <span className="h-2 w-2 rounded-full bg-emerald-500 shadow-[0_0_6px_#10B981]" />
                <span className="text-slate-300">Activo</span>
              </div>
              <div className="flex items-center gap-1.5">
                <span className="h-2 w-2 rounded-full bg-rose-500 shadow-[0_0_6px_#F43F5E]" />
                <span className="text-slate-300">Inactivo</span>
              </div>
              <div className="flex items-center gap-1.5">
                <span className="h-0.5 w-4 bg-[#D4AF37]" />
                <span className="text-slate-300">Smoothstep (Oro)</span>
              </div>
            </div>
          </div>
        ) : (
          <button
            onClick={() => setShowLegend(true)}
            className="flex items-center gap-1.5 rounded-xl border border-slate-800 bg-[#0A1424]/90 px-3 py-1.5 text-xs text-slate-300 hover:border-[#D4AF37] hover:text-[#F5D77F] backdrop-blur-md"
          >
            <Shield className="h-3 w-3 text-[#D4AF37]" />
            <span>Ver Leyenda</span>
          </button>
        )}
      </div>

      {/* ====================================================================
          PANEL LATERAL DE DETALLES FLOTANTE (SIDEBAR Z-INDEX 50)
          ==================================================================== */}
      {isSidebarOpen && selectedMember && (
        <DetailSidebar
          member={selectedMember}
          onClose={() => setIsSidebarOpen(false)}
          onFocusNode={focusOnNode}
          canPlace={pending.length > 0 && !!onPlaceMember}
          freeLegs={selectedFreeLegs}
          onPlace={handlePlace}
        />
      )}

      {/* Botón flotante para reabrir expediente si el panel fue cerrado */}
      {!isSidebarOpen && selectedMember && (
        <button
          onClick={() => setIsSidebarOpen(true)}
          className="fixed bottom-6 right-6 z-40 flex items-center gap-2 rounded-2xl border border-[#D4AF37] bg-gradient-to-r from-[#0F223F] to-[#0A1424] px-4 py-2.5 text-xs font-bold text-[#F5D77F] shadow-[0_10px_25px_rgba(0,0,0,0.8),0_0_20px_rgba(212,175,55,0.25)] backdrop-blur-md transition-all hover:scale-105 active:scale-95"
        >
          <div className="flex h-6 w-6 shrink-0 items-center justify-center overflow-hidden rounded-full border border-[#D4AF37] bg-[#1A365D] text-[10px] font-bold text-white">
            {selectedMember.avatarUrl ? (
              // eslint-disable-next-line @next/next/no-img-element
              <img src={selectedMember.avatarUrl} alt={selectedMember.name} className="h-full w-full object-cover" />
            ) : (
              selectedMember.avatarInitials
            )}
          </div>
          <span>Ver Expediente (#{selectedMember.code})</span>
        </button>
      )}

      {/* Selector de referido pendiente al pulsar un "+" de pierna libre */}
      {placeTarget && (
        <div
          className="absolute inset-0 z-50 flex items-center justify-center bg-black/70 p-4"
          onClick={() => setPlaceTarget(null)}
        >
          <div
            className="w-full max-w-sm rounded-2xl border border-[#D4AF37]/40 bg-[#0A1424] p-4 shadow-[0_20px_50px_rgba(0,0,0,0.8)]"
            onClick={(e) => e.stopPropagation()}
          >
            <div className="flex items-center justify-between">
              <h3 className="text-sm font-bold text-white">
                Colocar en pierna {placeTarget.leg}
              </h3>
              <button onClick={() => setPlaceTarget(null)} className="text-slate-400 hover:text-white">
                <X className="h-4 w-4" />
              </button>
            </div>
            <p className="mt-1 text-[11px] text-slate-400">
              Elige a cuál de tus referidos directos sin colocar quieres asignar esta posición.
            </p>

            {pending.length === 0 ? (
              <p className="py-6 text-center text-xs text-slate-400">
                No tienes referidos pendientes por colocar.
              </p>
            ) : (
              <div className="mt-3 max-h-64 space-y-1.5 overflow-y-auto">
                {pending.map((m) => (
                  <button
                    key={m.id}
                    disabled={placing}
                    onClick={() => confirmPlace(m.id)}
                    className="flex w-full items-center justify-between rounded-xl border border-slate-700 bg-[#0F223F] px-3 py-2 text-left transition-colors hover:border-[#D4AF37] disabled:opacity-50"
                  >
                    <span className="min-w-0">
                      <span className="block truncate text-xs font-semibold text-white">{m.name}</span>
                      <span className="font-mono text-[10px] text-slate-400">#{m.code}</span>
                    </span>
                    <ChevronRight className="h-4 w-4 shrink-0 text-[#D4AF37]" />
                  </button>
                ))}
              </div>
            )}

            {placeError && (
              <p className="mt-3 rounded-lg border border-rose-500/40 bg-rose-500/10 px-3 py-2 text-[11px] text-rose-300">
                {placeError}
              </p>
            )}
          </div>
        </div>
      )}
    </div>
  );
}

/* ==========================================================================
   COMPONENTE PRINCIPAL (EXPORT DEFAULT APP)
   Envuelve con ReactFlowProvider para permitir el control global de navegación
   ========================================================================== */

export function MlmStructure(props: MlmStructureProps) {
  return (
    <ReactFlowProvider>
      <MLMCanvasViewer {...props} />
    </ReactFlowProvider>
  );
}
```

### 83. `components/tree/MlmStructureClient.tsx`
Puente cliente que conecta la colocación con la Server Action y refresca el árbol.
```tsx
"use client";

import { useRouter } from "next/navigation";
import { MlmStructure, type PendingMember } from "./MlmStructure";
import type { BinaryMemberData, UnilevelMemberData } from "./MlmStructure";
import { placeMemberAction } from "@/app/(distribuidor)/red/actions";

/**
 * Puente cliente del visor: conecta la colocación de referidos pendientes con
 * la acción de servidor y refresca el árbol cuando se coloca a alguien.
 */
export function MlmStructureClient({
  binary,
  unilevel,
  rootId,
  pending,
}: {
  binary: BinaryMemberData[];
  unilevel: UnilevelMemberData[];
  rootId: string;
  pending: PendingMember[];
}) {
  const router = useRouter();

  async function handlePlace(memberId: string, parentId: string, leg: "left" | "right") {
    const res = await placeMemberAction(memberId, parentId, leg);
    if (res.error) return res.error;
    router.refresh();
    return null;
  }

  return (
    <MlmStructure
      binary={binary}
      unilevel={unilevel}
      rootId={rootId}
      pending={pending}
      onPlaceMember={handlePlace}
    />
  );
}
```

### 84. `components/tree/GenealogyTree.tsx`
Árbol anterior de conectores CSS (lo usa el admin).
```tsx
"use client";

import { useEffect, useState, useTransition } from "react";
import { Plus, User, ZoomIn, ZoomOut, RotateCcw, ChevronDown, X } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import { RankBadge } from "@/components/ui/RankBadge";
import { rankNameFor } from "@/lib/ranks";
import type { TreeNode } from "@/lib/data/tree";

type Tree = "binary" | "sponsor";
type LoadFn = (nodeId: string, tree: Tree) => Promise<TreeNode | null>;
type PlaceFn = (
  memberId: string,
  placementId: string,
  leg: "left" | "right"
) => Promise<{ error?: string }>;

export interface PendingMember {
  id: string;
  code: string;
  name: string;
}

interface Props {
  initialRoot: TreeNode;
  rootId: string;
  load: LoadFn;
  place: PlaceFn;
  pending: PendingMember[]; // miembros pendientes que el usuario puede colocar
}

export function GenealogyTree({ initialRoot, rootId, load, place, pending }: Props) {
  const [tree, setTree] = useState<Tree>("binary");
  const [root, setRoot] = useState<TreeNode>(initialRoot);
  const [stack, setStack] = useState<string[]>([]);
  const [zoom, setZoom] = useState(1);
  const [hovered, setHovered] = useState<TreeNode | null>(null);
  const [activeId, setActiveId] = useState<string | null>(null); // nodo con "+" visibles
  const [placing, setPlacing] = useState<{ placementId: string; leg: "left" | "right" } | null>(null);
  const [pendingList, setPendingList] = useState<PendingMember[]>(pending);
  const [error, setError] = useState<string | null>(null);
  const [, startTransition] = useTransition();
  const { t } = useT();

  useEffect(() => setPendingList(pending), [pending]);

  function refresh(nodeId: string, t: Tree) {
    startTransition(async () => {
      const data = await load(nodeId, t);
      if (data) setRoot(data);
    });
  }

  function switchTree(t: Tree) {
    setTree(t);
    setStack([]);
    setActiveId(null);
    refresh(rootId, t);
  }

  function drill(nodeId: string) {
    setStack((s) => [...s, root.id]);
    setActiveId(null);
    refresh(nodeId, tree);
  }

  function goUp() {
    const prev = stack[stack.length - 1];
    if (!prev) return;
    setStack((s) => s.slice(0, -1));
    setActiveId(null);
    refresh(prev, tree);
  }

  async function doPlace(memberId: string) {
    if (!placing) return;
    setError(null);
    const res = await place(memberId, placing.placementId, placing.leg);
    if (res.error) {
      setError(res.error);
      return;
    }
    setPendingList((list) => list.filter((m) => m.id !== memberId));
    setPlacing(null);
    setActiveId(null);
    refresh(root.id, tree);
  }

  return (
    <div className="relative">
      {/* Tabs + controles */}
      <div className="mb-4 flex flex-wrap items-center justify-between gap-3">
        <div className="inline-flex rounded-lg border border-border p-1">
          {(["binary", "sponsor"] as const).map((tb) => (
            <button
              key={tb}
              onClick={() => switchTree(tb)}
              className={`rounded-md px-4 py-1.5 text-xs font-medium transition-colors ${
                tree === tb ? "bg-gold text-goldInk" : "text-muted hover:text-cream"
              }`}
            >
              {tb === "binary" ? t("net.treeBinary") : t("net.treeSponsor")}
            </button>
          ))}
        </div>
        <div className="flex items-center gap-2">
          {stack.length > 0 && (
            <button onClick={goUp} className="rounded-md border border-border px-3 py-1.5 text-xs text-cream hover:border-gold">
              {t("net.up")}
            </button>
          )}
          <button onClick={() => setZoom((z) => Math.max(0.5, z - 0.1))} className="rounded-md border border-border p-2 text-cream hover:border-gold" aria-label="Alejar">
            <ZoomOut size={16} />
          </button>
          <button onClick={() => setZoom((z) => Math.min(1.5, z + 0.1))} className="rounded-md border border-border p-2 text-cream hover:border-gold" aria-label="Acercar">
            <ZoomIn size={16} />
          </button>
          <button onClick={() => setZoom(1)} className="rounded-md border border-border p-2 text-cream hover:border-gold" aria-label="Restablecer">
            <RotateCcw size={16} />
          </button>
        </div>
      </div>

      {tree === "binary" && (
        <p className="mb-3 text-xs text-muted">{t("net.hoverTip")}</p>
      )}

      {/* Lienzo */}
      <div className="overflow-auto rounded-card border border-border bg-navyDeep/40 p-6">
        <div className="tree inline-block min-w-full origin-top transition-transform" style={{ transform: `scale(${zoom})` }}>
          <ul>
            <li>
              <RenderNode
                node={root}
                tree={tree}
                isRoot
                activeId={activeId}
                onHover={setHovered}
                onToggleActive={(id) => setActiveId((cur) => (cur === id ? null : id))}
                onDrill={drill}
                onPlus={(placementId, leg) => {
                  setError(null);
                  setPlacing({ placementId, leg });
                }}
              />
            </li>
          </ul>
        </div>
      </div>

      {/* Tarjeta de datos (hover) */}
      {hovered && !placing && (
        <DetailCard node={hovered} tree={tree} />
      )}

      {/* Selector de pendiente al pulsar "+" */}
      {placing && (
        <PendingPicker
          pending={pendingList}
          error={error}
          onPick={doPlace}
          onClose={() => {
            setPlacing(null);
            setError(null);
          }}
        />
      )}
    </div>
  );
}

function RenderNode({
  node,
  tree,
  isRoot = false,
  activeId,
  onHover,
  onToggleActive,
  onDrill,
  onPlus,
}: {
  node: TreeNode;
  tree: Tree;
  isRoot?: boolean;
  activeId: string | null;
  onHover: (n: TreeNode | null) => void;
  onToggleActive: (id: string) => void;
  onDrill: (id: string) => void;
  onPlus: (placementId: string, leg: "left" | "right") => void;
}) {
  const { t } = useT();
  const isActive = activeId === node.id;
  // En binario, los "+" de un nodo se muestran al hacerle clic (isActive).
  const showBinaryChildren =
    tree === "binary" && (node.left || node.right || isActive || isRoot);
  const showSponsorChildren = tree === "sponsor" && node.children.length > 0;

  return (
    <>
      <NodeCard
        node={node}
        tree={tree}
        onHover={onHover}
        onClick={() => onToggleActive(node.id)}
      />

      {node.hasMore && (
        <button onClick={() => onDrill(node.id)} className="mx-auto mt-1 flex items-center gap-1 text-[11px] text-cobalt hover:text-gold">
          <ChevronDown size={12} /> {t("net.viewMore")}
        </button>
      )}

      {showBinaryChildren && (
        <ul>
          <li>
            {node.left ? (
              <RenderNode node={node.left} tree={tree} activeId={activeId} onHover={onHover} onToggleActive={onToggleActive} onDrill={onDrill} onPlus={onPlus} />
            ) : isActive || isRoot ? (
              <PlusSlot onClick={() => onPlus(node.id, "left")} />
            ) : (
              <span className="inline-block h-2" />
            )}
          </li>
          <li>
            {node.right ? (
              <RenderNode node={node.right} tree={tree} activeId={activeId} onHover={onHover} onToggleActive={onToggleActive} onDrill={onDrill} onPlus={onPlus} />
            ) : isActive || isRoot ? (
              <PlusSlot onClick={() => onPlus(node.id, "right")} />
            ) : (
              <span className="inline-block h-2" />
            )}
          </li>
        </ul>
      )}

      {showSponsorChildren && (
        <ul>
          {node.children.map((c) => (
            <li key={c.id}>
              <RenderNode node={c} tree={tree} activeId={activeId} onHover={onHover} onToggleActive={onToggleActive} onDrill={onDrill} onPlus={onPlus} />
            </li>
          ))}
        </ul>
      )}
    </>
  );
}

function NodeCard({
  node,
  tree,
  onHover,
  onClick,
}: {
  node: TreeNode;
  tree: Tree;
  onHover: (n: TreeNode | null) => void;
  onClick: () => void;
}) {
  const { t } = useT();
  return (
    <button
      onClick={onClick}
      onMouseEnter={() => onHover(node)}
      onMouseLeave={() => onHover(null)}
      className="mx-auto inline-flex flex-col items-center"
    >
      <span className={`flex h-14 w-14 items-center justify-center rounded-full border-2 ${node.status === "active" ? "border-gold" : "border-border"} bg-card`}>
        <User size={24} className="text-muted" />
      </span>
      <span className="mt-1 max-w-[110px] truncate text-xs font-semibold text-cream">{node.code}</span>
      <span className="text-[10px] text-muted">
        {tree === "binary"
          ? `${t("net.left")}: ${node.leftCount}  ${t("net.right")}: ${node.rightCount}`
          : `${t("net.referrals")}: ${node.childrenCount}`}
      </span>
    </button>
  );
}

function PlusSlot({ onClick }: { onClick: () => void }) {
  return (
    <button
      onClick={onClick}
      className="mx-auto flex h-10 w-10 items-center justify-center rounded-full bg-cobalt text-white transition-transform hover:scale-110"
      title="Colocar un pendiente aquí"
    >
      <Plus size={18} />
    </button>
  );
}

function PendingPicker({
  pending,
  error,
  onPick,
  onClose,
}: {
  pending: PendingMember[];
  error: string | null;
  onPick: (id: string) => void;
  onClose: () => void;
}) {
  const { t } = useT();
  return (
    <div className="fixed inset-0 z-50 flex items-end justify-center bg-navyDeep/60 p-4 sm:items-center">
      <div className="w-full max-w-md rounded-card border border-gold/40 bg-card p-5 shadow-card">
        <div className="flex items-center justify-between">
          <h3 className="font-display text-lg font-bold text-cream">{t("net.placeTitle")}</h3>
          <button onClick={onClose} className="text-muted hover:text-cream" aria-label="X">
            <X size={18} />
          </button>
        </div>
        <p className="mt-1 text-xs text-muted">{t("net.placeTip")}</p>

        {error && (
          <p className="mt-3 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{error}</p>
        )}

        <div className="mt-4 max-h-72 space-y-2 overflow-auto">
          {pending.length === 0 ? (
            <p className="text-sm text-muted">{t("net.noPending")}</p>
          ) : (
            pending.map((m) => (
              <button
                key={m.id}
                onClick={() => onPick(m.id)}
                className="flex w-full items-center justify-between rounded-lg border border-border bg-navyDeep px-3 py-2 text-left hover:border-gold"
              >
                <span>
                  <span className="block text-sm text-cream">{m.name}</span>
                  <span className="block text-[11px] text-muted">{m.code}</span>
                </span>
                <span className="text-xs font-semibold text-gold">{t("net.place")}</span>
              </button>
            ))
          )}
        </div>
      </div>
    </div>
  );
}

function DetailCard({ node, tree }: { node: TreeNode; tree: Tree }) {
  const { t } = useT();
  return (
    <div className="pointer-events-none fixed inset-x-0 bottom-6 z-40 flex justify-center px-4">
      <div className="pointer-events-auto w-full max-w-md rounded-card border border-gold/40 bg-card p-5 shadow-card">
        <div className="flex items-center gap-3">
          <RankBadge level={node.rank} size="md" />
          <div>
            <p className="font-display text-base font-bold text-cream">{node.code}</p>
            <p className="text-xs text-muted">{node.name}</p>
          </div>
        </div>
        <div className="mt-4 grid grid-cols-2 gap-x-6 gap-y-2 text-sm">
          <Row label={t("net.pvPersonal")} value={node.personalPV} />
          <Row label={t("dash.classification")} value={node.rank > 0 ? rankNameFor(node.rank) : "—"} />
          <Row label={t("net.volLeft")} value={node.leftVolume} />
          <Row label={t("net.volRight")} value={node.rightVolume} />
          {tree === "binary" ? (
            <>
              <Row label={t("net.membersLeft")} value={node.leftCount} />
              <Row label={t("net.membersRight")} value={node.rightCount} />
            </>
          ) : (
            <Row label={t("net.referrals")} value={node.childrenCount} />
          )}
        </div>
      </div>
    </div>
  );
}

function Row({ label, value }: { label: string; value: number | string }) {
  return (
    <div className="flex justify-between border-b border-border/50 py-1">
      <span className="text-muted">{label}</span>
      <span className="font-medium text-cream">{value}</span>
    </div>
  );
}
```


---

# PARTE 10 — HUB TRIADA (CENTRO DE IA)
Una sola app con pestañas, panel lateral de historial y panel derecho estático de ajustes por sección.

### 85. `components/dashboard/ai/types.ts`
Pestañas del Hub.
```ts
/** Secciones del Hub Triada (pestañas horizontales). */
export type AiMode =
  | "chat"
  | "agentes"
  | "code"
  | "imagenes"
  | "editor"
  | "audio"
  | "video"
  | "superprompter"
  | "tendencias";
```

### 86. `components/dashboard/ai/AiHub.tsx`
Orquestador: mantiene todos los paneles montados (visibilidad por CSS) para conservar su estado al cambiar de pestaña.
```tsx
"use client";

import { useState } from "react";
import { Video } from "lucide-react";
import type { AiAssistant, AiConversationSummary, AiModel } from "@/lib/ai/hub";
import type { AiCreation, PublicCreation } from "@/lib/ai/creative";
import type { ActivityData } from "@/lib/ai/activity";
import type { AutoCapabilities } from "@/lib/ai/router";
import type { AiMode } from "./types";
import { TopNav } from "./TopNav";
import { ProfileMenu } from "./ProfileMenu";
import { ChatPane, type ChatLaunch } from "./ChatPane";
import { AgentsPane } from "./AgentsPane";
import { ImagePane } from "./ImagePane";
import { EditorPane } from "./EditorPane";
import { AudioPane } from "./AudioPane";
import { TrendsPane } from "./TrendsPane";
import { SuperPrompterPane } from "./SuperPrompterPane";
import { ComingSoonPane } from "./ComingSoonPane";

interface Props {
  userName: string;
  userAvatarUrl?: string | null;
  userTier: number;
  chatModels: AiModel[];
  codeModels: AiModel[];
  imageModels: AiModel[];
  audioModels: AiModel[];
  assistants: AiAssistant[];
  autoCaps: AutoCapabilities;
  codeCaps: AutoCapabilities;
  activity: ActivityData;
  conversations: AiConversationSummary[];
  creations: AiCreation[];
  publicCreations: PublicCreation[];
  initialBalance: number;
}

/** Hub Triada — una sola app: nav horizontal + panel lateral + panel de ajustes. */
export function AiHub({
  userName,
  userAvatarUrl,
  userTier,
  chatModels,
  codeModels,
  imageModels,
  audioModels,
  assistants,
  autoCaps,
  codeCaps,
  activity,
  conversations,
  creations,
  publicCreations,
  initialBalance,
}: Props) {
  const [mode, setMode] = useState<AiMode>("chat");
  const [collapsed, setCollapsed] = useState(false);
  const [balance, setBalance] = useState(initialBalance);
  const [recreate, setRecreate] = useState<{ prompt: string; model: string; nonce: number } | null>(null);
  const [launch, setLaunch] = useState<ChatLaunch | null>(null);

  function handleRecreate(prompt: string, modelCode: string) {
    setRecreate({ prompt, model: modelCode, nonce: Date.now() });
    setMode("imagenes");
    setCollapsed(false);
  }

  function handleUseFromPrompter(prompt: string, targetMode: "imagenes" | "video") {
    setRecreate({ prompt, model: "", nonce: Date.now() });
    setMode(targetMode === "video" ? "video" : "imagenes");
  }

  /** Desde Agentes: abre el chat con ese agente ya puesto. */
  function handleStartAgent(code: string) {
    setLaunch({ assistant: code, nonce: Date.now() });
    setMode("chat");
  }

  return (
    <div className="flex h-[calc(100vh-8rem)] flex-col">
      <div className="mb-4 flex flex-wrap items-center justify-between gap-3">
        <TopNav mode={mode} onChange={setMode} />
        <ProfileMenu
          balance={balance}
          name={userName}
          avatarUrl={userAvatarUrl}
          tier={userTier}
          onOpenGallery={() => setMode("imagenes")}
        />
      </div>

      <div className={mode === "chat" ? "min-h-0 flex-1" : "hidden"}>
        <ChatPane
          variant="chat"
          models={chatModels}
          assistants={assistants}
          autoCaps={autoCaps}
          activity={activity}
          initialConversations={conversations}
          balance={balance}
          onBalanceChange={setBalance}
          collapsed={collapsed}
          onToggleCollapsed={() => setCollapsed((c) => !c)}
          launch={launch}
        />
      </div>

      <div className={mode === "agentes" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <AgentsPane assistants={assistants} onStart={handleStartAgent} />
      </div>

      <div className={mode === "code" ? "min-h-0 flex-1" : "hidden"}>
        <ChatPane
          variant="code"
          models={codeModels}
          assistants={[]}
          autoCaps={codeCaps}
          activity={null}
          initialConversations={[]}
          balance={balance}
          onBalanceChange={setBalance}
          collapsed={collapsed}
          onToggleCollapsed={() => setCollapsed((c) => !c)}
        />
      </div>

      <div className={mode === "imagenes" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <ImagePane
          models={imageModels}
          initialCreations={creations}
          balance={balance}
          onBalanceChange={setBalance}
          collapsed={collapsed}
          onToggleCollapsed={() => setCollapsed((c) => !c)}
          recreate={recreate}
        />
      </div>

      <div className={mode === "editor" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <EditorPane
          models={imageModels}
          initialCreations={creations}
          balance={balance}
          onBalanceChange={setBalance}
          collapsed={collapsed}
          onToggleCollapsed={() => setCollapsed((c) => !c)}
        />
      </div>

      <div className={mode === "audio" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <AudioPane models={audioModels} balance={balance} onBalanceChange={setBalance} />
      </div>

      <div className={mode === "video" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <ComingSoonPane icon={Video} titleKey="aihub.tabVideo" descKey="aihub.videoSoonDesc" />
      </div>

      <div className={mode === "superprompter" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <SuperPrompterPane balance={balance} onBalanceChange={setBalance} onUse={handleUseFromPrompter} />
      </div>

      <div className={mode === "tendencias" ? "min-h-0 flex-1 overflow-y-auto" : "hidden"}>
        <TrendsPane creations={publicCreations} onRecreate={handleRecreate} />
      </div>
    </div>
  );
}
```

### 87. `components/dashboard/ai/TopNav.tsx`
Navegación horizontal de pestañas.
```tsx
"use client";

import {
  MessageSquare,
  Bot,
  Code2,
  ImagePlus,
  Pencil,
  Video,
  Music,
  Wand2,
  TrendingUp,
  type LucideIcon,
} from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiMode } from "./types";

const TABS: { id: AiMode; labelKey: string; icon: LucideIcon }[] = [
  { id: "chat", labelKey: "aihub.tabChat", icon: MessageSquare },
  { id: "agentes", labelKey: "aihub.tabAgents", icon: Bot },
  { id: "code", labelKey: "aihub.tabCode", icon: Code2 },
  { id: "imagenes", labelKey: "aihub.tabImages", icon: ImagePlus },
  { id: "editor", labelKey: "aihub.tabEditor", icon: Pencil },
  { id: "audio", labelKey: "aihub.tabAudio", icon: Music },
  { id: "video", labelKey: "aihub.tabVideo", icon: Video },
  { id: "superprompter", labelKey: "aihub.tabSuperPrompt", icon: Wand2 },
  { id: "tendencias", labelKey: "aihub.tabTrends", icon: TrendingUp },
];

export function TopNav({ mode, onChange }: { mode: AiMode; onChange: (m: AiMode) => void }) {
  const { t } = useT();
  return (
    <nav className="flex items-center gap-1 overflow-x-auto">
      {TABS.map((tab) => {
        const Icon = tab.icon;
        const active = mode === tab.id;
        return (
          <button
            key={tab.id}
            onClick={() => onChange(tab.id)}
            className={`flex shrink-0 items-center gap-1.5 rounded-lg px-3 py-2 text-sm font-medium transition-colors ${
              active ? "bg-gold text-goldInk" : "text-muted hover:bg-card hover:text-cream"
            }`}
          >
            <Icon size={15} />
            {t(tab.labelKey)}
          </button>
        );
      })}
    </nav>
  );
}
```

### 88. `components/dashboard/ai/ProfileMenu.tsx`
Menú de perfil del Hub: foto, paquete, créditos, perfil, comprar créditos y cerrar sesión.
```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import Link from "next/link";
import { User as UserIcon, ChevronDown, Image as ImageIcon, ShoppingBag, LogOut } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import { logoutAction } from "@/app/(auth)/actions";

const TIER_NAME: Record<number, string> = { 1: "Básico", 2: "Estándar", 3: "Nova", 4: "Quantum" };

export function ProfileMenu({
  balance,
  name,
  avatarUrl,
  tier,
  onOpenGallery,
}: {
  balance: number;
  name: string;
  avatarUrl?: string | null;
  tier: number;
  onOpenGallery: () => void;
}) {
  const { t } = useT();
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    function onClick(e: MouseEvent) {
      if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false);
    }
    document.addEventListener("mousedown", onClick);
    return () => document.removeEventListener("mousedown", onClick);
  }, []);

  const tierLabel = TIER_NAME[tier] ?? t("aihub.noPlan");

  return (
    <div className="relative" ref={ref}>
      <button
        onClick={() => setOpen((o) => !o)}
        className="flex items-center gap-2 rounded-full border border-border bg-navyDeep py-1 pl-1 pr-3 hover:border-gold"
      >
        <span className="flex h-7 w-7 items-center justify-center overflow-hidden rounded-full bg-card">
          {avatarUrl ? (
            // eslint-disable-next-line @next/next/no-img-element
            <img src={avatarUrl} alt="" className="h-full w-full object-cover" />
          ) : (
            <UserIcon size={14} className="text-gold" />
          )}
        </span>
        <span className="hidden text-xs font-semibold text-gold sm:inline">{balance.toLocaleString()}</span>
        <ChevronDown size={13} className="text-muted" />
      </button>

      {open && (
        <div className="absolute right-0 z-50 mt-2 w-64 overflow-hidden rounded-card border border-border bg-card shadow-card">
          <div className="flex flex-col items-center border-b border-border px-4 py-5">
            <span className="flex h-16 w-16 items-center justify-center overflow-hidden rounded-full border-2 border-gold/50 bg-navyDeep">
              {avatarUrl ? (
                // eslint-disable-next-line @next/next/no-img-element
                <img src={avatarUrl} alt="" className="h-full w-full object-cover" />
              ) : (
                <UserIcon size={26} className="text-gold" />
              )}
            </span>
            <p className="mt-2 truncate text-sm font-semibold text-cream">{name}</p>
            <span className="mt-1 rounded-full bg-gold/15 px-2.5 py-0.5 text-[10px] font-bold uppercase tracking-wide text-gold">
              {tierLabel}
            </span>
            <p className="mt-2 flex items-center gap-1 text-sm font-semibold text-gold">
              ⚡ {balance.toLocaleString()} {t("ia.creditsShort")}
            </p>
          </div>
          <nav className="py-1">
            <Link
              href="/perfil"
              onClick={() => setOpen(false)}
              className="flex items-center gap-3 px-4 py-2.5 text-sm text-cream hover:bg-navyDeep"
            >
              <UserIcon size={16} className="text-muted" /> {t("menu.profile")}
            </Link>
            <button
              onClick={() => {
                setOpen(false);
                onOpenGallery();
              }}
              className="flex w-full items-center gap-3 px-4 py-2.5 text-left text-sm text-cream hover:bg-navyDeep"
            >
              <ImageIcon size={16} className="text-muted" /> {t("aihub.myGallery")}
            </button>
            <Link
              href="/tienda"
              onClick={() => setOpen(false)}
              className="flex items-center gap-3 px-4 py-2.5 text-sm text-cream hover:bg-navyDeep"
            >
              <ShoppingBag size={16} className="text-muted" /> {t("aihub.buyCredits")}
            </Link>
            <form action={logoutAction} className="border-t border-border">
              <button className="flex w-full items-center gap-3 px-4 py-2.5 text-left text-sm text-danger hover:bg-navyDeep">
                <LogOut size={16} /> {t("common.logout")}
              </button>
            </form>
          </nav>
        </div>
      )}
    </div>
  );
}
```

### 89. `components/dashboard/ai/SidebarShell.tsx`
Panel izquierdo colapsable (historial).
```tsx
"use client";

import { ChevronLeft, ChevronRight } from "lucide-react";

/** Panel lateral colapsable reutilizado por todas las secciones del Centro de IA. */
export function SidebarShell({
  collapsed,
  onToggle,
  children,
}: {
  collapsed: boolean;
  onToggle: () => void;
  children: React.ReactNode;
}) {
  return (
    <div className={`relative hidden shrink-0 transition-all duration-200 md:block ${collapsed ? "w-0" : "w-60"}`}>
      <button
        onClick={onToggle}
        aria-label={collapsed ? "Mostrar panel" : "Ocultar panel"}
        className="absolute -right-3 top-3 z-10 flex h-6 w-6 items-center justify-center rounded-full border border-border bg-navyDeep text-muted hover:border-gold hover:text-cream"
      >
        {collapsed ? <ChevronRight size={13} /> : <ChevronLeft size={13} />}
      </button>
      {!collapsed && (
        <div className="h-full overflow-hidden rounded-card border border-border bg-navyDeep/40 p-3">
          {children}
        </div>
      )}
    </div>
  );
}
```

### 90. `components/dashboard/ai/RightPanel.tsx`
Panel derecho estático colapsable con Restablecer / Ocultar todo.
```tsx
"use client";

import { ChevronLeft, ChevronRight, RotateCcw, EyeOff } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";

/**
 * Panel derecho ESTÁTICO (no popover) de configuración: Selección de modelo +
 * ajustes propios de cada sección (Formato/Calidad en Imágenes, etc). Se puede
 * ocultar por completo con "Ocultar todo", igual que el panel de historial.
 */
export function RightPanel({
  collapsed,
  onToggle,
  onReset,
  children,
}: {
  collapsed: boolean;
  onToggle: () => void;
  onReset?: () => void;
  children: React.ReactNode;
}) {
  const { t } = useT();
  return (
    <div className={`relative hidden shrink-0 transition-all duration-200 lg:block ${collapsed ? "w-0" : "w-72"}`}>
      <button
        onClick={onToggle}
        aria-label={collapsed ? t("aihub.showPanel") : t("aihub.hidePanel")}
        className="absolute -left-3 top-3 z-10 flex h-6 w-6 items-center justify-center rounded-full border border-border bg-navyDeep text-muted hover:border-gold hover:text-cream"
      >
        {collapsed ? <ChevronLeft size={13} /> : <ChevronRight size={13} />}
      </button>
      {!collapsed && (
        <div className="flex h-full flex-col gap-3 overflow-y-auto pl-1">
          {children}
          <div className="mt-auto flex gap-2 pt-1">
            {onReset && (
              <button
                onClick={onReset}
                className="flex flex-1 items-center justify-center gap-1.5 rounded-lg border border-border px-2 py-2 text-[11px] font-medium text-muted hover:border-gold hover:text-cream"
              >
                <RotateCcw size={12} /> {t("aihub.resetAll")}
              </button>
            )}
            <button
              onClick={onToggle}
              className="flex flex-1 items-center justify-center gap-1.5 rounded-lg border border-border px-2 py-2 text-[11px] font-medium text-muted hover:border-gold hover:text-cream"
            >
              <EyeOff size={12} /> {t("aihub.hideAll")}
            </button>
          </div>
        </div>
      )}
    </div>
  );
}
```

### 91. `components/dashboard/ai/CollapsibleSection.tsx`
Tarjeta plegable de ajustes.
```tsx
"use client";

import { useState } from "react";
import { ChevronUp, ChevronDown, type LucideIcon } from "lucide-react";

/** Tarjeta colapsable del panel derecho (Selección de modelo / Formato / Calidad...). */
export function CollapsibleSection({
  title,
  icon: Icon,
  defaultOpen = true,
  children,
}: {
  title: string;
  icon?: LucideIcon;
  defaultOpen?: boolean;
  children: React.ReactNode;
}) {
  const [open, setOpen] = useState(defaultOpen);
  return (
    <div className="rounded-card border border-border bg-card">
      <button
        onClick={() => setOpen((o) => !o)}
        className="flex w-full items-center gap-2 px-3 py-2.5 text-left text-xs font-semibold uppercase tracking-label text-cream"
      >
        {Icon && <Icon size={13} className="text-gold" />}
        <span className="flex-1">{title}</span>
        {open ? <ChevronUp size={13} className="text-muted" /> : <ChevronDown size={13} className="text-muted" />}
      </button>
      {open && <div className="border-t border-border p-3">{children}</div>}
    </div>
  );
}
```

### 92. `components/dashboard/ai/ModelSection.tsx`
Selección de modelo: Triada Auto Max o catálogo con buscador, agrupado por proveedor, con costo y capacidades.
```tsx
"use client";

import { useMemo, useState } from "react";
import { Sparkles, Image as ImageIcon, FileText, Mic, Check, Search } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiModel } from "@/lib/ai/hub";

/** O deja elegir a Triada Auto Max, o el usuario fija un modelo del catálogo. */
export type ModelSelection = { auto: true } | { manual: string };

export const AUTO: ModelSelection = { auto: true };

/**
 * Selección de modelo. Vive DENTRO de un CollapsibleSection del panel derecho.
 *
 * El catálogo tiene cientos de modelos, así que se agrupa por proveedor y se
 * filtra con un buscador en vez de mostrarlos todos de golpe.
 */
export function ModelSection({
  models,
  selection,
  onChange,
  showAuto = true,
}: {
  models: AiModel[];
  selection: ModelSelection;
  onChange: (s: ModelSelection) => void;
  showAuto?: boolean;
}) {
  const { t } = useT();
  const [query, setQuery] = useState("");

  const groups = useMemo(() => {
    const q = query.trim().toLowerCase();
    const list = q
      ? models.filter((m) => m.name.toLowerCase().includes(q) || m.provider.toLowerCase().includes(q))
      : models;
    const byProvider = new Map<string, AiModel[]>();
    for (const m of list) {
      const arr = byProvider.get(m.provider) ?? [];
      arr.push(m);
      byProvider.set(m.provider, arr);
    }
    return [...byProvider.entries()];
  }, [models, query]);

  return (
    <div className="space-y-3">
      {showAuto && (
        <button
          onClick={() => onChange(AUTO)}
          className={`flex w-full items-start gap-2 rounded-lg border px-2.5 py-2 text-left ${
            "auto" in selection
              ? "border-gold/50 bg-gold/15 text-gold"
              : "border-border text-cream hover:bg-navyDeep"
          }`}
        >
          <Sparkles size={15} className="mt-0.5 shrink-0" />
          <span className="min-w-0 flex-1">
            <span className="block text-sm font-semibold">{t("ia.autoMax")}</span>
            <span className="block text-[11px] leading-snug text-muted">{t("ia.autoMaxHint")}</span>
          </span>
          {"auto" in selection && <Check size={14} className="mt-0.5 shrink-0" />}
        </button>
      )}

      <label className="flex items-center gap-2 rounded-lg border border-border bg-navyDeep px-2.5 py-1.5">
        <Search size={13} className="shrink-0 text-muted" />
        <input
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder={t("aihub.searchModel")}
          className="min-w-0 flex-1 bg-transparent text-xs text-cream outline-none placeholder:text-muted"
        />
        <span className="shrink-0 text-[10px] text-muted">{models.length}</span>
      </label>

      <div className="max-h-80 space-y-2 overflow-y-auto">
        {groups.length === 0 && <p className="py-4 text-center text-xs text-muted">{t("aihub.noModels")}</p>}
        {groups.map(([provider, list]) => (
          <div key={provider}>
            <p className="px-1 pb-1 text-[10px] font-semibold uppercase text-muted/70">{provider}</p>
            {list.map((m) => {
              const on = "manual" in selection && selection.manual === m.code;
              return (
                <button
                  key={m.code}
                  onClick={() => onChange({ manual: m.code })}
                  className={`flex w-full items-center gap-2 rounded-lg px-2 py-1.5 text-left text-sm ${
                    on ? "bg-gold/15 text-gold" : "text-cream hover:bg-navyDeep"
                  }`}
                >
                  <span className="min-w-0 flex-1 truncate" title={m.name}>
                    {m.name}
                  </span>
                  <span className="flex shrink-0 items-center gap-1 text-muted">
                    {m.acceptsImage && <ImageIcon size={11} />}
                    {m.acceptsPdf && <FileText size={11} />}
                    {m.acceptsAudio && <Mic size={11} />}
                  </span>
                  <span className="shrink-0 text-[10px] text-muted">
                    {m.creditCost > 0 ? `${m.creditCost}cr` : t("creative.soonFree")}
                  </span>
                </button>
              );
            })}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 93. `components/dashboard/ai/ChatPane.tsx`
Chat con streaming, historial persistente, adjuntos, dictado por voz, respuestas en markdown y resumen de actividad al abrir un chat nuevo. También sirve para la sección Código.
```tsx
"use client";

import { useEffect, useMemo, useRef, useState } from "react";
import {
  Send,
  Plus,
  Trash2,
  Pencil,
  Check,
  X,
  MessageSquare,
  Image as ImageIcon,
  FileText,
  Music,
  SlidersHorizontal,
  Code2,
} from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiAssistant, AiConversationSummary, AiChatMessage, AiModel } from "@/lib/ai/hub";
import type { ActivityData } from "@/lib/ai/activity";
import type { Attachment } from "@/lib/ai/attachments";
import {
  listConversationsAction,
  loadConversationAction,
  renameConversationAction,
  deleteConversationAction,
} from "@/app/(distribuidor)/ia/actions";
import { SidebarShell } from "./SidebarShell";
import { RightPanel } from "./RightPanel";
import { CollapsibleSection } from "./CollapsibleSection";
import { ModelSection, AUTO, type ModelSelection } from "./ModelSection";
import { AttachButton } from "./AttachButton";
import { DictateButton } from "./DictateButton";
import { ActivitySummary } from "./ActivitySummary";
import { Markdown } from "./Markdown";

type Msg = AiChatMessage & { model?: string; cost?: number };

/** Petición de abrir un chat con un agente concreto (viene de la pestaña Agentes). */
export interface ChatLaunch {
  assistant: string | null;
  nonce: number;
}

export function ChatPane({
  variant = "chat",
  models,
  assistants,
  autoCaps,
  activity,
  initialConversations,
  balance,
  onBalanceChange,
  collapsed,
  onToggleCollapsed,
  launch,
}: {
  variant?: "chat" | "code";
  models: AiModel[];
  assistants: AiAssistant[];
  autoCaps: { accepts_image: boolean; accepts_pdf: boolean; accepts_audio: boolean };
  activity: ActivityData | null;
  initialConversations: AiConversationSummary[];
  balance: number;
  onBalanceChange: (n: number) => void;
  collapsed: boolean;
  onToggleCollapsed: () => void;
  launch?: ChatLaunch | null;
}) {
  const { t } = useT();
  const [selection, setSelection] = useState<ModelSelection>(AUTO);
  const [rightCollapsed, setRightCollapsed] = useState(false);
  const [conversations, setConversations] = useState(initialConversations);
  const [activeId, setActiveId] = useState<string | null>(null);
  const [assistant, setAssistant] = useState<string | null>(null);
  const [messages, setMessages] = useState<Msg[]>([]);
  const [input, setInput] = useState("");
  const [attachments, setAttachments] = useState<Attachment[]>([]);
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [editingId, setEditingId] = useState<string | null>(null);
  const [editTitle, setEditTitle] = useState("");
  const [drawer, setDrawer] = useState(false);
  const endRef = useRef<HTMLDivElement>(null);
  const lastLaunch = useRef(0);

  const assistantMap = useMemo(() => Object.fromEntries(assistants.map((a) => [a.code, a])), [assistants]);
  const activeAssistant = assistant ? assistantMap[assistant] : null;

  const selectedModel = "manual" in selection ? models.find((m) => m.code === selection.manual) : null;
  const caps = selectedModel
    ? {
        accepts_image: selectedModel.acceptsImage,
        accepts_pdf: selectedModel.acceptsPdf,
        accepts_audio: selectedModel.acceptsAudio,
      }
    : autoCaps;

  useEffect(() => {
    endRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  // La pestaña Agentes pide abrir un chat nuevo con un agente ya puesto.
  useEffect(() => {
    if (!launch || launch.nonce === lastLaunch.current) return;
    lastLaunch.current = launch.nonce;
    setActiveId(null);
    setMessages([]);
    setAssistant(launch.assistant);
    setAttachments([]);
    setError(null);
  }, [launch]);

  function newChat() {
    setActiveId(null);
    setMessages([]);
    setAssistant(null);
    setAttachments([]);
    setError(null);
    setDrawer(false);
  }

  async function openConversation(c: AiConversationSummary) {
    setError(null);
    setDrawer(false);
    setActiveId(c.id);
    setAssistant(c.assistant_code ?? null);
    setAttachments([]);
    setMessages(await loadConversationAction(c.id));
  }

  async function refreshList() {
    try {
      setConversations(await listConversationsAction());
    } catch {
      /* ignorar */
    }
  }

  async function doRename(id: string) {
    const title = editTitle.trim();
    setEditingId(null);
    await renameConversationAction(id, title);
    setConversations((cs) => cs.map((c) => (c.id === id ? { ...c, title: title || null } : c)));
  }

  async function doDelete(id: string) {
    if (!confirm(t("ia.confirmDelete"))) return;
    await deleteConversationAction(id);
    setConversations((cs) => cs.filter((c) => c.id !== id));
    if (activeId === id) newChat();
  }

  async function send() {
    const text = input.trim();
    if ((!text && attachments.length === 0) || busy) return;
    setError(null);
    const outgoingAttachments = attachments;
    const base: Msg[] = [...messages, { role: "user", content: text, attachments: outgoingAttachments }];
    setMessages([...base, { role: "assistant", content: "" }]);
    setInput("");
    setAttachments([]);
    setBusy(true);

    try {
      const res = await fetch("/api/ia/chat", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          messages: base,
          assistant,
          conversationId: activeId,
          variant,
          modelCode: "manual" in selection ? selection.manual : undefined,
        }),
      });

      if (!res.ok || !res.body) {
        let msg = t("ia.error");
        try {
          const j = await res.json();
          if (j?.error) msg = j.error;
        } catch {
          /* sin JSON */
        }
        setError(msg);
        setMessages(base);
        return;
      }

      const newId = res.headers.get("X-Conversation-Id");
      if (newId && newId !== activeId) setActiveId(newId);
      const usedModel = res.headers.get("X-Model-Used") ?? undefined;
      const spent = Number(res.headers.get("X-Credits-Spent") ?? 0);
      const newBal = res.headers.get("X-Balance");
      if (newBal !== null) onBalanceChange(Number(newBal));

      const reader = res.body.getReader();
      const dec = new TextDecoder();
      let buf = "";
      let acc = "";
      for (;;) {
        const { done, value } = await reader.read();
        if (done) break;
        buf += dec.decode(value, { stream: true });
        let i: number;
        while ((i = buf.indexOf("\n")) !== -1) {
          const line = buf.slice(0, i).trim();
          buf = buf.slice(i + 1);
          if (!line.startsWith("data:")) continue;
          const d = line.slice(5).trim();
          if (!d || d === "[DONE]") continue;
          try {
            const j = JSON.parse(d);
            const delta: string | undefined = j.choices?.[0]?.delta?.content;
            if (delta) {
              acc += delta;
              setMessages((m) => {
                const copy = m.slice();
                copy[copy.length - 1] = { role: "assistant", content: acc, model: usedModel, cost: spent };
                return copy;
              });
            }
          } catch {
            /* chunk parcial */
          }
        }
      }
      refreshList();
    } catch {
      setError(t("ia.error"));
      setMessages(base);
    } finally {
      setBusy(false);
    }
  }

  function onKey(e: React.KeyboardEvent<HTMLTextAreaElement>) {
    if (e.key === "Enter" && !e.shiftKey) {
      e.preventDefault();
      send();
    }
  }

  const listPanel = (
    <div className="flex h-full flex-col">
      <button
        onClick={newChat}
        className="mb-3 flex items-center justify-center gap-2 rounded-lg border border-gold/50 bg-gold/10 px-3 py-2 text-sm font-semibold text-gold hover:bg-gold/20"
      >
        <Plus size={16} /> {t("ia.newChat")}
      </button>
      <div className="flex-1 space-y-1 overflow-y-auto">
        {conversations.length === 0 && (
          <p className="px-2 py-4 text-center text-xs text-muted">{t("ia.noChats")}</p>
        )}
        {conversations.map((c) => {
          const isActive = c.id === activeId;
          return (
            <div
              key={c.id}
              className={`group flex items-center gap-1 rounded-lg px-2 py-1.5 text-sm ${
                isActive ? "bg-card text-cream" : "text-muted hover:bg-card/50"
              }`}
            >
              {editingId === c.id ? (
                <>
                  <input
                    autoFocus
                    value={editTitle}
                    onChange={(e) => setEditTitle(e.target.value)}
                    onKeyDown={(e) => e.key === "Enter" && doRename(c.id)}
                    className="min-w-0 flex-1 rounded bg-navyDeep px-1.5 py-0.5 text-xs text-cream outline-none"
                  />
                  <button onClick={() => doRename(c.id)} className="text-success" aria-label="Guardar">
                    <Check size={14} />
                  </button>
                  <button onClick={() => setEditingId(null)} className="text-muted" aria-label="Cancelar">
                    <X size={14} />
                  </button>
                </>
              ) : (
                <>
                  <button onClick={() => openConversation(c)} className="flex min-w-0 flex-1 items-center gap-2 text-left">
                    <MessageSquare size={13} className="shrink-0 opacity-60" />
                    <span className="truncate">{c.title || t("ia.untitled")}</span>
                  </button>
                  <button
                    onClick={() => {
                      setEditingId(c.id);
                      setEditTitle(c.title ?? "");
                    }}
                    className="shrink-0 text-muted opacity-0 hover:text-cream group-hover:opacity-100"
                    aria-label={t("ia.rename")}
                  >
                    <Pencil size={13} />
                  </button>
                  <button
                    onClick={() => doDelete(c.id)}
                    className="shrink-0 text-muted opacity-0 hover:text-danger group-hover:opacity-100"
                    aria-label={t("ia.delete")}
                  >
                    <Trash2 size={13} />
                  </button>
                </>
              )}
            </div>
          );
        })}
      </div>
    </div>
  );

  return (
    <div className="flex h-full gap-4">
      <SidebarShell collapsed={collapsed} onToggle={onToggleCollapsed}>
        {listPanel}
      </SidebarShell>

      {drawer && (
        <div className="fixed inset-0 z-50 flex md:hidden" onClick={() => setDrawer(false)}>
          <div className="absolute inset-0 bg-black/50" />
          <div
            className="relative z-10 h-full w-72 max-w-[80%] border-r border-border bg-navyDeep p-3"
            onClick={(e) => e.stopPropagation()}
          >
            {listPanel}
          </div>
        </div>
      )}

      <div className="flex min-w-0 flex-1 flex-col">
        <div className="mb-3 flex flex-wrap items-center justify-between gap-2">
          <div className="flex items-center gap-2">
            <button
              onClick={() => setDrawer(true)}
              className="rounded-md border border-border p-2 text-cream hover:border-gold md:hidden"
              aria-label={t("ia.chats")}
            >
              <MessageSquare size={16} />
            </button>
            {activeAssistant && (
              <p className="flex items-center gap-1 text-sm text-gold">
                {activeAssistant.name}
                <button onClick={() => setAssistant(null)} aria-label="Quitar agente">
                  <X size={12} className="text-muted hover:text-cream" />
                </button>
              </p>
            )}
          </div>
          {rightCollapsed && (
            <button
              onClick={() => setRightCollapsed(false)}
              className="flex items-center gap-1.5 rounded-lg border border-border bg-navyDeep px-3 py-2 text-xs font-medium text-cream hover:border-gold"
            >
              <SlidersHorizontal size={13} className="text-gold" /> {t("aihub.settings")}
            </button>
          )}
        </div>

        <div className="flex-1 space-y-4 overflow-y-auto rounded-card border border-border bg-navyDeep/40 p-4">
          {messages.length === 0 ? (
            <div className="flex h-full flex-col items-center justify-center gap-5">
              {variant === "code" ? (
                <div className="flex flex-col items-center">
                  <span className="flex h-14 w-14 items-center justify-center rounded-full border-2 border-gold/50 bg-navyDeep">
                    <Code2 size={26} className="text-gold" />
                  </span>
                  <p className="mt-3 max-w-sm text-center text-sm text-muted">{t("aihub.codeEmpty")}</p>
                </div>
              ) : activity ? (
                <ActivitySummary data={activity} />
              ) : (
                <p className="text-sm text-muted">{t("ia.empty")}</p>
              )}
            </div>
          ) : (
            messages.map((msg, i) => (
              <div key={i} className={msg.role === "user" ? "flex justify-end" : "flex flex-col items-start"}>
                {(msg.attachments?.length ?? 0) > 0 && (
                  <div className={`mb-1 flex flex-wrap gap-1.5 ${msg.role === "user" ? "justify-end self-end" : ""}`}>
                    {msg.attachments!.map((a, ai) => {
                      const Icon = a.kind === "image" ? ImageIcon : a.kind === "pdf" ? FileText : Music;
                      return (
                        <span
                          key={ai}
                          className="flex items-center gap-1 rounded-full border border-border bg-navyDeep px-2 py-0.5 text-[10px] text-muted"
                        >
                          <Icon size={10} /> {a.name}
                        </span>
                      );
                    })}
                  </div>
                )}
                <div
                  className={
                    msg.role === "user"
                      ? "max-w-[85%] self-end whitespace-pre-wrap rounded-2xl rounded-br-sm bg-gold px-4 py-2.5 text-sm text-goldInk"
                      : "max-w-[85%] min-w-0 rounded-2xl rounded-bl-sm border border-border bg-card px-4 py-2.5 text-sm text-cream"
                  }
                >
                  {msg.role === "assistant" && msg.content ? (
                    <Markdown content={msg.content} />
                  ) : (
                    msg.content || (busy ? <span className="text-muted">{t("ia.thinking")}</span> : "")
                  )}
                </div>
                {msg.role === "assistant" && msg.content && msg.cost !== undefined && (
                  <p className="mt-1 pl-1 text-[10px] text-muted">
                    {msg.model} · {msg.cost} {t("ia.creditsShort")}
                  </p>
                )}
              </div>
            ))
          )}
          <div ref={endRef} />
        </div>

        {error && (
          <p className="mt-2 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{error}</p>
        )}

        {attachments.length > 0 && (
          <div className="mt-2 flex flex-wrap gap-1.5">
            {attachments.map((a, i) => {
              const Icon = a.kind === "image" ? ImageIcon : a.kind === "pdf" ? FileText : Music;
              return (
                <span
                  key={i}
                  className="flex items-center gap-1.5 rounded-full border border-border bg-navyDeep px-2.5 py-1 text-[11px] text-cream"
                >
                  <Icon size={12} className="text-gold" />
                  <span className="max-w-[120px] truncate">{a.name}</span>
                  <button
                    onClick={() => setAttachments((list) => list.filter((_, idx) => idx !== i))}
                    aria-label="Quitar"
                    className="text-muted hover:text-danger"
                  >
                    <X size={11} />
                  </button>
                </span>
              );
            })}
          </div>
        )}

        <div className="mt-3 flex items-end gap-2">
          <AttachButton caps={caps} onAdd={(a) => setAttachments((list) => [...list, a])} onError={setError} disabled={busy} />
          <textarea
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyDown={onKey}
            rows={1}
            placeholder={variant === "code" ? t("aihub.codePlaceholder") : t("ia.placeholder")}
            className="max-h-40 flex-1 resize-none rounded-xl border border-border bg-navyDeep px-4 py-3 text-sm text-cream outline-none focus:border-gold"
          />
          <DictateButton onText={(txt) => setInput((cur) => (cur ? `${cur} ${txt}` : txt))} />
          <button
            onClick={send}
            disabled={busy || (!input.trim() && attachments.length === 0)}
            className="flex h-11 w-11 shrink-0 items-center justify-center rounded-xl bg-gold text-goldInk transition-opacity hover:bg-goldLight disabled:opacity-40"
            aria-label={t("ia.send")}
          >
            <Send size={18} />
          </button>
        </div>
      </div>

      <RightPanel collapsed={rightCollapsed} onToggle={() => setRightCollapsed((c) => !c)} onReset={() => setSelection(AUTO)}>
        <CollapsibleSection title={t("aihub.chooseModel")} icon={SlidersHorizontal}>
          <ModelSection models={models} selection={selection} onChange={setSelection} />
        </CollapsibleSection>
      </RightPanel>
    </div>
  );
}
```

### 94. `components/dashboard/ai/ActivitySummary.tsx`
Resumen de actividad: métricas, mapa de calor y uso por modelo (Todo / 30d / 7d).
```tsx
"use client";

import { useState } from "react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { ActivityData, ActivityRange, ActivityRangeData } from "@/lib/ai/activity";

type Tab = "resumen" | "modelos";

const RANGES: ActivityRange[] = ["todo", "30d", "7d"];

/** Escala del mapa de calor: 0 = sin uso, 4 = día más intenso. */
const HEAT = ["var(--heat-0)", "var(--heat-1)", "var(--heat-2)", "var(--heat-3)", "var(--heat-4)"];

/** Paleta estable por posición en la leyenda (el catálogo cambia, el orden no). */
const SERIES = ["#D4AF37", "#5B8DEF", "#43C59E", "#B98CE8", "#E8845B", "#6FD3E8", "#9AA7BD"];

/** Moby-Dick ≈ 270.000 tokens: unidad de medida simpática para el total. */
const MOBY_TOKENS = 270_000;

function fmt(n: number): string {
  if (n >= 1_000_000) return `${(n / 1_000_000).toFixed(1)}M`;
  if (n >= 1_000) return `${(n / 1_000).toFixed(1)}k`;
  return String(n);
}

export function ActivitySummary({ data }: { data: ActivityData }) {
  const { t } = useT();
  const [tab, setTab] = useState<Tab>("resumen");
  const [range, setRange] = useState<ActivityRange>("todo");
  const current = data[range];

  const hasData = current.metrics.messages > 0;

  return (
    <div className="w-full rounded-card border border-border bg-card p-5">
      <div className="mb-5 flex flex-wrap items-center justify-between gap-3">
        <div className="flex gap-1 rounded-lg bg-navyDeep p-1">
          {(["resumen", "modelos"] as Tab[]).map((x) => (
            <button
              key={x}
              onClick={() => setTab(x)}
              className={`rounded-md px-3 py-1 text-sm transition-colors ${
                tab === x ? "bg-gold text-goldInk" : "text-muted hover:text-cream"
              }`}
            >
              {t(`activity.tab.${x}`)}
            </button>
          ))}
        </div>
        <div className="flex gap-1 text-sm">
          {RANGES.map((r) => (
            <button
              key={r}
              onClick={() => setRange(r)}
              className={`rounded-md px-2.5 py-1 transition-colors ${
                range === r ? "bg-gold/15 text-gold" : "text-muted hover:text-cream"
              }`}
            >
              {t(`activity.range.${r}`)}
            </button>
          ))}
        </div>
      </div>

      {!hasData ? (
        <p className="py-10 text-center text-sm text-muted">{t("activity.empty")}</p>
      ) : tab === "resumen" ? (
        <Resumen data={current} />
      ) : (
        <Modelos data={current} />
      )}
    </div>
  );
}

function Resumen({ data }: { data: ActivityRangeData }) {
  const { t } = useT();
  const { metrics, heat } = data;
  const cols = Math.max(1, Math.ceil(heat.length / 7));
  const moby = metrics.tokens / MOBY_TOKENS;

  const cards = [
    { label: t("activity.sessions"), value: metrics.sessions.toLocaleString() },
    { label: t("activity.messages"), value: metrics.messages.toLocaleString() },
    { label: t("activity.tokens"), value: fmt(metrics.tokens) },
    { label: t("activity.activeDays"), value: String(metrics.activeDays) },
    { label: t("activity.peakHour"), value: metrics.peakHour ?? "—" },
    { label: t("activity.topModel"), value: metrics.topModel ?? "—" },
  ];

  return (
    <>
      <div className="grid grid-cols-2 gap-3 sm:grid-cols-3">
        {cards.map((c) => (
          <div key={c.label} className="rounded-xl border border-border bg-navyDeep/60 p-3">
            <div className="text-xs text-muted">{c.label}</div>
            <div className="mt-1 truncate text-xl font-semibold text-cream" title={c.value}>
              {c.value}
            </div>
          </div>
        ))}
      </div>

      <div className="mt-5 overflow-x-auto pb-1">
        <div
          className="grid grid-flow-col gap-[3px]"
          style={{ gridTemplateRows: "repeat(7, 12px)", gridTemplateColumns: `repeat(${cols}, 12px)` }}
        >
          {heat.map((level, i) => (
            <div
              key={i}
              title={`${t("activity.level")} ${level}`}
              className="rounded-[3px] transition hover:ring-1 hover:ring-gold/60"
              style={{ background: HEAT[level] }}
            />
          ))}
        </div>
      </div>

      {moby >= 0.1 && (
        <p className="mt-4 text-sm text-muted">
          {t("activity.moby").replace("{n}", moby >= 1 ? moby.toFixed(0) : moby.toFixed(1))}
        </p>
      )}
    </>
  );
}

function Modelos({ data }: { data: ActivityRangeData }) {
  const { bars, labels, legend } = data;
  const color = new Map(legend.map((r, i) => [r.model, SERIES[i % SERIES.length]]));
  const max = Math.max(...bars.map((b) => b.segments.reduce((a, s) => a + s.value, 0)), 1);
  const gap = bars.length > 40 ? 2 : bars.length > 12 ? 6 : 24;

  return (
    <>
      {/* Tope de ancho por barra: con uno o dos días, sin él la barra se
          estiraría hasta ocupar todo el gráfico y parecería un bloque. */}
      <div className="flex h-48 items-end border-b border-border px-1" style={{ gap }}>
        {bars.map((b, i) => (
          <div key={b.date} className="flex h-full flex-1 flex-col-reverse" style={{ maxWidth: 56 }} title={b.date}>
            {b.segments.map((s, j) => (
              <div
                key={j}
                title={`${s.model} · ${fmt(s.value)}`}
                className="first:rounded-b-[2px] last:rounded-t-[2px]"
                style={{
                  height: `${(s.value / max) * 100}%`,
                  background: color.get(s.model),
                  animation: "activityGrow .5s ease both",
                  animationDelay: `${Math.min(i * 8, 400)}ms`,
                }}
              />
            ))}
          </div>
        ))}
      </div>

      <div className="mt-2 flex justify-between text-[11px] text-muted">
        {labels.map((l, i) => (
          <span key={`${l}-${i}`}>{l}</span>
        ))}
      </div>

      <div className="mt-5 space-y-2">
        {legend.map((r) => (
          <div key={r.model} className="flex items-center gap-3 text-sm">
            <span className="h-2.5 w-2.5 shrink-0 rounded-sm" style={{ background: color.get(r.model) }} />
            <span className="w-32 shrink-0 truncate text-cream" title={r.model}>
              {r.model}
            </span>
            <span className="flex-1 truncate text-muted">
              {fmt(r.input)} ↑ · {fmt(r.output)} ↓
            </span>
            <span className="shrink-0 text-cream">{r.pct.toFixed(1)}%</span>
          </div>
        ))}
      </div>

      <style>{`@keyframes activityGrow{from{transform:scaleY(0)}to{transform:scaleY(1)}}`}</style>
    </>
  );
}
```

### 95. `components/dashboard/ai/Markdown.tsx`
Renderizador de markdown con bloques de código y botón copiar.
```tsx
"use client";

import { memo, useState } from "react";
import ReactMarkdown from "react-markdown";
import remarkGfm from "remark-gfm";
import { Check, Copy } from "lucide-react";

/** Bloque de código con lenguaje y botón de copiar. */
function CodeBlock({ language, code }: { language: string; code: string }) {
  const [copied, setCopied] = useState(false);

  async function copy() {
    try {
      await navigator.clipboard.writeText(code);
      setCopied(true);
      setTimeout(() => setCopied(false), 1500);
    } catch {
      /* portapapeles no disponible */
    }
  }

  return (
    <div className="my-2 overflow-hidden rounded-lg border border-border bg-navyDeep">
      <div className="flex items-center justify-between border-b border-border px-3 py-1.5">
        <span className="text-[10px] font-semibold uppercase tracking-wide text-muted">
          {language || "código"}
        </span>
        <button
          onClick={copy}
          className="flex items-center gap-1 text-[10px] text-muted transition-colors hover:text-gold"
        >
          {copied ? <Check size={11} /> : <Copy size={11} />}
          {copied ? "Copiado" : "Copiar"}
        </button>
      </div>
      <pre className="overflow-x-auto px-3 py-2.5 text-[12.5px] leading-relaxed">
        <code className="font-mono text-cream">{code}</code>
      </pre>
    </div>
  );
}

/**
 * Renderiza la respuesta del modelo como markdown. Sin esto los bloques de
 * código salían con los backticks crudos, que en la sección Código es
 * especialmente ilegible.
 */
export const Markdown = memo(function Markdown({ content }: { content: string }) {
  return (
    <div className="space-y-2 text-sm leading-relaxed [&>*:first-child]:mt-0 [&>*:last-child]:mb-0">
      <ReactMarkdown
        remarkPlugins={[remarkGfm]}
        components={{
          p: ({ children }) => <p className="whitespace-pre-wrap">{children}</p>,
          h1: ({ children }) => <h1 className="mt-3 text-base font-semibold text-cream">{children}</h1>,
          h2: ({ children }) => <h2 className="mt-3 text-[15px] font-semibold text-cream">{children}</h2>,
          h3: ({ children }) => <h3 className="mt-3 text-sm font-semibold text-cream">{children}</h3>,
          ul: ({ children }) => <ul className="ml-5 list-disc space-y-1">{children}</ul>,
          ol: ({ children }) => <ol className="ml-5 list-decimal space-y-1">{children}</ol>,
          a: ({ href, children }) => (
            <a href={href} target="_blank" rel="noreferrer" className="text-gold underline hover:text-goldLight">
              {children}
            </a>
          ),
          blockquote: ({ children }) => (
            <blockquote className="border-l-2 border-gold/40 pl-3 text-muted">{children}</blockquote>
          ),
          strong: ({ children }) => <strong className="font-semibold text-cream">{children}</strong>,
          hr: () => <hr className="border-border" />,
          table: ({ children }) => (
            <div className="overflow-x-auto">
              <table className="w-full border-collapse text-[13px]">{children}</table>
            </div>
          ),
          th: ({ children }) => (
            <th className="border border-border bg-navyDeep px-2 py-1 text-left font-semibold">{children}</th>
          ),
          td: ({ children }) => <td className="border border-border px-2 py-1">{children}</td>,
          code: ({ className, children }) => {
            const text = String(children).replace(/\n$/, "");
            // react-markdown marca los bloques con `language-xxx`; sin clase y
            // sin salto de línea es código en línea.
            const language = /language-(\w+)/.exec(className ?? "")?.[1];
            if (!language && !text.includes("\n")) {
              return <code className="rounded bg-navyDeep px-1.5 py-0.5 font-mono text-[12.5px] text-gold">{text}</code>;
            }
            return <CodeBlock language={language ?? ""} code={text} />;
          },
          pre: ({ children }) => <>{children}</>,
        }}
      >
        {content}
      </ReactMarkdown>
    </div>
  );
});
```

### 96. `components/dashboard/ai/AgentsPane.tsx`
Agentes (roles preconfigurados) que abren un chat con su system prompt.
```tsx
"use client";

import { useMemo, useState } from "react";
import {
  Sparkles,
  Compass,
  Megaphone,
  Target,
  Clapperboard,
  Search,
  Users,
  ArrowRight,
  type LucideIcon,
} from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiAssistant } from "@/lib/ai/hub";

const ICONS: Record<string, LucideIcon> = {
  compass: Compass,
  megaphone: Megaphone,
  target: Target,
  clapperboard: Clapperboard,
  search: Search,
  users: Users,
  sparkles: Sparkles,
};

/**
 * Agentes ("Mis Ayudantes"): roles preconfigurados que abren un chat con su
 * propio system prompt. Antes vivían como tarjetas dentro del chat vacío.
 */
export function AgentsPane({
  assistants,
  onStart,
}: {
  assistants: AiAssistant[];
  onStart: (code: string) => void;
}) {
  const { t } = useT();
  const [query, setQuery] = useState("");

  const list = useMemo(() => {
    const q = query.trim().toLowerCase();
    if (!q) return assistants;
    return assistants.filter(
      (a) => a.name.toLowerCase().includes(q) || a.description.toLowerCase().includes(q)
    );
  }, [assistants, query]);

  return (
    <div className="mx-auto w-full max-w-5xl">
      <div className="mb-5 flex flex-wrap items-end justify-between gap-3">
        <div>
          <h2 className="font-display text-xl font-semibold text-cream">{t("aihub.tabAgents")}</h2>
          <p className="mt-1 text-sm text-muted">{t("aihub.agentsSubtitle")}</p>
        </div>
        <label className="flex items-center gap-2 rounded-lg border border-border bg-navyDeep px-3 py-2">
          <Search size={14} className="shrink-0 text-muted" />
          <input
            value={query}
            onChange={(e) => setQuery(e.target.value)}
            placeholder={t("aihub.searchAgent")}
            className="w-44 bg-transparent text-sm text-cream outline-none placeholder:text-muted"
          />
        </label>
      </div>

      {list.length === 0 ? (
        <p className="py-12 text-center text-sm text-muted">{t("aihub.noAgents")}</p>
      ) : (
        <div className="grid gap-3 sm:grid-cols-2 lg:grid-cols-3">
          {list.map((a) => {
            const Icon = ICONS[a.icon] ?? Sparkles;
            return (
              <button
                key={a.code}
                onClick={() => onStart(a.code)}
                className="group flex flex-col items-start gap-3 rounded-card border border-border bg-card p-4 text-left transition-colors hover:border-gold/50"
              >
                <span className="flex h-10 w-10 items-center justify-center rounded-lg bg-gold/15 text-gold">
                  <Icon size={18} />
                </span>
                <span className="min-w-0 flex-1">
                  <span className="block text-sm font-semibold text-cream">{a.name}</span>
                  <span className="mt-1 block text-[12px] leading-snug text-muted">{a.description}</span>
                </span>
                <span className="flex items-center gap-1 text-[11px] font-medium text-gold opacity-0 transition-opacity group-hover:opacity-100">
                  {t("aihub.startAgent")} <ArrowRight size={12} />
                </span>
              </button>
            );
          })}
        </div>
      )}
    </div>
  );
}
```

### 97. `components/dashboard/ai/ImagePane.tsx`
Generación de imágenes con panel de modelo, formato y calidad.
```tsx
"use client";

import { useEffect, useState } from "react";
import { Sparkles, Wand2, Download, Loader2, Globe2, Image as ImageIcon, SlidersHorizontal, Maximize, Gauge } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiModel } from "@/lib/ai/hub";
import type { AiCreation } from "@/lib/ai/creative";
import { toggleCreationPublicAction } from "@/app/(distribuidor)/ia/actions";
import { SidebarShell } from "./SidebarShell";
import { RightPanel } from "./RightPanel";
import { CollapsibleSection } from "./CollapsibleSection";
import { ModelSection, type ModelSelection } from "./ModelSection";

interface Result {
  url: string;
  prompt: string;
  id?: string;
  isPublic?: boolean;
}

const ASPECT_RATIOS = ["1:1", "16:9", "9:16", "4:5", "3:2"];
const QUALITIES = ["standard", "high"] as const;

export function ImagePane({
  models,
  initialCreations,
  balance,
  onBalanceChange,
  collapsed,
  onToggleCollapsed,
  recreate,
}: {
  models: AiModel[];
  initialCreations: AiCreation[];
  balance: number;
  onBalanceChange: (n: number) => void;
  collapsed: boolean;
  onToggleCollapsed: () => void;
  recreate?: { prompt: string; model: string; nonce: number } | null;
}) {
  const { t } = useT();
  const def = models.find((m) => m.isDefault) ?? models[0];
  const [selection, setSelection] = useState<ModelSelection>({ manual: def?.code ?? "" });
  const [rightCollapsed, setRightCollapsed] = useState(false);
  const [aspectRatio, setAspectRatio] = useState(ASPECT_RATIOS[0]);
  const [quality, setQuality] = useState<(typeof QUALITIES)[number]>("standard");
  const [prompt, setPrompt] = useState("");

  useEffect(() => {
    if (!recreate) return;
    setPrompt(recreate.prompt);
    if (recreate.model && models.some((m) => m.code === recreate.model)) setSelection({ manual: recreate.model });
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [recreate?.nonce]);
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [results, setResults] = useState<Result[]>([]);
  const [gallery, setGallery] = useState(initialCreations);

  const modelCode = "manual" in selection ? selection.manual : def?.code ?? "";
  const model = models.find((m) => m.code === modelCode);

  function resetAll() {
    setSelection({ manual: def?.code ?? "" });
    setAspectRatio(ASPECT_RATIOS[0]);
    setQuality("standard");
  }

  async function generate() {
    const p = prompt.trim();
    if (!p || busy) return;
    setError(null);
    setBusy(true);
    try {
      const res = await fetch("/api/ia/imagen", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ prompt: p, modelCode, aspectRatio, quality }),
      });
      const j = await res.json().catch(() => ({}));
      if (!res.ok) {
        setError(j?.error ?? t("img.error"));
        return;
      }
      setResults((r) => [{ url: j.url, prompt: p }, ...r]);
      if (typeof j.balance === "number") onBalanceChange(j.balance);
    } catch {
      setError(t("img.error"));
    } finally {
      setBusy(false);
    }
  }

  async function togglePublic(id: string, next: boolean) {
    setGallery((g) => g.map((c) => (c.id === id ? { ...c, is_public: next } : c)));
    await toggleCreationPublicAction(id, next);
  }

  const galleryPanel = (
    <div className="flex h-full flex-col">
      <p className="mb-2 px-1 text-xs font-semibold text-cream">{t("creative.gallery")}</p>
      <div className="flex-1 space-y-2 overflow-y-auto">
        {gallery.length === 0 && <p className="px-1 py-4 text-center text-xs text-muted">{t("creative.galleryEmpty")}</p>}
        {gallery.map((c) => (
          <div key={c.id} className="overflow-hidden rounded-lg border border-border bg-card">
            {/* eslint-disable-next-line @next/next/no-img-element */}
            <img src={c.url} alt={c.prompt} className="aspect-square w-full object-cover" />
            <div className="flex items-center justify-between p-1.5">
              <p className="truncate text-[10px] text-muted">{c.prompt}</p>
              <button
                onClick={() => togglePublic(c.id, !c.is_public)}
                title={t("aihub.makePublic")}
                className={c.is_public ? "text-gold" : "text-muted hover:text-cream"}
              >
                <Globe2 size={13} />
              </button>
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  return (
    <div className="flex h-full gap-4">
      <SidebarShell collapsed={collapsed} onToggle={onToggleCollapsed}>
        {galleryPanel}
      </SidebarShell>

      <div className="mx-auto flex min-w-0 max-w-2xl flex-1 flex-col overflow-y-auto">
        <div className="mb-3 flex flex-wrap items-center justify-between gap-3">
          <p className="label flex items-center gap-1.5">
            <ImageIcon size={12} className="text-gold" /> {t("creative.image")}
          </p>
          <span className="flex items-center gap-1 rounded-full border border-gold/40 bg-gold/10 px-3 py-1.5 text-sm font-semibold text-gold">
            <Sparkles size={13} /> {balance.toLocaleString()}
          </span>
        </div>

        <div className="rounded-card border border-border bg-card p-5">
          <label className="label">{t("img.prompt")}</label>
          <textarea
            value={prompt}
            onChange={(e) => setPrompt(e.target.value)}
            rows={3}
            placeholder={t("img.placeholder")}
            className="mt-2 w-full resize-none rounded-xl border border-border bg-navyDeep px-4 py-3 text-sm text-cream outline-none focus:border-gold"
          />
          <div className="mt-3 flex flex-wrap items-center justify-between gap-3">
            <span className="text-xs text-muted">
              {model?.name}
              {model?.tier ? ` · ${model.tier}` : ""} · {model?.creditCost ?? 0} {t("ia.creditsShort")}
            </span>
            <button
              onClick={generate}
              disabled={busy || !prompt.trim()}
              className="flex items-center gap-2 rounded-xl bg-gold px-5 py-2.5 text-sm font-semibold text-goldInk transition-opacity hover:bg-goldLight disabled:opacity-40"
            >
              {busy ? <Loader2 size={16} className="animate-spin" /> : <Wand2 size={16} />}
              {busy ? t("img.creating") : t("img.create")}
            </button>
          </div>
          {error && (
            <p className="mt-3 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{error}</p>
          )}
        </div>

        {results.length > 0 && (
          <div className="mt-4 grid grid-cols-1 gap-4 sm:grid-cols-2">
            {results.map((r, i) => (
              <div key={i} className="overflow-hidden rounded-card border border-border bg-card">
                {/* eslint-disable-next-line @next/next/no-img-element */}
                <img src={r.url} alt={r.prompt} className="aspect-square w-full object-cover" />
                <div className="flex items-center justify-between gap-2 p-3">
                  <p className="truncate text-xs text-muted">{r.prompt}</p>
                  <a href={r.url} target="_blank" rel="noreferrer" className="shrink-0 text-gold hover:text-goldLight" aria-label={t("img.download")}>
                    <Download size={16} />
                  </a>
                </div>
              </div>
            ))}
          </div>
        )}
      </div>

      <RightPanel collapsed={rightCollapsed} onToggle={() => setRightCollapsed((c) => !c)} onReset={resetAll}>
        <CollapsibleSection title={t("aihub.chooseModel")} icon={SlidersHorizontal}>
          <ModelSection models={models} selection={selection} onChange={setSelection} showAuto={false} />
        </CollapsibleSection>
        <CollapsibleSection title={t("aihub.format")} icon={Maximize}>
          <select
            value={aspectRatio}
            onChange={(e) => setAspectRatio(e.target.value)}
            className="w-full rounded-lg border border-border bg-navyDeep px-3 py-2 text-sm text-cream outline-none focus:border-gold"
          >
            {ASPECT_RATIOS.map((r) => (
              <option key={r} value={r}>
                {r}
              </option>
            ))}
          </select>
        </CollapsibleSection>
        <CollapsibleSection title={t("aihub.quality")} icon={Gauge}>
          <select
            value={quality}
            onChange={(e) => setQuality(e.target.value as (typeof QUALITIES)[number])}
            className="w-full rounded-lg border border-border bg-navyDeep px-3 py-2 text-sm text-cream outline-none focus:border-gold"
          >
            <option value="standard">{t("aihub.qualityStandard")}</option>
            <option value="high">{t("aihub.qualityHigh")}</option>
          </select>
        </CollapsibleSection>
      </RightPanel>
    </div>
  );
}
```

### 98. `components/dashboard/ai/EditorPane.tsx`
Editor de imagen (imagen de referencia + instrucciones).
```tsx
"use client";

import { useRef, useState } from "react";
import { Sparkles, Wand2, Download, Loader2, Upload, X, Pencil, SlidersHorizontal, Gauge } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiModel } from "@/lib/ai/hub";
import type { AiCreation } from "@/lib/ai/creative";
import { SidebarShell } from "./SidebarShell";
import { RightPanel } from "./RightPanel";
import { CollapsibleSection } from "./CollapsibleSection";
import { ModelSection, type ModelSelection } from "./ModelSection";

const QUALITIES = ["standard", "high"] as const;

export function EditorPane({
  models,
  initialCreations,
  balance,
  onBalanceChange,
  collapsed,
  onToggleCollapsed,
}: {
  models: AiModel[];
  initialCreations: AiCreation[];
  balance: number;
  onBalanceChange: (n: number) => void;
  collapsed: boolean;
  onToggleCollapsed: () => void;
}) {
  const { t } = useT();
  const def = models.find((m) => m.isDefault) ?? models[0];
  const [selection, setSelection] = useState<ModelSelection>({ manual: def?.code ?? "" });
  const [rightCollapsed, setRightCollapsed] = useState(false);
  const [quality, setQuality] = useState<(typeof QUALITIES)[number]>("standard");
  const [refUrl, setRefUrl] = useState<string | null>(null);
  const [refName, setRefName] = useState<string>("");
  const [uploading, setUploading] = useState(false);
  const [prompt, setPrompt] = useState("");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [result, setResult] = useState<string | null>(null);
  const inputRef = useRef<HTMLInputElement>(null);

  const modelCode = "manual" in selection ? selection.manual : def?.code ?? "";
  const model = models.find((m) => m.code === modelCode);

  function resetAll() {
    setSelection({ manual: def?.code ?? "" });
    setQuality("standard");
  }

  async function uploadRef(file: File) {
    setUploading(true);
    setError(null);
    try {
      const form = new FormData();
      form.append("file", file);
      const res = await fetch("/api/ia/upload", { method: "POST", body: form });
      const j = await res.json();
      if (!res.ok) {
        setError(j?.error ?? t("aihub.uploadError"));
        return;
      }
      setRefUrl(j.attachment.url);
      setRefName(j.attachment.name);
    } catch {
      setError(t("aihub.uploadError"));
    } finally {
      setUploading(false);
    }
  }

  async function generate() {
    const p = prompt.trim();
    if (!p || !refUrl || busy) return;
    setError(null);
    setBusy(true);
    try {
      const res = await fetch("/api/ia/imagen", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ prompt: p, modelCode, referenceImageUrl: refUrl, quality }),
      });
      const j = await res.json().catch(() => ({}));
      if (!res.ok) {
        setError(j?.error ?? t("img.error"));
        return;
      }
      setResult(j.url);
      if (typeof j.balance === "number") onBalanceChange(j.balance);
    } catch {
      setError(t("img.error"));
    } finally {
      setBusy(false);
    }
  }

  const galleryPanel = (
    <div className="flex h-full flex-col">
      <p className="mb-2 px-1 text-xs font-semibold text-cream">{t("creative.gallery")}</p>
      <div className="flex-1 space-y-2 overflow-y-auto">
        {initialCreations.length === 0 && <p className="px-1 py-4 text-center text-xs text-muted">{t("creative.galleryEmpty")}</p>}
        {initialCreations.map((c) => (
          <button
            key={c.id}
            onClick={() => {
              setRefUrl(c.url);
              setRefName(c.prompt.slice(0, 30));
            }}
            className="block w-full overflow-hidden rounded-lg border border-border bg-card"
          >
            {/* eslint-disable-next-line @next/next/no-img-element */}
            <img src={c.url} alt={c.prompt} className="aspect-square w-full object-cover" />
          </button>
        ))}
      </div>
    </div>
  );

  return (
    <div className="flex h-full gap-4">
      <SidebarShell collapsed={collapsed} onToggle={onToggleCollapsed}>
        {galleryPanel}
      </SidebarShell>

      <div className="mx-auto flex min-w-0 max-w-2xl flex-1 flex-col overflow-y-auto">
        <div className="mb-3 flex flex-wrap items-center justify-between gap-3">
          <p className="label flex items-center gap-1.5">
            <Pencil size={12} className="text-gold" /> {t("creative.editor")}
          </p>
          <span className="flex items-center gap-1 rounded-full border border-gold/40 bg-gold/10 px-3 py-1.5 text-sm font-semibold text-gold">
            <Sparkles size={13} /> {balance.toLocaleString()}
          </span>
        </div>

        <div className="rounded-card border border-border bg-card p-5">
          <input
            ref={inputRef}
            type="file"
            accept="image/*"
            className="hidden"
            onChange={(e) => {
              const f = e.target.files?.[0];
              if (f) uploadRef(f);
              e.target.value = "";
            }}
          />
          {refUrl ? (
            <div className="relative">
              {/* eslint-disable-next-line @next/next/no-img-element */}
              <img src={refUrl} alt={refName} className="max-h-72 w-full rounded-xl object-contain" />
              <button
                onClick={() => setRefUrl(null)}
                className="absolute right-2 top-2 flex h-7 w-7 items-center justify-center rounded-full bg-navyDeep/90 text-cream hover:text-danger"
                aria-label="Quitar"
              >
                <X size={14} />
              </button>
            </div>
          ) : (
            <button
              onClick={() => inputRef.current?.click()}
              disabled={uploading}
              className="flex w-full flex-col items-center justify-center gap-2 rounded-xl border border-dashed border-border py-10 text-muted hover:border-gold hover:text-cream"
            >
              {uploading ? <Loader2 size={22} className="animate-spin" /> : <Upload size={22} />}
              <span className="text-sm">{t("aihub.uploadImage")}</span>
            </button>
          )}

          <label className="label mt-4 block">{t("aihub.editInstructions")}</label>
          <textarea
            value={prompt}
            onChange={(e) => setPrompt(e.target.value)}
            rows={2}
            placeholder={t("aihub.editPlaceholder")}
            className="mt-2 w-full resize-none rounded-xl border border-border bg-navyDeep px-4 py-3 text-sm text-cream outline-none focus:border-gold"
          />
          <div className="mt-3 flex flex-wrap items-center justify-between gap-3">
            <span className="text-xs text-muted">
              {model?.name}
              {model?.tier ? ` · ${model.tier}` : ""} · {model?.creditCost ?? 0} {t("ia.creditsShort")}
            </span>
            <button
              onClick={generate}
              disabled={busy || !prompt.trim() || !refUrl}
              className="flex items-center gap-2 rounded-xl bg-gold px-5 py-2.5 text-sm font-semibold text-goldInk transition-opacity hover:bg-goldLight disabled:opacity-40"
            >
              {busy ? <Loader2 size={16} className="animate-spin" /> : <Wand2 size={16} />}
              {busy ? t("img.creating") : t("aihub.applyEdit")}
            </button>
          </div>
          {error && (
            <p className="mt-3 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{error}</p>
          )}
        </div>

        {result && (
          <div className="mt-4 overflow-hidden rounded-card border border-border bg-card">
            {/* eslint-disable-next-line @next/next/no-img-element */}
            <img src={result} alt={prompt} className="w-full object-contain" />
            <div className="flex items-center justify-end gap-2 p-3">
              <a href={result} target="_blank" rel="noreferrer" className="text-gold hover:text-goldLight" aria-label={t("img.download")}>
                <Download size={16} />
              </a>
            </div>
          </div>
        )}
      </div>

      <RightPanel collapsed={rightCollapsed} onToggle={() => setRightCollapsed((c) => !c)} onReset={resetAll}>
        <CollapsibleSection title={t("aihub.chooseModel")} icon={SlidersHorizontal}>
          <ModelSection models={models} selection={selection} onChange={setSelection} showAuto={false} />
        </CollapsibleSection>
        <CollapsibleSection title={t("aihub.quality")} icon={Gauge}>
          <select
            value={quality}
            onChange={(e) => setQuality(e.target.value as (typeof QUALITIES)[number])}
            className="w-full rounded-lg border border-border bg-navyDeep px-3 py-2 text-sm text-cream outline-none focus:border-gold"
          >
            <option value="standard">{t("aihub.qualityStandard")}</option>
            <option value="high">{t("aihub.qualityHigh")}</option>
          </select>
        </CollapsibleSection>
      </RightPanel>
    </div>
  );
}
```

### 99. `components/dashboard/ai/AudioPane.tsx`
Generación de audio: música y voz.
```tsx
"use client";

import { useState } from "react";
import { Sparkles, Wand2, Loader2, Music, SlidersHorizontal, Mic2, Download } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiModel } from "@/lib/ai/hub";
import { RightPanel } from "./RightPanel";
import { CollapsibleSection } from "./CollapsibleSection";
import { ModelSection, type ModelSelection } from "./ModelSection";

/** Voces de la API de audio de OpenAI (se ignoran en los modelos de música). */
const VOICES = ["alloy", "echo", "fable", "onyx", "nova", "shimmer"];

interface Result {
  url: string;
  prompt: string;
}

export function AudioPane({
  models,
  balance,
  onBalanceChange,
}: {
  models: AiModel[];
  balance: number;
  onBalanceChange: (n: number) => void;
}) {
  const { t } = useT();
  const def = models.find((m) => m.isDefault) ?? models[0];
  const [selection, setSelection] = useState<ModelSelection>({ manual: def?.code ?? "" });
  const [rightCollapsed, setRightCollapsed] = useState(false);
  const [voice, setVoice] = useState(VOICES[0]);
  const [prompt, setPrompt] = useState("");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [results, setResults] = useState<Result[]>([]);

  const modelCode = "manual" in selection ? selection.manual : def?.code ?? "";
  const model = models.find((m) => m.code === modelCode);

  function resetAll() {
    setSelection({ manual: def?.code ?? "" });
    setVoice(VOICES[0]);
  }

  async function generate() {
    const p = prompt.trim();
    if (!p || busy) return;
    setError(null);
    setBusy(true);
    try {
      const res = await fetch("/api/ia/audio", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ prompt: p, modelCode, voice }),
      });
      const j = await res.json().catch(() => ({}));
      if (!res.ok) {
        setError(j?.error ?? t("audio.error"));
        return;
      }
      setResults((r) => [{ url: j.url, prompt: p }, ...r]);
      if (typeof j.balance === "number") onBalanceChange(j.balance);
    } catch {
      setError(t("audio.error"));
    } finally {
      setBusy(false);
    }
  }

  if (models.length === 0) {
    return <p className="py-16 text-center text-sm text-muted">{t("aihub.noModels")}</p>;
  }

  return (
    <div className="flex h-full gap-4">
      <div className="mx-auto flex min-w-0 max-w-2xl flex-1 flex-col overflow-y-auto">
        <div className="mb-3 flex flex-wrap items-center justify-between gap-3">
          <p className="label flex items-center gap-1.5">
            <Music size={12} className="text-gold" /> {t("aihub.tabAudio")}
          </p>
          <span className="flex items-center gap-1 rounded-full border border-gold/40 bg-gold/10 px-3 py-1.5 text-sm font-semibold text-gold">
            <Sparkles size={13} /> {balance.toLocaleString()}
          </span>
        </div>

        <div className="rounded-card border border-border bg-card p-5">
          <label className="label">{t("audio.prompt")}</label>
          <textarea
            value={prompt}
            onChange={(e) => setPrompt(e.target.value)}
            rows={3}
            placeholder={t("audio.placeholder")}
            className="mt-2 w-full resize-none rounded-xl border border-border bg-navyDeep px-4 py-3 text-sm text-cream outline-none focus:border-gold"
          />
          <div className="mt-3 flex flex-wrap items-center justify-between gap-3">
            <span className="text-xs text-muted">
              {model?.name}
              {model?.tier ? ` · ${model.tier}` : ""} · {model?.creditCost ?? 0} {t("ia.creditsShort")}
            </span>
            <button
              onClick={generate}
              disabled={busy || !prompt.trim()}
              className="flex items-center gap-2 rounded-xl bg-gold px-5 py-2.5 text-sm font-semibold text-goldInk transition-opacity hover:bg-goldLight disabled:opacity-40"
            >
              {busy ? <Loader2 size={16} className="animate-spin" /> : <Wand2 size={16} />}
              {busy ? t("audio.creating") : t("audio.create")}
            </button>
          </div>
          {error && (
            <p className="mt-3 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{error}</p>
          )}
        </div>

        {results.length > 0 && (
          <div className="mt-4 space-y-3">
            {results.map((r, i) => (
              <div key={i} className="rounded-card border border-border bg-card p-4">
                <p className="mb-2 truncate text-xs text-muted">{r.prompt}</p>
                <div className="flex items-center gap-3">
                  <audio controls src={r.url} className="min-w-0 flex-1" />
                  <a
                    href={r.url}
                    target="_blank"
                    rel="noreferrer"
                    className="shrink-0 text-gold hover:text-goldLight"
                    aria-label={t("img.download")}
                  >
                    <Download size={16} />
                  </a>
                </div>
              </div>
            ))}
          </div>
        )}
      </div>

      <RightPanel collapsed={rightCollapsed} onToggle={() => setRightCollapsed((c) => !c)} onReset={resetAll}>
        <CollapsibleSection title={t("aihub.chooseModel")} icon={SlidersHorizontal}>
          <ModelSection models={models} selection={selection} onChange={setSelection} showAuto={false} />
        </CollapsibleSection>
        <CollapsibleSection title={t("audio.voice")} icon={Mic2}>
          <select
            value={voice}
            onChange={(e) => setVoice(e.target.value)}
            className="w-full rounded-lg border border-border bg-navyDeep px-3 py-2 text-sm text-cream outline-none focus:border-gold"
          >
            {VOICES.map((v) => (
              <option key={v} value={v}>
                {v}
              </option>
            ))}
          </select>
          <p className="mt-2 text-[11px] leading-snug text-muted">{t("audio.voiceHint")}</p>
        </CollapsibleSection>
      </RightPanel>
    </div>
  );
}
```

### 100. `components/dashboard/ai/SuperPrompterPane.tsx`
Convierte una idea corta en un prompt detallado.
```tsx
"use client";

import { useState } from "react";
import { Wand2, Loader2, Copy, Check, ImagePlus, Video, Sparkles } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";

export function SuperPrompterPane({
  balance,
  onBalanceChange,
  onUse,
}: {
  balance: number;
  onBalanceChange: (n: number) => void;
  onUse: (prompt: string, target: "imagenes" | "video") => void;
}) {
  const { t } = useT();
  const [idea, setIdea] = useState("");
  const [target, setTarget] = useState<"image" | "video">("image");
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [result, setResult] = useState<string | null>(null);
  const [copied, setCopied] = useState(false);

  async function generate() {
    const i = idea.trim();
    if (!i || busy) return;
    setError(null);
    setBusy(true);
    setResult(null);
    try {
      const res = await fetch("/api/ia/superprompt", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ idea: i, target }),
      });
      const j = await res.json().catch(() => ({}));
      if (!res.ok) {
        setError(j?.error ?? t("ia.error"));
        return;
      }
      setResult(j.prompt);
      if (typeof j.balance === "number") onBalanceChange(j.balance);
    } catch {
      setError(t("ia.error"));
    } finally {
      setBusy(false);
    }
  }

  function copy() {
    if (!result) return;
    navigator.clipboard?.writeText(result).then(() => {
      setCopied(true);
      setTimeout(() => setCopied(false), 1500);
    });
  }

  return (
    <div className="mx-auto max-w-2xl">
      <div className="mb-4 flex flex-wrap items-center justify-between gap-3">
        <p className="label flex items-center gap-1.5">
          <Wand2 size={12} className="text-gold" /> {t("aihub.tabSuperPrompt")}
        </p>
        <span className="flex items-center gap-1 rounded-full border border-gold/40 bg-gold/10 px-3 py-1.5 text-sm font-semibold text-gold">
          <Sparkles size={13} /> {balance.toLocaleString()}
        </span>
      </div>
      <p className="mb-4 text-sm text-muted">{t("aihub.superPromptDesc")}</p>

      <div className="rounded-card border border-border bg-card p-5">
        <div className="mb-3 inline-flex rounded-lg border border-border p-0.5">
          <button
            onClick={() => setTarget("image")}
            className={`flex items-center gap-1.5 rounded-md px-3 py-1.5 text-xs font-medium transition-colors ${
              target === "image" ? "bg-gold text-goldInk" : "text-muted hover:text-cream"
            }`}
          >
            <ImagePlus size={13} /> {t("creative.image")}
          </button>
          <button
            onClick={() => setTarget("video")}
            className={`flex items-center gap-1.5 rounded-md px-3 py-1.5 text-xs font-medium transition-colors ${
              target === "video" ? "bg-gold text-goldInk" : "text-muted hover:text-cream"
            }`}
          >
            <Video size={13} /> {t("aihub.tabVideo")}
          </button>
        </div>

        <textarea
          value={idea}
          onChange={(e) => setIdea(e.target.value)}
          rows={3}
          placeholder={t("aihub.superPromptPlaceholder")}
          className="w-full resize-none rounded-xl border border-border bg-navyDeep px-4 py-3 text-sm text-cream outline-none focus:border-gold"
        />
        <div className="mt-3 flex justify-end">
          <button
            onClick={generate}
            disabled={busy || !idea.trim()}
            className="flex items-center gap-2 rounded-xl bg-gold px-5 py-2.5 text-sm font-semibold text-goldInk transition-opacity hover:bg-goldLight disabled:opacity-40"
          >
            {busy ? <Loader2 size={16} className="animate-spin" /> : <Wand2 size={16} />}
            {busy ? t("img.creating") : t("aihub.enhance")}
          </button>
        </div>
        {error && (
          <p className="mt-3 rounded-md border border-danger/30 bg-danger/10 px-3 py-2 text-sm text-danger">{error}</p>
        )}
      </div>

      {result && (
        <div className="mt-4 rounded-card border border-gold/40 bg-card p-5">
          <div className="flex items-center justify-between">
            <p className="label">{t("aihub.enhancedPrompt")}</p>
            <button onClick={copy} className="flex items-center gap-1 text-xs text-gold hover:text-goldLight">
              {copied ? <Check size={13} /> : <Copy size={13} />} {copied ? t("aihub.copied") : t("aihub.copy")}
            </button>
          </div>
          <p className="mt-2 whitespace-pre-wrap rounded-lg bg-navyDeep px-3 py-2.5 text-sm text-cream">{result}</p>
          <button
            onClick={() => onUse(result, target === "video" ? "video" : "imagenes")}
            className="mt-3 flex w-full items-center justify-center gap-2 rounded-xl bg-gold px-4 py-2.5 text-sm font-semibold text-goldInk hover:bg-goldLight"
          >
            {target === "video" ? <Video size={15} /> : <ImagePlus size={15} />}
            {target === "video" ? t("aihub.useInVideo") : t("aihub.useInImages")}
          </button>
        </div>
      )}
    </div>
  );
}
```

### 101. `components/dashboard/ai/TrendsPane.tsx`
Galería pública de Tendencias con 'Recrear'.
```tsx
"use client";

import { useState } from "react";
import { TrendingUp, X, RefreshCcw, Eye } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { PublicCreation } from "@/lib/ai/creative";
import { bumpCreationViewAction } from "@/app/(distribuidor)/ia/actions";

export function TrendsPane({
  creations,
  onRecreate,
}: {
  creations: PublicCreation[];
  onRecreate: (prompt: string, modelCode: string) => void;
}) {
  const { t } = useT();
  const [selected, setSelected] = useState<PublicCreation | null>(null);

  function open(c: PublicCreation) {
    setSelected(c);
    bumpCreationViewAction(c.id);
  }

  return (
    <div className="flex h-full gap-4">
      <div className="flex-1 overflow-y-auto">
        <div className="mb-4">
          <p className="label flex items-center gap-1.5">
            <TrendingUp size={12} className="text-gold" /> {t("aihub.tabTrends")}
          </p>
          <p className="mt-1 text-sm text-muted">{t("aihub.trendsSubtitle")}</p>
        </div>

        {creations.length === 0 ? (
          <div className="flex flex-col items-center justify-center rounded-card border border-border bg-navyDeep/40 py-16 text-center">
            <TrendingUp size={26} className="text-gold" />
            <p className="mt-2 max-w-xs text-sm text-muted">{t("aihub.trendsEmpty")}</p>
          </div>
        ) : (
          <div className="grid grid-cols-2 gap-3 sm:grid-cols-3 lg:grid-cols-4">
            {creations.map((c) => (
              <button
                key={c.id}
                onClick={() => open(c)}
                className="group relative overflow-hidden rounded-card border border-border text-left"
              >
                {/* eslint-disable-next-line @next/next/no-img-element */}
                <img src={c.url} alt={c.prompt} className="aspect-square w-full object-cover transition-transform group-hover:scale-105" />
                <div className="absolute inset-x-0 bottom-0 bg-gradient-to-t from-navyDeep/90 to-transparent p-2">
                  <p className="truncate text-[10px] text-cream">{c.authorName}</p>
                </div>
              </button>
            ))}
          </div>
        )}
      </div>

      {/* Panel de detalle (estilo syntx: prompt, modelo, costo, Recrear) */}
      {selected && (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/60 p-4" onClick={() => setSelected(null)}>
          <div
            className="flex max-h-[85vh] w-full max-w-3xl overflow-hidden rounded-card border border-gold/40 bg-card"
            onClick={(e) => e.stopPropagation()}
          >
            {/* eslint-disable-next-line @next/next/no-img-element */}
            <img src={selected.url} alt={selected.prompt} className="hidden w-1/2 object-cover sm:block" />
            <div className="flex w-full flex-col overflow-y-auto p-5 sm:w-1/2">
              <div className="mb-3 flex items-center justify-between">
                <span className="rounded-full bg-gold/15 px-2 py-0.5 text-[10px] font-semibold uppercase text-gold">
                  {t("aihub.viral")}
                </span>
                <button onClick={() => setSelected(null)} className="text-muted hover:text-cream" aria-label="Cerrar">
                  <X size={18} />
                </button>
              </div>
              <p className="mb-1 text-xs text-muted">{selected.authorName}</p>
              <p className="mb-4 whitespace-pre-wrap rounded-lg bg-navyDeep px-3 py-2.5 text-sm text-cream">{selected.prompt}</p>
              <div className="mb-4 grid grid-cols-2 gap-2 text-xs">
                <Stat label={t("aihub.model")} value={selected.model_code ?? "—"} />
                <Stat label={t("img.costLabel")} value={`${selected.credits_spent} ${t("ia.creditsShort")}`} />
                <Stat label={t("aihub.views")} value={String(selected.views)} icon={<Eye size={11} />} />
                <Stat label={t("aihub.date")} value={new Date(selected.created_at).toLocaleDateString()} />
              </div>
              <button
                onClick={() => {
                  onRecreate(selected.prompt, selected.model_code ?? "");
                  setSelected(null);
                }}
                className="mt-auto flex items-center justify-center gap-2 rounded-xl bg-gold px-4 py-2.5 text-sm font-semibold text-goldInk hover:bg-goldLight"
              >
                <RefreshCcw size={15} /> {t("aihub.recreate")}
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

function Stat({ label, value, icon }: { label: string; value: string; icon?: React.ReactNode }) {
  return (
    <div className="rounded-lg bg-navyDeep p-2">
      <p className="flex items-center gap-1 text-[9px] uppercase tracking-label text-muted">
        {icon} {label}
      </p>
      <p className="mt-0.5 truncate text-xs font-semibold text-gold" title={value}>
        {value}
      </p>
    </div>
  );
}
```

### 102. `components/dashboard/ai/ProjectsGrid.tsx`
Mis Proyectos: tarjetas con detalle al pasar el cursor y vista a pantalla completa.
```tsx
"use client";

import { useMemo, useState } from "react";
import {
  Globe2,
  Download,
  Image as ImageIcon,
  Music,
  Video,
  X,
  Maximize2,
  Upload,
  Sparkles,
  BarChart3,
} from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { AiCreation } from "@/lib/ai/creative";
import { toggleCreationPublicAction } from "@/app/(distribuidor)/ia/actions";

type Filter = "todo" | "image" | "audio" | "video";

const FILTERS: Filter[] = ["todo", "image", "audio", "video"];

/** 'edit' es una imagen a ojos de la galería. */
function kindOf(c: AiCreation): "image" | "audio" | "video" {
  const k = c.kind === "edit" ? "image" : c.kind || "image";
  return k === "audio" || k === "video" ? k : "image";
}

/**
 * Mis Proyectos: todo lo que el distribuidor ha generado. Las tarjetas revelan
 * el detalle al pasar el cursor y, al abrirlas, el proyecto se sobreexpone
 * sobre la pantalla con su ficha completa a la derecha.
 */
export function ProjectsGrid({ creations }: { creations: AiCreation[] }) {
  const { t } = useT();
  const [items, setItems] = useState(creations);
  const [filter, setFilter] = useState<Filter>("todo");
  const [detail, setDetail] = useState<AiCreation | null>(null);

  const list = useMemo(
    () => (filter === "todo" ? items : items.filter((c) => kindOf(c) === filter)),
    [items, filter]
  );

  async function togglePublic(id: string, next: boolean) {
    setItems((g) => g.map((c) => (c.id === id ? { ...c, is_public: next } : c)));
    setDetail((d) => (d && d.id === id ? { ...d, is_public: next } : d));
    await toggleCreationPublicAction(id, next);
  }

  return (
    <>
      <div className="mb-5 flex flex-wrap items-center justify-between gap-3">
        <div className="flex gap-1 rounded-lg bg-navyDeep p-1">
          {FILTERS.map((f) => (
            <button
              key={f}
              onClick={() => setFilter(f)}
              className={`rounded-md px-3 py-1 text-sm transition-colors ${
                filter === f ? "bg-gold text-goldInk" : "text-muted hover:text-cream"
              }`}
            >
              {t(`aihub.filter.${f}`)}
            </button>
          ))}
        </div>
        <span className="text-xs text-muted">
          {list.length} {t("projects.count")}
        </span>
      </div>

      {list.length === 0 ? (
        <p className="py-20 text-center text-sm text-muted">{t("creative.galleryEmpty")}</p>
      ) : (
        <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-3">
          {list.map((c) => (
            <ProjectCard key={c.id} creation={c} onOpen={() => setDetail(c)} />
          ))}
        </div>
      )}

      {detail && (
        <DetailOverlay creation={detail} onClose={() => setDetail(null)} onTogglePublic={togglePublic} />
      )}
    </>
  );
}

/** Tarjeta con revelado al pasar el cursor (título, prompt y etiquetas). */
function ProjectCard({ creation, onOpen }: { creation: AiCreation; onOpen: () => void }) {
  const { t } = useT();
  const kind = kindOf(creation);

  return (
    <button
      onClick={onOpen}
      className="group relative block w-full overflow-hidden rounded-card border border-border bg-card text-left transition-colors hover:border-gold/60"
    >
      <Media creation={creation} className="aspect-[4/3] w-full object-cover" />

      {/* Capa de detalle: aparece al pasar el cursor, como en la referencia. */}
      <div className="pointer-events-none absolute inset-0 flex flex-col justify-end bg-gradient-to-t from-black/90 via-black/40 to-transparent p-3 opacity-0 transition-opacity duration-200 group-hover:opacity-100">
        <p className="truncate text-sm font-semibold text-white">{creation.prompt.slice(0, 60)}</p>
        <p className="mt-0.5 line-clamp-2 text-[11px] leading-snug text-white/70">{creation.prompt}</p>
        <div className="mt-2 flex items-center justify-between gap-2">
          <span className="flex items-center gap-1.5">
            <Chip icon={BarChart3} label={(creation.model_code ?? "—").split("/").pop() ?? "—"} />
            <Chip icon={kind === "audio" ? Music : kind === "video" ? Video : ImageIcon} label={t(`aihub.filter.${kind}`)} />
          </span>
          <span className="flex items-center gap-1 rounded-md border border-white/25 px-2 py-1 text-[10px] font-medium text-white">
            <Maximize2 size={10} /> {t("projects.more")}
          </span>
        </div>
      </div>
    </button>
  );
}

function Chip({ icon: Icon, label }: { icon: typeof ImageIcon; label: string }) {
  return (
    <span className="flex items-center gap-1 rounded-md border border-white/25 bg-black/40 px-2 py-1 text-[10px] font-medium text-white">
      <Icon size={10} /> {label}
    </span>
  );
}

/** Vista sobreexpuesta: el medio a la izquierda y la ficha del proyecto a la derecha. */
function DetailOverlay({
  creation,
  onClose,
  onTogglePublic,
}: {
  creation: AiCreation;
  onClose: () => void;
  onTogglePublic: (id: string, next: boolean) => void;
}) {
  const { t } = useT();
  const kind = kindOf(creation);

  const params: { label: string; value: string }[] = [
    { label: t("projects.ai"), value: (creation.model_code ?? "—").split("/")[0] },
    { label: t("aihub.model"), value: (creation.model_code ?? "—").split("/").pop() ?? "—" },
    { label: t("projects.type"), value: t(`aihub.filter.${kind}`) },
    { label: t("projects.cost"), value: `${creation.credits_spent} ${t("ia.creditsShort")}` },
    { label: t("aihub.views"), value: String(creation.views) },
    { label: t("aihub.date"), value: new Date(creation.created_at).toLocaleDateString() },
  ];

  return (
    <div className="fixed inset-0 z-50 flex bg-black/85 backdrop-blur-sm" onClick={onClose}>
      <div className="flex min-h-0 w-full flex-col lg:flex-row" onClick={(e) => e.stopPropagation()}>
        {/* Medio */}
        <div className="flex min-h-0 flex-1 items-center justify-center p-4 lg:p-8">
          <Media creation={creation} className="max-h-[80vh] max-w-full rounded-lg object-contain" controls />
        </div>

        {/* Ficha */}
        <aside className="flex w-full shrink-0 flex-col overflow-y-auto border-l border-border bg-navyDeep p-5 lg:w-[420px]">
          <div className="flex items-start justify-between gap-3">
            <span className="text-[11px] font-bold uppercase tracking-wide text-gold">
              {creation.is_public ? t("aihub.public") : t("projects.private")}
            </span>
            <button onClick={onClose} className="text-muted hover:text-cream" aria-label="Cerrar">
              <X size={18} />
            </button>
          </div>

          <h2 className="mt-2 font-display text-2xl font-semibold text-cream">
            {creation.prompt.slice(0, 48)}
          </h2>

          <div className="mt-4 max-h-44 overflow-y-auto rounded-lg border border-border bg-card p-3">
            <p className="whitespace-pre-wrap text-[13px] leading-relaxed text-cream/90">{creation.prompt}</p>
          </div>
          <p className="mt-1 text-right text-[11px] text-muted">{creation.prompt.length} {t("projects.chars")}</p>

          <div className="mt-5">
            <p className="text-sm font-semibold text-cream">{t("projects.referenceFiles")}</p>
            <div className="mt-2 flex items-center justify-center gap-2 rounded-lg border border-dashed border-border bg-card/50 px-4 py-5 text-xs text-muted">
              <Upload size={14} /> {t("projects.dropFile")}
            </div>
          </div>

          <div className="mt-5">
            <p className="text-sm font-semibold text-cream">{t("projects.params")}</p>
            <div className="mt-2 grid grid-cols-2 gap-x-4 gap-y-3">
              {params.map((p) => (
                <div key={p.label}>
                  <p className="text-[11px] text-muted">{p.label}</p>
                  <p className="truncate text-sm font-medium text-cream" title={p.value}>
                    {p.value}
                  </p>
                </div>
              ))}
            </div>
          </div>

          <div className="mt-auto flex gap-2 pt-6">
            <a
              href={creation.url}
              target="_blank"
              rel="noreferrer"
              className="flex items-center justify-center gap-1.5 rounded-lg border border-border px-3 py-2.5 text-xs font-medium text-cream hover:border-gold"
            >
              <Download size={14} />
            </a>
            <button
              onClick={() => onTogglePublic(creation.id, !creation.is_public)}
              className={`flex flex-1 items-center justify-center gap-1.5 rounded-lg border px-3 py-2.5 text-xs font-medium ${
                creation.is_public
                  ? "border-gold/50 bg-gold/10 text-gold"
                  : "border-border text-cream hover:border-gold"
              }`}
            >
              <Globe2 size={14} /> {creation.is_public ? t("aihub.public") : t("aihub.makePublic")}
            </button>
            <a
              href="/ia"
              className="flex flex-1 items-center justify-center gap-1.5 rounded-lg bg-gold px-4 py-2.5 text-xs font-semibold text-goldInk hover:bg-goldLight"
            >
              <Sparkles size={14} /> {t("aihub.recreate")}
            </a>
          </div>
        </aside>
      </div>
    </div>
  );
}

function Media({
  creation,
  className,
  controls = false,
}: {
  creation: AiCreation;
  className?: string;
  controls?: boolean;
}) {
  const kind = kindOf(creation);

  if (kind === "video") {
    return <video src={creation.url} className={className} controls={controls} muted playsInline />;
  }
  if (kind === "audio") {
    return (
      <div className={`flex flex-col items-center justify-center gap-3 bg-navyDeep ${className ?? ""}`}>
        <Music size={36} className="text-gold" />
        {controls && <audio controls src={creation.url} className="w-full max-w-sm" />}
      </div>
    );
  }
  // eslint-disable-next-line @next/next/no-img-element
  return <img src={creation.url} alt={creation.prompt} className={className} />;
}
```

### 103. `components/dashboard/ai/ComingSoonPane.tsx`
Estado honesto para funciones sin proveedor conectado (Video).
```tsx
"use client";

import type { LucideIcon } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";

/**
 * Estado honesto para secciones que necesitan un proveedor externo aún no
 * conectado (video/audio requieren fal.ai, Replicate o ElevenLabs — ninguno
 * está integrado todavía). No simula generación: explica qué falta.
 */
export function ComingSoonPane({
  icon: Icon,
  titleKey,
  descKey,
}: {
  icon: LucideIcon;
  titleKey: string;
  descKey: string;
}) {
  const { t } = useT();
  return (
    <div className="mx-auto flex max-w-md flex-col items-center justify-center py-20 text-center">
      <span className="flex h-16 w-16 items-center justify-center rounded-full border-2 border-gold/40 bg-navyDeep">
        <Icon size={28} className="text-gold" />
      </span>
      <h2 className="mt-4 font-display text-xl font-bold text-cream">{t(titleKey)}</h2>
      <p className="mt-2 text-sm text-muted">{t(descKey)}</p>
      <span className="mt-4 rounded-full bg-gold/15 px-3 py-1 text-xs font-semibold text-gold">
        {t("creative.soon")}
      </span>
    </div>
  );
}
```

### 104. `components/dashboard/ai/AttachButton.tsx`
Adjuntar archivos según lo que acepta el modelo.
```tsx
"use client";

import { useRef, useState } from "react";
import { Paperclip, Loader2 } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";
import type { Attachment } from "@/lib/ai/attachments";

/** Botón de adjuntar: sube el archivo y habilita solo lo que el modelo activo acepta. */
export function AttachButton({
  caps,
  onAdd,
  onError,
  disabled,
}: {
  caps: { accepts_image: boolean; accepts_pdf: boolean; accepts_audio: boolean };
  onAdd: (a: Attachment) => void;
  onError: (msg: string) => void;
  disabled?: boolean;
}) {
  const { t } = useT();
  const inputRef = useRef<HTMLInputElement>(null);
  const [busy, setBusy] = useState(false);

  const anyCap = caps.accepts_image || caps.accepts_pdf || caps.accepts_audio;
  const accept = [
    caps.accepts_image ? "image/*" : "",
    caps.accepts_pdf ? "application/pdf" : "",
    caps.accepts_audio ? "audio/*,text/plain" : "",
  ]
    .filter(Boolean)
    .join(",");

  async function handleFile(file: File) {
    setBusy(true);
    try {
      const form = new FormData();
      form.append("file", file);
      const res = await fetch("/api/ia/upload", { method: "POST", body: form });
      const j = await res.json();
      if (!res.ok) {
        onError(j?.error ?? t("aihub.uploadError"));
        return;
      }
      onAdd(j.attachment as Attachment);
    } catch {
      onError(t("aihub.uploadError"));
    } finally {
      setBusy(false);
    }
  }

  return (
    <>
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        className="hidden"
        onChange={(e) => {
          const f = e.target.files?.[0];
          if (f) handleFile(f);
          e.target.value = "";
        }}
      />
      <button
        type="button"
        onClick={() => inputRef.current?.click()}
        disabled={disabled || busy || !anyCap}
        title={anyCap ? t("aihub.attach") : t("aihub.noAttachSupport")}
        className="flex h-11 w-11 shrink-0 items-center justify-center rounded-xl border border-border text-muted transition-colors hover:border-gold hover:text-cream disabled:opacity-30"
      >
        {busy ? <Loader2 size={16} className="animate-spin" /> : <Paperclip size={16} />}
      </button>
    </>
  );
}
```

### 105. `components/dashboard/ai/DictateButton.tsx`
Dictado por voz con la Web Speech API del navegador.
```tsx
"use client";

import { useEffect, useRef, useState } from "react";
import { Mic, MicOff } from "lucide-react";
import { useT } from "@/components/i18n/LanguageProvider";

// Tipado mínimo de la Web Speech API (no está en el lib.dom.d.ts estándar).
interface SpeechRecognitionResultLike {
  transcript: string;
}
interface SpeechRecognitionEventLike extends Event {
  results: { [i: number]: { [j: number]: SpeechRecognitionResultLike }; length: number };
}
interface SpeechRecognitionLike extends EventTarget {
  lang: string;
  continuous: boolean;
  interimResults: boolean;
  start(): void;
  stop(): void;
  onresult: ((e: SpeechRecognitionEventLike) => void) | null;
  onend: (() => void) | null;
  onerror: (() => void) | null;
}

/**
 * Dictado por voz: transcribe a texto con la API nativa del navegador
 * (gratis, sin créditos, funciona con micro de compu o celular). No requiere
 * que el modelo activo acepte audio — es solo una forma de escribir.
 */
export function DictateButton({ onText, locale = "es-ES" }: { onText: (t: string) => void; locale?: string }) {
  const { t } = useT();
  const [supported, setSupported] = useState(true);
  const [listening, setListening] = useState(false);
  const recRef = useRef<SpeechRecognitionLike | null>(null);

  useEffect(() => {
    const Ctor =
      (window as unknown as { SpeechRecognition?: new () => SpeechRecognitionLike }).SpeechRecognition ??
      (window as unknown as { webkitSpeechRecognition?: new () => SpeechRecognitionLike }).webkitSpeechRecognition;
    if (!Ctor) {
      setSupported(false);
      return;
    }
    const rec = new Ctor();
    rec.lang = locale;
    rec.continuous = false;
    rec.interimResults = false;
    rec.onresult = (e) => {
      let text = "";
      for (let i = 0; i < e.results.length; i++) text += e.results[i][0].transcript;
      onText(text.trim());
    };
    rec.onend = () => setListening(false);
    rec.onerror = () => setListening(false);
    recRef.current = rec;
  }, [locale, onText]);

  if (!supported) return null;

  function toggle() {
    if (!recRef.current) return;
    if (listening) {
      recRef.current.stop();
      setListening(false);
    } else {
      recRef.current.start();
      setListening(true);
    }
  }

  return (
    <button
      type="button"
      onClick={toggle}
      title={listening ? t("aihub.stopDictate") : t("aihub.dictate")}
      className={`flex h-11 w-11 shrink-0 items-center justify-center rounded-xl border transition-colors ${
        listening
          ? "animate-pulse border-danger bg-danger/10 text-danger"
          : "border-border text-muted hover:border-gold hover:text-cream"
      }`}
    >
      {listening ? <MicOff size={16} /> : <Mic size={16} />}
    </button>
  );
}
```


---

# PARTE 11 — HUB: SERVIDOR (RUTAS API Y LIBRERÍAS)
Rutas que llaman a OpenRouter y la lógica de créditos y enrutamiento.

### 106. `app/api/ia/chat/route.ts`
Chat en streaming: resuelve modelo y costo (Auto Max o manual), filtra adjuntos por capacidad, cobra, llama al proveedor, reembolsa si falla, guarda el turno y registra el uso.
```ts
/**
 * Ruta-proxy del Hub Triada — COSTO FIJO + ROUTER + ADJUNTOS.
 *
 * Dos modos:
 *  - Sin `modelCode`: decide Triada Auto Max según lo que se pidió; el costo y
 *    el modelo salen de `ia_tasks_config`.
 *  - `modelCode` presente: el usuario eligió un modelo del catálogo
 *    (`ai_models`); se cobra el `credit_cost` propio de ESE modelo.
 *
 * `variant` separa las secciones Chat y Código (cada una con su familia de
 * modelos y su system prompt).
 *
 * Los adjuntos se filtran por lo que el modelo realmente acepta antes de
 * construir el mensaje multimodal. La OPENROUTER_API_KEY vive solo en el servidor.
 */
import { createClient } from "@/lib/supabase/server";
import { TRIADA_SYSTEM_PROMPT, CODE_SYSTEM_PROMPT } from "@/lib/ai/triada-context";
import { getAssistantPrompt, ensureConversation, persistTurn, getModelByCode } from "@/lib/ai/hub";
import { resolveTextTask, getTaskConfig, type TextVariant } from "@/lib/ai/router";
import { spendCredits, refundCredits } from "@/lib/ai/credits";
import { recordUsage } from "@/lib/ai/activity";
import { filterByCapability, buildMultimodalContent, type Attachment } from "@/lib/ai/attachments";

export const dynamic = "force-dynamic";

function json(obj: unknown, status: number): Response {
  return new Response(JSON.stringify(obj), { status, headers: { "Content-Type": "application/json" } });
}

interface ChatMessage {
  role: "user" | "assistant" | "system";
  content: string;
  attachments?: Attachment[];
}

export async function POST(req: Request): Promise<Response> {
  const supabase = createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return json({ error: "No autorizado." }, 401);

  let body: {
    messages?: ChatMessage[];
    assistant?: string | null;
    conversationId?: string | null;
    variant?: TextVariant;
    modelCode?: string | null; // selección manual (anula el router)
  };
  try {
    body = await req.json();
  } catch {
    return json({ error: "Petición inválida." }, 400);
  }
  const variant: TextVariant = body.variant === "code" ? "code" : "chat";
  const messages = (body.messages ?? []).filter((m) => m.role === "user" || m.role === "assistant");
  if (messages.length === 0) return json({ error: "Petición inválida." }, 400);

  const lastMsg = [...messages].reverse().find((x) => x.role === "user");
  const lastUser = lastMsg?.content ?? "";
  const firstUser = messages.find((x) => x.role === "user")?.content ?? "";

  // --- Resolver modelo + costo: manual (catálogo) o automático (router) ---
  let taskId = "manual";
  let modelCode: string;
  let cost: number;
  let caps = { accepts_image: false, accepts_pdf: false, accepts_audio: false };

  if (body.modelCode) {
    const m = await getModelByCode(body.modelCode);
    // Chat y Código comparten la mecánica de texto, así que ambos aceptan
    // modelos de cualquiera de las dos familias; imagen y audio no sirven aquí.
    if (!m || (m.category !== "chat" && m.category !== "code")) {
      return json({ error: "Modelo no disponible." }, 400);
    }
    modelCode = m.code;
    cost = m.creditCost;
    caps = { accepts_image: m.acceptsImage, accepts_pdf: m.acceptsPdf, accepts_audio: m.acceptsAudio };
  } else {
    taskId = resolveTextTask(variant, lastUser);
    const task = await getTaskConfig(taskId);
    if (!task || !task.modelCode) return json({ error: "Configuración de IA no disponible." }, 500);
    modelCode = task.modelCode;
    cost = task.cost;
  }

  const key = process.env.OPENROUTER_API_KEY;
  if (!key) return json({ error: "El servidor no tiene configurada la IA (OPENROUTER_API_KEY)." }, 503);

  const charge = await spendCredits(user.id, cost, {
    task_id: taskId,
    router_selected_model: modelCode,
    prompt_preview: lastUser.slice(0, 120),
  });
  if (!charge.ok) {
    return json({ error: "No tienes créditos suficientes. Recarga en la tienda." }, 402);
  }

  // Cerebro TRIADA (+ modo código, + asistente elegido en la sección Agentes).
  let systemPrompt = TRIADA_SYSTEM_PROMPT;
  if (variant === "code") systemPrompt += `\n\n--- Modo código ---\n${CODE_SYSTEM_PROMPT}`;
  if (body.assistant) {
    const ap = await getAssistantPrompt(body.assistant);
    if (ap) systemPrompt += `\n\n--- Rol especializado ---\n${ap}`;
  }

  let conversationId: string;
  try {
    conversationId = await ensureConversation({
      userId: user.id,
      conversationId: body.conversationId,
      modelCode,
      assistantCode: body.assistant ?? null,
      firstUserContent: firstUser,
    });
  } catch {
    await refundCredits(user.id, cost, { motivo: "conversacion" });
    return json({ error: "No se pudo iniciar la conversación." }, 500);
  }

  // Adjuntos del último mensaje del usuario: solo los que el modelo acepta.
  const rawAttachments = lastMsg?.attachments ?? [];
  const usableAttachments = filterByCapability(rawAttachments, caps);

  // Construir el arreglo de mensajes para OpenRouter, multimodal en el último turno.
  const orMessages: Array<{ role: string; content: string | Array<Record<string, unknown>> }> = [
    { role: "system", content: systemPrompt },
    ...messages.slice(0, -1).map((m) => ({ role: m.role, content: m.content })),
  ];
  if (lastMsg) {
    orMessages.push({
      role: "user",
      content:
        usableAttachments.length > 0
          ? buildMultimodalContent(lastMsg.content, usableAttachments)
          : lastMsg.content,
    });
  }

  let upstream: Response;
  try {
    upstream = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "HTTP-Referer": "https://triada.app",
        "X-Title": "Hub Triada",
      },
      body: JSON.stringify({
        model: modelCode,
        messages: orMessages,
        stream: true,
        // Pide el conteo de tokens en el último chunk (alimenta el resumen de actividad).
        stream_options: { include_usage: true },
      }),
    });
  } catch {
    await refundCredits(user.id, cost, { motivo: "conexion" });
    return json({ error: "No se pudo conectar con el proveedor de IA." }, 502);
  }

  if (!upstream.ok || !upstream.body) {
    const txt = await upstream.text().catch(() => "");
    await refundCredits(user.id, cost, { motivo: `proveedor_${upstream.status}` });
    return json({ error: `Error del proveedor (${upstream.status}). ${txt.slice(0, 160)}` }, 502);
  }

  let assistantAcc = "";
  let promptTokens: number | null = null;
  let completionTokens: number | null = null;
  const decoder = new TextDecoder();
  let buf = "";
  const transform = new TransformStream<Uint8Array, Uint8Array>({
    transform(chunk, controller) {
      controller.enqueue(chunk);
      buf += decoder.decode(chunk, { stream: true });
      let i: number;
      while ((i = buf.indexOf("\n")) !== -1) {
        const line = buf.slice(0, i).trim();
        buf = buf.slice(i + 1);
        if (line.startsWith("data:")) {
          const d = line.slice(5).trim();
          if (d && d !== "[DONE]") {
            try {
              const j = JSON.parse(d);
              const delta: string | undefined = j.choices?.[0]?.delta?.content;
              if (delta) assistantAcc += delta;
              if (j.usage) {
                promptTokens = j.usage.prompt_tokens ?? promptTokens;
                completionTokens = j.usage.completion_tokens ?? completionTokens;
              }
            } catch {
              /* parcial */
            }
          }
        }
      }
    },
    async flush() {
      if (!assistantAcc.trim()) return;
      try {
        await persistTurn(conversationId, lastUser, assistantAcc, rawAttachments);
      } catch {
        /* no romper por guardado */
      }
      try {
        await recordUsage({
          userId: user.id,
          modelCode,
          promptTokens,
          completionTokens,
          creditsSpent: cost,
          promptText: lastUser,
          completionText: assistantAcc,
        });
      } catch {
        /* la métrica no debe romper la respuesta */
      }
    },
  });

  return new Response(upstream.body.pipeThrough(transform), {
    headers: {
      "Content-Type": "text/event-stream; charset=utf-8",
      "Cache-Control": "no-cache, no-transform",
      "X-Conversation-Id": conversationId,
      "X-Task-Id": taskId,
      "X-Model-Used": modelCode,
      "X-Credits-Spent": String(cost),
      "X-Balance": String(charge.balance),
    },
  });
}
```

### 107. `app/api/ia/imagen/route.ts`
Genera o edita imágenes con costo fijo por modelo; guarda en Storage y en `ai_creations`.
```ts
/**
 * Generación/edición de imágenes — COSTO FIJO por modelo elegido.
 * Si viene `referenceImageUrl`, es una EDICIÓN (image-to-image): se envía la
 * imagen + instrucciones al modelo. Si no, es generación desde texto.
 */
import { createClient } from "@/lib/supabase/server";
import { createAdminClient } from "@/lib/supabase/admin";
import { TRIADA_SYSTEM_PROMPT } from "@/lib/ai/triada-context";
import { getModelByCode } from "@/lib/ai/hub";
import { spendCredits, refundCredits } from "@/lib/ai/credits";
import { recordUsage } from "@/lib/ai/activity";

export const dynamic = "force-dynamic";

function json(obj: unknown, status: number): Response {
  return new Response(JSON.stringify(obj), { status, headers: { "Content-Type": "application/json" } });
}

export async function POST(req: Request): Promise<Response> {
  const supabase = createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return json({ error: "No autorizado." }, 401);

  let body: {
    prompt?: string;
    modelCode?: string;
    referenceImageUrl?: string | null;
    aspectRatio?: string;
    quality?: "standard" | "high";
  };
  try {
    body = await req.json();
  } catch {
    return json({ error: "Petición inválida." }, 400);
  }
  const basePrompt = (body.prompt ?? "").trim();
  if (!basePrompt) return json({ error: "Escribe qué imagen quieres crear." }, 400);

  // Formato/calidad como instrucción textual (funciona en cualquier proveedor,
  // sin depender de un parámetro nativo que no todos soportan igual).
  const hints: string[] = [];
  if (body.aspectRatio) hints.push(`relación de aspecto ${body.aspectRatio}`);
  if (body.quality === "high") hints.push("máxima calidad y resolución, muy detallado");
  const prompt = hints.length > 0 ? `${basePrompt}\n\n(${hints.join(", ")})` : basePrompt;

  const model = body.modelCode ? await getModelByCode(body.modelCode) : await getModelByCode("google/gemini-3.1-flash-image");
  if (!model || model.category !== "image") return json({ error: "Modelo de imagen no disponible." }, 400);

  const key = process.env.OPENROUTER_API_KEY;
  if (!key) return json({ error: "El servidor no tiene configurada la IA (OPENROUTER_API_KEY)." }, 503);

  const charge = await spendCredits(user.id, model.creditCost, {
    task_id: body.referenceImageUrl ? "image-edit" : "premium-image",
    model: model.code,
    prompt_preview: basePrompt.slice(0, 120),
  });
  if (!charge.ok) return json({ error: "No tienes créditos suficientes para generar esta imagen." }, 402);

  const admin = createAdminClient();
  const userContent: Array<Record<string, unknown>> = [{ type: "text", text: prompt }];
  if (body.referenceImageUrl) {
    userContent.unshift({ type: "image_url", image_url: { url: body.referenceImageUrl } });
  }

  let data: { choices?: { message?: { images?: { image_url?: { url?: string } }[] } }[] };
  try {
    const upstream = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "HTTP-Referer": "https://triada.app",
        "X-Title": "TRIADA Creative Hub",
      },
      body: JSON.stringify({
        model: model.code,
        modalities: ["image", "text"],
        messages: [
          { role: "system", content: TRIADA_SYSTEM_PROMPT },
          { role: "user", content: userContent },
        ],
      }),
    });
    if (!upstream.ok) {
      const txt = await upstream.text().catch(() => "");
      await refundCredits(user.id, model.creditCost, { motivo: `proveedor_${upstream.status}` });
      return json({ error: `Error del proveedor (${upstream.status}). ${txt.slice(0, 160)}` }, 502);
    }
    data = await upstream.json();
  } catch {
    await refundCredits(user.id, model.creditCost, { motivo: "conexion" });
    return json({ error: "No se pudo conectar con el proveedor de IA." }, 502);
  }

  const imgUrl = data.choices?.[0]?.message?.images?.[0]?.image_url?.url;
  if (!imgUrl) {
    await refundCredits(user.id, model.creditCost, { motivo: "sin_imagen" });
    return json({ error: "El modelo no devolvió una imagen. Intenta con otro prompt." }, 502);
  }

  let publicUrl = imgUrl;
  if (imgUrl.startsWith("data:")) {
    try {
      await admin.storage.createBucket("creaciones", { public: true });
    } catch {
      /* ya existe */
    }
    const b64 = imgUrl.split(",")[1] ?? "";
    const buffer = Buffer.from(b64, "base64");
    const path = `${user.id}/${crypto.randomUUID()}.png`;
    const { error: upErr } = await admin.storage
      .from("creaciones")
      .upload(path, buffer, { contentType: "image/png", upsert: true });
    if (upErr) return json({ error: `No se pudo guardar la imagen: ${upErr.message}` }, 500);
    publicUrl = admin.storage.from("creaciones").getPublicUrl(path).data.publicUrl;
  }

  await admin.from("ai_creations").insert({
    user_id: user.id,
    kind: body.referenceImageUrl ? "edit" : "image",
    prompt: basePrompt,
    model_code: model.code,
    url: publicUrl,
    credits_spent: model.creditCost,
  });
  await recordUsage({
    userId: user.id,
    modelCode: model.code,
    creditsSpent: model.creditCost,
    promptText: prompt,
    completionText: "",
  });

  return json({ url: publicUrl, prompt: basePrompt, balance: charge.balance }, 200);
}
```

### 108. `app/api/ia/audio/route.ts`
Genera audio (música/voz) y lo guarda.
```ts
/**
 * Generación de audio (música y voz) — COSTO FIJO por modelo elegido.
 *
 * OpenRouter expone los modelos de audio con la forma de OpenAI: se pide
 * `modalities: ["text","audio"]` y la respuesta trae el audio en base64. Los
 * modelos de música (Lyria) y los de voz (GPT Audio) devuelven el binario en
 * campos distintos según el proveedor, así que se leen ambas formas.
 */
import { createClient } from "@/lib/supabase/server";
import { createAdminClient } from "@/lib/supabase/admin";
import { getModelByCode } from "@/lib/ai/hub";
import { spendCredits, refundCredits } from "@/lib/ai/credits";
import { recordUsage } from "@/lib/ai/activity";

export const dynamic = "force-dynamic";

function json(obj: unknown, status: number): Response {
  return new Response(JSON.stringify(obj), { status, headers: { "Content-Type": "application/json" } });
}

interface AudioChoice {
  message?: {
    audio?: { data?: string; format?: string; transcript?: string };
    images?: { image_url?: { url?: string } }[];
    content?: string;
  };
}

/** Extrae el audio de la respuesta, cubriendo las dos formas que devuelve OpenRouter. */
function extractAudio(choice: AudioChoice | undefined): { b64: string; format: string } | null {
  const audio = choice?.message?.audio;
  if (audio?.data) return { b64: audio.data, format: audio.format || "mp3" };

  // Algunos proveedores (Gemini/Lyria) entregan el medio como data-URL.
  const url = choice?.message?.images?.[0]?.image_url?.url;
  if (url?.startsWith("data:audio")) {
    const [head, b64] = url.split(",");
    const format = head.match(/data:audio\/(\w+)/)?.[1] ?? "mp3";
    if (b64) return { b64, format };
  }
  return null;
}

export async function POST(req: Request): Promise<Response> {
  const supabase = createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return json({ error: "No autorizado." }, 401);

  let body: { prompt?: string; modelCode?: string; voice?: string };
  try {
    body = await req.json();
  } catch {
    return json({ error: "Petición inválida." }, 400);
  }
  const prompt = (body.prompt ?? "").trim();
  if (!prompt) return json({ error: "Escribe qué audio quieres crear." }, 400);

  const model = body.modelCode
    ? await getModelByCode(body.modelCode)
    : await getModelByCode("openai/gpt-audio-mini");
  if (!model || model.category !== "audio") return json({ error: "Modelo de audio no disponible." }, 400);

  const key = process.env.OPENROUTER_API_KEY;
  if (!key) return json({ error: "El servidor no tiene configurada la IA (OPENROUTER_API_KEY)." }, 503);

  const charge = await spendCredits(user.id, model.creditCost, {
    task_id: "premium-audio",
    model: model.code,
    prompt_preview: prompt.slice(0, 120),
  });
  if (!charge.ok) return json({ error: "No tienes créditos suficientes para generar este audio." }, 402);

  let data: { choices?: AudioChoice[] };
  try {
    const upstream = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "HTTP-Referer": "https://triada.app",
        "X-Title": "Hub Triada",
      },
      body: JSON.stringify({
        model: model.code,
        modalities: ["text", "audio"],
        audio: { voice: body.voice || "alloy", format: "mp3" },
        messages: [{ role: "user", content: prompt }],
      }),
    });
    if (!upstream.ok) {
      const txt = await upstream.text().catch(() => "");
      await refundCredits(user.id, model.creditCost, { motivo: `proveedor_${upstream.status}` });
      return json({ error: `Error del proveedor (${upstream.status}). ${txt.slice(0, 160)}` }, 502);
    }
    data = await upstream.json();
  } catch {
    await refundCredits(user.id, model.creditCost, { motivo: "conexion" });
    return json({ error: "No se pudo conectar con el proveedor de IA." }, 502);
  }

  const audio = extractAudio(data.choices?.[0]);
  if (!audio) {
    await refundCredits(user.id, model.creditCost, { motivo: "sin_audio" });
    return json({ error: "El modelo no devolvió audio. Intenta con otro prompt u otro modelo." }, 502);
  }

  const admin = createAdminClient();
  try {
    await admin.storage.createBucket("creaciones", { public: true });
  } catch {
    /* ya existe */
  }
  const buffer = Buffer.from(audio.b64, "base64");
  const path = `${user.id}/${crypto.randomUUID()}.${audio.format}`;
  const { error: upErr } = await admin.storage
    .from("creaciones")
    .upload(path, buffer, { contentType: `audio/${audio.format}`, upsert: true });
  if (upErr) return json({ error: `No se pudo guardar el audio: ${upErr.message}` }, 500);
  const publicUrl = admin.storage.from("creaciones").getPublicUrl(path).data.publicUrl;

  await admin.from("ai_creations").insert({
    user_id: user.id,
    kind: "audio",
    prompt,
    model_code: model.code,
    url: publicUrl,
    credits_spent: model.creditCost,
  });
  await recordUsage({
    userId: user.id,
    modelCode: model.code,
    creditsSpent: model.creditCost,
    promptText: prompt,
    completionText: "",
  });

  return json({ url: publicUrl, prompt, balance: charge.balance }, 200);
}
```

### 109. `app/api/ia/superprompt/route.ts`
Mejora de prompts.
```ts
/** Super Prompter: expande una idea corta en un prompt detallado (costo fijo, económico). */
import { createClient } from "@/lib/supabase/server";
import { getTaskConfig } from "@/lib/ai/router";
import { spendCredits, refundCredits } from "@/lib/ai/credits";
import { superPrompterSystem } from "@/lib/ai/superprompter-context";

export const dynamic = "force-dynamic";

function json(obj: unknown, status: number): Response {
  return new Response(JSON.stringify(obj), { status, headers: { "Content-Type": "application/json" } });
}

export async function POST(req: Request): Promise<Response> {
  const supabase = createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return json({ error: "No autorizado." }, 401);

  let body: { idea?: string; target?: "image" | "video" };
  try {
    body = await req.json();
  } catch {
    return json({ error: "Petición inválida." }, 400);
  }
  const idea = (body.idea ?? "").trim();
  if (!idea) return json({ error: "Escribe una idea." }, 400);
  const target = body.target === "video" ? "video" : "image";

  const task = await getTaskConfig("prompt-enhance");
  if (!task || !task.modelCode) return json({ error: "Configuración no disponible." }, 500);

  const key = process.env.OPENROUTER_API_KEY;
  if (!key) return json({ error: "El servidor no tiene configurada la IA (OPENROUTER_API_KEY)." }, 503);

  const charge = await spendCredits(user.id, task.cost, { task_id: "prompt-enhance", idea_preview: idea.slice(0, 120) });
  if (!charge.ok) return json({ error: "No tienes créditos suficientes." }, 402);

  try {
    const upstream = await fetch("https://openrouter.ai/api/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "HTTP-Referer": "https://triada.app",
        "X-Title": "TRIADA Super Prompter",
      },
      body: JSON.stringify({
        model: task.modelCode,
        messages: [
          { role: "system", content: superPrompterSystem(target) },
          { role: "user", content: idea },
        ],
      }),
    });
    if (!upstream.ok) {
      const txt = await upstream.text().catch(() => "");
      await refundCredits(user.id, task.cost, { motivo: `proveedor_${upstream.status}` });
      return json({ error: `Error del proveedor (${upstream.status}). ${txt.slice(0, 160)}` }, 502);
    }
    const data = await upstream.json();
    const prompt: string = data.choices?.[0]?.message?.content?.trim() ?? "";
    if (!prompt) {
      await refundCredits(user.id, task.cost, { motivo: "sin_respuesta" });
      return json({ error: "No se pudo generar el prompt." }, 502);
    }
    return json({ prompt, balance: charge.balance }, 200);
  } catch {
    await refundCredits(user.id, task.cost, { motivo: "conexion" });
    return json({ error: "No se pudo conectar con el proveedor de IA." }, 502);
  }
}
```

### 110. `app/api/ia/upload/route.ts`
Subida de adjuntos del chat.
```ts
/** Sube un adjunto del chat (imagen/PDF/audio-o-guion). Requiere sesión. */
import { createClient } from "@/lib/supabase/server";
import { uploadChatAttachment } from "@/lib/ai/attachments";

export const dynamic = "force-dynamic";

function json(obj: unknown, status: number): Response {
  return new Response(JSON.stringify(obj), { status, headers: { "Content-Type": "application/json" } });
}

export async function POST(req: Request): Promise<Response> {
  const supabase = createClient();
  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) return json({ error: "No autorizado." }, 401);

  const form = await req.formData().catch(() => null);
  const file = form?.get("file");
  if (!file || !(file instanceof File)) return json({ error: "Falta el archivo." }, 400);

  const result = await uploadChatAttachment(user.id, file);
  if (!result.ok) return json({ error: result.error }, 400);
  return json({ attachment: result.attachment }, 200);
}
```

### 111. `lib/ai/hub.ts`
Catálogo de modelos por categoría, asistentes, paquetes de recarga, nivel del usuario, saldo y persistencia de conversaciones.
```ts
/**
 * Centro de IA — catálogo de modelos (con capacidades), asistentes, créditos
 * y persistencia de conversaciones.
 * ⚠️ SOLO SERVIDOR (usa el cliente admin).
 */
import { createAdminClient } from "@/lib/supabase/admin";
import type { Attachment } from "./attachments";

export type AiCategory = "chat" | "code" | "image" | "audio";

const CATEGORIES: AiCategory[] = ["chat", "code", "image", "audio"];

export interface AiModel {
  code: string; // id de OpenRouter
  name: string;
  provider: string;
  tier: string; // '', '$'..'$$$$', 'Gratis'
  category: AiCategory;
  creditCost: number; // costo fijo por uso cuando se elige este modelo a mano
  acceptsImage: boolean;
  acceptsPdf: boolean;
  acceptsAudio: boolean;
  maxAttachmentMb: number;
  isDefault: boolean;
}

export interface AiAssistant {
  code: string;
  name: string;
  description: string;
  icon: string;
  category: string;
  model_code: string | null;
}

export interface AiConversationSummary {
  id: string;
  title: string | null;
  updated_at: string;
  assistant_code: string | null;
}

export interface AiChatMessage {
  role: "user" | "assistant";
  content: string;
  attachments?: Attachment[];
}

function mapModel(m: {
  code: string;
  name: string;
  provider: string;
  tier: string | null;
  category: string;
  credit_cost: number | string;
  accepts_image: boolean;
  accepts_pdf: boolean;
  accepts_audio: boolean;
  max_attachment_mb: number;
  is_default: boolean;
}): AiModel {
  return {
    code: m.code,
    name: m.name,
    provider: m.provider,
    tier: m.tier ?? "",
    category: (CATEGORIES as string[]).includes(m.category) ? (m.category as AiCategory) : "chat",
    creditCost: Number(m.credit_cost),
    acceptsImage: m.accepts_image,
    acceptsPdf: m.accepts_pdf,
    acceptsAudio: m.accepts_audio,
    maxAttachmentMb: m.max_attachment_mb,
    isDefault: m.is_default,
  };
}

const MODEL_FIELDS =
  "code, name, provider, tier, category, credit_cost, accepts_image, accepts_pdf, accepts_audio, max_attachment_mb, is_default";

/** Catálogo de modelos activos (opcionalmente filtrado por categoría). */
export async function loadAiModels(category?: AiCategory): Promise<AiModel[]> {
  const supabase = createAdminClient();
  let q = supabase
    .from("ai_models")
    .select(MODEL_FIELDS)
    .eq("is_active", true)
    .order("sort", { ascending: true });
  if (category) q = q.eq("category", category);
  const { data } = await q;
  return (data ?? []).map(mapModel);
}

/** Un modelo por su código (para validar/costear una elección manual). */
export async function getModelByCode(code: string): Promise<AiModel | null> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_models")
    .select(MODEL_FIELDS)
    .eq("code", code)
    .eq("is_active", true)
    .maybeSingle();
  return data ? mapModel(data) : null;
}

/** Asistentes ("Mis Ayudantes") activos. */
export async function loadAssistants(): Promise<AiAssistant[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_assistants")
    .select("code, name, description, icon, category, model_code, sort")
    .eq("is_active", true)
    .order("sort", { ascending: true });
  return (data ?? []).map((a) => ({
    code: a.code,
    name: a.name,
    description: a.description,
    icon: a.icon,
    category: a.category,
    model_code: a.model_code ?? null,
  }));
}

/** system_prompt de un asistente (o null si no existe). Solo servidor. */
export async function getAssistantPrompt(code: string): Promise<string | null> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_assistants")
    .select("system_prompt")
    .eq("code", code)
    .eq("is_active", true)
    .maybeSingle();
  return data?.system_prompt ?? null;
}

export interface CreditPack {
  id: string;
  name: string;
  priceUsd: number;
  credits: number;
  bonusLabel: string;
}

/** Paquetes de recarga de créditos activos (tienda). */
export async function getCreditPacks(): Promise<CreditPack[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ia_credit_packs")
    .select("id, name, price_usd, credits, bonus_label, sort")
    .eq("is_active", true)
    .order("sort", { ascending: true });
  return (data ?? []).map((p) => ({
    id: p.id,
    name: p.name,
    priceUsd: Number(p.price_usd),
    credits: p.credits,
    bonusLabel: p.bonus_label ?? "",
  }));
}

/**
 * Nivel de paquete del usuario (1=Básico..4=Quantum; 0 si no tiene servicio
 * activo). Sirve para el gating de funciones (video≥2, agentes≥3).
 */
export async function getUserTier(userId: string): Promise<number> {
  const supabase = createAdminClient();
  const nowIso = new Date().toISOString();
  const { data } = await supabase
    .from("memberships")
    .select("program_code, status, expires_at, type")
    .eq("user_id", userId)
    .eq("type", "service")
    .eq("status", "active");
  const codes = (data ?? [])
    .filter((m) => m.expires_at === null || m.expires_at > nowIso)
    .map((m) => m.program_code)
    .filter((c): c is string => !!c);
  if (codes.length === 0) return 0;
  const { data: prods } = await supabase.from("products").select("code, tier_level").in("code", codes);
  return (prods ?? []).reduce((max, p) => Math.max(max, Number(p.tier_level ?? 0)), 0);
}

/** Saldo de créditos del usuario. */
export async function getAiBalance(userId: string): Promise<number> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_credits")
    .select("balance")
    .eq("user_id", userId)
    .maybeSingle();
  return Number(data?.balance ?? 0);
}

/** Lista de conversaciones del usuario (más recientes primero). */
export async function getConversations(userId: string): Promise<AiConversationSummary[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_conversations")
    .select("id, title, updated_at, assistant_code")
    .eq("user_id", userId)
    .order("updated_at", { ascending: false })
    .limit(100);
  return (data ?? []) as AiConversationSummary[];
}

/** Mensajes de una conversación (verifica propiedad). */
export async function getConversationMessages(
  userId: string,
  conversationId: string
): Promise<AiChatMessage[]> {
  const supabase = createAdminClient();
  const { data: conv } = await supabase
    .from("ai_conversations")
    .select("id")
    .eq("id", conversationId)
    .eq("user_id", userId)
    .maybeSingle();
  if (!conv) return [];

  const { data } = await supabase
    .from("ai_messages")
    .select("role, content, attachments")
    .eq("conversation_id", conversationId)
    .order("created_at", { ascending: true });
  return (data ?? []).map((m) => ({
    role: m.role === "assistant" ? "assistant" : "user",
    content: m.content,
    attachments: Array.isArray(m.attachments) ? (m.attachments as Attachment[]) : [],
  }));
}

/**
 * Devuelve el id de conversación válido para este turno: si viene uno y es del
 * usuario, lo usa; si no, crea una nueva (título derivado del primer mensaje).
 * Solo servidor (lo llama la ruta-proxy).
 */
export async function ensureConversation(params: {
  userId: string;
  conversationId?: string | null;
  modelCode: string;
  assistantCode?: string | null;
  firstUserContent: string;
}): Promise<string> {
  const supabase = createAdminClient();
  const { userId, conversationId, modelCode, assistantCode, firstUserContent } = params;

  if (conversationId) {
    const { data } = await supabase
      .from("ai_conversations")
      .select("id")
      .eq("id", conversationId)
      .eq("user_id", userId)
      .maybeSingle();
    if (data) return data.id;
  }

  const title = firstUserContent.trim().slice(0, 60) || null;
  const { data: created, error } = await supabase
    .from("ai_conversations")
    .insert({
      user_id: userId,
      title,
      assistant_code: assistantCode ?? null,
      model_code: modelCode,
    })
    .select("id")
    .single();
  if (error || !created) throw new Error("No se pudo crear la conversación.");
  return created.id;
}

/** Guarda el turno (mensaje del usuario + respuesta) y toca updated_at. */
export async function persistTurn(
  conversationId: string,
  userContent: string,
  assistantContent: string,
  userAttachments: Attachment[] = []
): Promise<void> {
  const supabase = createAdminClient();
  await supabase.from("ai_messages").insert([
    { conversation_id: conversationId, role: "user", content: userContent, attachments: userAttachments },
    { conversation_id: conversationId, role: "assistant", content: assistantContent },
  ]);
  await supabase
    .from("ai_conversations")
    .update({ updated_at: new Date().toISOString() })
    .eq("id", conversationId);
}
```

### 112. `lib/ai/router.ts`
Triada Auto Max: heurística que elige el modelo según lo que se pide.
```ts
/**
 * Triada Auto Max — clasificador semántico pre-petición.
 *
 * Decide, ANTES de llamar a OpenRouter, qué modelo atiende la petición según lo
 * que el usuario pidió: una charla corta va a un modelo económico, un encargo de
 * razonamiento o análisis va a uno potente, y en la sección Código va a un modelo
 * especializado. El usuario también puede saltarse esto y elegir el modelo a mano.
 *
 * El costo y el modelo destino de cada tarea viven en `ia_tasks_config` (editable
 * por SQL), así que afinar el router no requiere tocar código.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { getModelByCode } from "./hub";

/** Secciones que enrutan texto. Cada una tiene su propia familia de modelos. */
export type TextVariant = "chat" | "code";
export type TextTaskId = "economy-text" | "premium-text" | "code-text";

export interface TaskConfig {
  id: string;
  cost: number;
  modelCode: string | null;
}

// Señales de que la tarea requiere razonamiento profundo → modelo potente.
const PREMIUM_SIGNALS = [
  "codigo", "código", "code", "funcion", "función", "function", "script",
  "base de datos", "database", "sql", "api", "algoritmo", "depura", "debug",
  "refactor", "auditor", "analiza", "análisis", "analisis", "estrategia",
  "plan de", "fórmula", "formula", "excel", "programa", "backend", "frontend",
  "typescript", "python", "javascript", "react", "next.js", "compila", "error",
  "traduce", "ensayo", "tesis", "resume este", "analízame", "optimiza",
];

/** Heurística: ¿el prompt amerita el modelo potente? */
export function routeLLMTask(prompt: string): TextTaskId {
  const p = (prompt || "").toLowerCase();
  const isLong = p.length > 600;
  const hit = PREMIUM_SIGNALS.some((w) => p.includes(w));
  return isLong || hit ? "premium-text" : "economy-text";
}

/** Tarea de texto que corresponde a la sección + lo que se pidió. */
export function resolveTextTask(variant: TextVariant, prompt: string): TextTaskId {
  if (variant === "code") return "code-text";
  return routeLLMTask(prompt);
}

/** Lee costo fijo + modelo destino de una tarea desde ia_tasks_config. */
export async function getTaskConfig(taskId: string): Promise<TaskConfig | null> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ia_tasks_config")
    .select("id, fixed_credit_cost, model_code")
    .eq("id", taskId)
    .maybeSingle();
  if (!data) return null;
  return { id: data.id, cost: Number(data.fixed_credit_cost), modelCode: data.model_code };
}

export interface AutoCapabilities {
  accepts_image: boolean;
  accepts_pdf: boolean;
  accepts_audio: boolean;
}

/**
 * Capacidades de adjunto seguras para Triada Auto Max: intersección de lo que
 * aceptan todos los modelos a los que puede enrutar. No sabemos cuál elegirá
 * hasta enviar, así que solo habilitamos lo que TODOS soportan.
 */
export async function getAutoModeCapabilities(variant: TextVariant = "chat"): Promise<AutoCapabilities> {
  const taskIds: TextTaskId[] = variant === "code" ? ["code-text"] : ["economy-text", "premium-text"];
  const configs = await Promise.all(taskIds.map(getTaskConfig));
  const models = await Promise.all(
    configs.map((c) => (c?.modelCode ? getModelByCode(c.modelCode) : null))
  );
  if (models.some((m) => !m)) return { accepts_image: false, accepts_pdf: false, accepts_audio: false };
  return {
    accepts_image: models.every((m) => m!.acceptsImage),
    accepts_pdf: models.every((m) => m!.acceptsPdf),
    accepts_audio: models.every((m) => m!.acceptsAudio),
  };
}
```

### 113. `lib/ai/credits.ts`
Otorgar, cobrar y reembolsar créditos con ledger.
```ts
/**
 * Otorgamiento y registro de créditos IA (Fase F4).
 * ⚠️ SOLO SERVIDOR (cliente admin).
 *
 * Un solo saldo por usuario en `ai_credits`. Cada movimiento queda en el ledger
 * `ia_credits_transactions` (consumo/recarga/afiliacion/paquete).
 * Equivalencia: $1 USD = 1.000 créditos. El costo real de servir un crédito ≈ 1/3.
 */
import { createAdminClient } from "@/lib/supabase/admin";

export type CreditTxType = "consumo" | "recarga" | "afiliacion" | "paquete";

/** Suma (o resta, si amount<0) créditos al usuario y registra la transacción. */
export async function grantCredits(
  userId: string,
  amount: number,
  type: CreditTxType,
  metadata: Record<string, unknown> = {}
): Promise<number> {
  const supabase = createAdminClient();
  const { data: cur } = await supabase
    .from("ai_credits")
    .select("balance")
    .eq("user_id", userId)
    .maybeSingle();
  const next = Math.max(0, Number(cur?.balance ?? 0) + amount);

  await supabase
    .from("ai_credits")
    .upsert({ user_id: userId, balance: next, updated_at: new Date().toISOString() }, { onConflict: "user_id" });
  await supabase
    .from("ia_credits_transactions")
    .insert({ user_id: userId, type, amount, metadata });

  return next;
}

/** % de los créditos del paquete que gana el patrocinador al activar un directo. */
export const AFFILIATION_CREDIT_PCT = 0.1;

/**
 * Cobra un costo FIJO al usuario si tiene saldo suficiente. Registra 'consumo'.
 * Devuelve { ok, balance }. No lanza si falta saldo (ok=false).
 */
export async function spendCredits(
  userId: string,
  cost: number,
  metadata: Record<string, unknown> = {}
): Promise<{ ok: boolean; balance: number }> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_credits")
    .select("balance")
    .eq("user_id", userId)
    .maybeSingle();
  const bal = Number(data?.balance ?? 0);
  if (bal < cost) return { ok: false, balance: bal };

  const next = bal - cost;
  await supabase
    .from("ai_credits")
    .upsert({ user_id: userId, balance: next, updated_at: new Date().toISOString() }, { onConflict: "user_id" });
  await supabase
    .from("ia_credits_transactions")
    .insert({ user_id: userId, type: "consumo", amount: -cost, metadata });
  return { ok: true, balance: next };
}

/** Reembolsa un cobro (si la llamada al proveedor falló tras cobrar). */
export async function refundCredits(
  userId: string,
  cost: number,
  metadata: Record<string, unknown> = {}
): Promise<number> {
  return grantCredits(userId, cost, "recarga", { ...metadata, reembolso: true });
}
```

### 114. `lib/ai/creative.ts`
Creaciones del usuario y galería pública.
```ts
/**
 * Creative Hub — galería de creaciones (imágenes) y Tendencias (galería pública).
 * ⚠️ SOLO SERVIDOR (usa el cliente admin).
 */
import { createAdminClient } from "@/lib/supabase/admin";

export interface AiCreation {
  id: string;
  kind: string;
  prompt: string;
  model_code: string | null;
  url: string;
  created_at: string;
  is_public: boolean;
  credits_spent: number;
  views: number;
}

/** Creaciones del usuario (galería), más recientes primero. */
export async function getCreations(userId: string, limit = 60): Promise<AiCreation[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_creations")
    .select("id, kind, prompt, model_code, url, created_at, is_public, credits_spent, views")
    .eq("user_id", userId)
    .order("created_at", { ascending: false })
    .limit(limit);
  return (data ?? []) as AiCreation[];
}

export interface PublicCreation extends AiCreation {
  authorName: string;
}

/** Tendencias: creaciones marcadas públicas por cualquier usuario. */
export async function getPublicCreations(limit = 60): Promise<PublicCreation[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("ai_creations")
    .select("id, kind, prompt, model_code, url, created_at, is_public, credits_spent, views, users(full_name)")
    .eq("is_public", true)
    .order("created_at", { ascending: false })
    .limit(limit);
  return (data ?? []).map((c) => {
    const userRel = c.users as unknown as { full_name: string | null } | { full_name: string | null }[] | null;
    const author = Array.isArray(userRel) ? userRel[0] : userRel;
    return {
      id: c.id,
      kind: c.kind,
      prompt: c.prompt,
      model_code: c.model_code,
      url: c.url,
      created_at: c.created_at,
      is_public: c.is_public,
      credits_spent: Number(c.credits_spent),
      views: c.views,
      authorName: author?.full_name ?? "Distribuidor TRIADA",
    };
  });
}
```

### 115. `lib/ai/attachments.ts`
Subida y construcción del contenido multimodal según capacidades del modelo.
```ts
/**
 * Adjuntos del Centro de IA — subida a Storage + construcción del contenido
 * multimodal para OpenRouter (formato compatible OpenAI: image_url / file).
 * ⚠️ SOLO SERVIDOR.
 */
import { createAdminClient } from "@/lib/supabase/admin";

export type AttachmentKind = "image" | "pdf" | "audio";

export interface Attachment {
  kind: AttachmentKind;
  url: string;
  name: string;
  mime: string;
}

const KIND_BY_MIME = (mime: string): AttachmentKind | null => {
  if (mime.startsWith("image/")) return "image";
  if (mime === "application/pdf") return "pdf";
  if (mime.startsWith("audio/") || mime === "text/plain") return "audio"; // guion/texto viaja como "audio" en la UI de la sección Audio
  return null;
};

const MAX_MB: Record<AttachmentKind, number> = { image: 8, pdf: 15, audio: 15 };

/** Sube un adjunto al bucket "chat-adjuntos" (público) y devuelve su metadata. */
export async function uploadChatAttachment(
  userId: string,
  file: File
): Promise<{ ok: true; attachment: Attachment } | { ok: false; error: string }> {
  const kind = KIND_BY_MIME(file.type);
  if (!kind) return { ok: false, error: "Tipo de archivo no soportado." };

  const maxBytes = MAX_MB[kind] * 1024 * 1024;
  if (file.size > maxBytes) {
    return { ok: false, error: `El archivo supera el máximo permitido (${MAX_MB[kind]}MB).` };
  }

  const supabase = createAdminClient();
  try {
    await supabase.storage.createBucket("chat-adjuntos", { public: true });
  } catch {
    /* ya existe */
  }

  const ext = (file.name.split(".").pop() || "bin").toLowerCase();
  const path = `${userId}/${crypto.randomUUID()}.${ext}`;
  const buffer = Buffer.from(await file.arrayBuffer());
  const { error } = await supabase.storage
    .from("chat-adjuntos")
    .upload(path, buffer, { contentType: file.type, upsert: false });
  if (error) return { ok: false, error: `No se pudo subir: ${error.message}` };

  const url = supabase.storage.from("chat-adjuntos").getPublicUrl(path).data.publicUrl;
  return { ok: true, attachment: { kind, url, name: file.name, mime: file.type } };
}

/** Filtra los adjuntos según lo que el modelo activo realmente acepta. */
export function filterByCapability(
  attachments: Attachment[],
  caps: { accepts_image: boolean; accepts_pdf: boolean; accepts_audio: boolean }
): Attachment[] {
  return attachments.filter((a) => {
    if (a.kind === "image") return caps.accepts_image;
    if (a.kind === "pdf") return caps.accepts_pdf;
    if (a.kind === "audio") return caps.accepts_audio;
    return false;
  });
}

/**
 * Construye el `content` (array de partes) para el último mensaje del usuario,
 * en el formato multimodal que espera OpenRouter (compatible OpenAI).
 * Los adjuntos NO soportados por el modelo se omiten (ya filtrados antes).
 */
export function buildMultimodalContent(text: string, attachments: Attachment[]) {
  const parts: Array<Record<string, unknown>> = [{ type: "text", text }];
  for (const a of attachments) {
    if (a.kind === "image") {
      parts.push({ type: "image_url", image_url: { url: a.url } });
    } else if (a.kind === "pdf") {
      parts.push({ type: "file", file: { filename: a.name, file_data: a.url } });
    }
    // audio/texto-guion: se referencia por nombre en el texto (no hay input de audio nativo
    // fiable multi-proveedor en OpenRouter hoy); el usuario ve el adjunto en el chat igual.
  }
  return parts;
}
```

### 116. `lib/ai/activity.ts`
Registro de uso y agregados para el mapa de calor.
```ts
/**
 * Resumen de actividad del Hub Triada — alimenta el panel que se ve al abrir
 * un chat nuevo (métricas, mapa de calor y desglose por modelo).
 *
 * Se calculan los tres rangos de una vez para que el conmutador de la UI sea
 * instantáneo; el volumen por usuario es pequeño (una fila de `ai_usage` por
 * llamada), así que agregamos en memoria en vez de con SQL por rango.
 * ⚠️ SOLO SERVIDOR.
 */
import { createAdminClient } from "@/lib/supabase/admin";

export type ActivityRange = "todo" | "30d" | "7d";

/**
 * Registra una llamada en `ai_usage`. Es lo que alimenta el resumen de
 * actividad; el cobro vive aparte en el ledger de créditos.
 *
 * Los tokens son informativos (el cobro es fijo): si el proveedor no los
 * reporta, se estiman por longitud para que el mapa de calor no quede en cero.
 */
export async function recordUsage(params: {
  userId: string;
  modelCode: string;
  promptTokens?: number | null;
  completionTokens?: number | null;
  creditsSpent: number;
  promptText?: string;
  completionText?: string;
}): Promise<void> {
  const estimate = (s?: string) => Math.max(1, Math.round((s ?? "").length / 4));
  const supabase = createAdminClient();
  await supabase.from("ai_usage").insert({
    user_id: params.userId,
    model_code: params.modelCode,
    prompt_tokens: params.promptTokens ?? estimate(params.promptText),
    completion_tokens: params.completionTokens ?? estimate(params.completionText),
    credits_spent: params.creditsSpent,
  });
}

export interface ActivityMetrics {
  sessions: number;
  messages: number;
  tokens: number;
  activeDays: number;
  peakHour: string | null;
  topModel: string | null;
}

export interface ActivityBar {
  date: string;
  segments: { model: string; value: number }[];
}

export interface ActivityLegendRow {
  model: string;
  input: number;
  output: number;
  pct: number;
}

export interface ActivityRangeData {
  metrics: ActivityMetrics;
  bars: ActivityBar[];
  labels: string[];
  legend: ActivityLegendRow[];
  /** Celdas del mapa de calor en orden columna-por-columna (semanas), nivel 0–4. */
  heat: number[];
}

export type ActivityData = Record<ActivityRange, ActivityRangeData>;

interface UsageRow {
  model_code: string;
  prompt_tokens: number;
  completion_tokens: number;
  created_at: string;
}

const DAY_MS = 86_400_000;
const RANGE_DAYS: Record<ActivityRange, number | null> = { todo: null, "30d": 30, "7d": 7 };

/** Fecha local en formato YYYY-MM-DD (el mapa de calor se agrupa por día local). */
function dayKey(d: Date): string {
  return `${d.getFullYear()}-${String(d.getMonth() + 1).padStart(2, "0")}-${String(d.getDate()).padStart(2, "0")}`;
}

function shortLabel(key: string): string {
  const [, m, d] = key.split("-");
  const meses = ["ene", "feb", "mar", "abr", "may", "jun", "jul", "ago", "sept", "oct", "nov", "dic"];
  return `${Number(d)} ${meses[Number(m) - 1]}`;
}

function emptyRange(): ActivityRangeData {
  return {
    metrics: { sessions: 0, messages: 0, tokens: 0, activeDays: 0, peakHour: null, topModel: null },
    bars: [],
    labels: [],
    legend: [],
    heat: [],
  };
}

/**
 * Mapa de calor estilo contribuciones: 7 filas (lun→dom) × N semanas, llenado
 * columna por columna. El nivel 1–4 se reparte por cuartiles del uso real, así
 * que la escala se adapta a cada usuario en vez de usar umbrales fijos.
 */
function buildHeat(byDay: Map<string, number>, from: Date, to: Date): number[] {
  const start = new Date(from);
  // Alinear al lunes anterior para que cada columna sea una semana completa.
  const dow = (start.getDay() + 6) % 7;
  start.setDate(start.getDate() - dow);

  const values: number[] = [];
  const cells: number[] = [];
  for (let t = start.getTime(); t <= to.getTime(); t += DAY_MS) {
    const v = byDay.get(dayKey(new Date(t))) ?? 0;
    cells.push(v);
    if (v > 0) values.push(v);
  }
  if (values.length === 0) return cells.map(() => 0);

  values.sort((a, b) => a - b);
  const q = (p: number) => values[Math.min(values.length - 1, Math.floor(values.length * p))];
  const [q1, q2, q3] = [q(0.25), q(0.5), q(0.75)];
  return cells.map((v) => (v === 0 ? 0 : v <= q1 ? 1 : v <= q2 ? 2 : v <= q3 ? 3 : 4));
}

function summarize(
  rows: UsageRow[],
  conversationDates: string[],
  names: Map<string, string>,
  range: ActivityRange
): ActivityRangeData {
  const days = RANGE_DAYS[range];
  const now = new Date();
  const from = days === null ? null : new Date(now.getTime() - (days - 1) * DAY_MS);
  const inRange = (iso: string) => (from === null ? true : new Date(iso) >= from);

  const used = rows.filter((r) => inRange(r.created_at));
  if (used.length === 0) return emptyRange();

  const hours = new Array(24).fill(0);
  const callsByDay = new Map<string, number>();
  const byDayModel = new Map<string, Map<string, number>>();
  const totals = new Map<string, { input: number; output: number }>();
  let tokens = 0;

  for (const r of used) {
    const d = new Date(r.created_at);
    const key = dayKey(d);
    const inTok = Number(r.prompt_tokens) || 0;
    const outTok = Number(r.completion_tokens) || 0;

    hours[d.getHours()] += 1;
    callsByDay.set(key, (callsByDay.get(key) ?? 0) + 1);
    tokens += inTok + outTok;

    const perModel = byDayModel.get(key) ?? new Map<string, number>();
    perModel.set(r.model_code, (perModel.get(r.model_code) ?? 0) + inTok + outTok);
    byDayModel.set(key, perModel);

    const tot = totals.get(r.model_code) ?? { input: 0, output: 0 };
    tot.input += inTok;
    tot.output += outTok;
    totals.set(r.model_code, tot);
  }

  const sortedDays = [...byDayModel.keys()].sort();
  const bars: ActivityBar[] = sortedDays.map((date) => ({
    date,
    segments: [...byDayModel.get(date)!.entries()]
      .map(([code, value]) => ({ model: names.get(code) ?? code, value }))
      .sort((a, b) => b.value - a.value),
  }));

  // Hasta 7 etiquetas repartidas por el eje, para no amontonar texto.
  const step = Math.max(1, Math.ceil(sortedDays.length / 7));
  const labels = sortedDays.filter((_, i) => i % step === 0).map(shortLabel);

  const grand = [...totals.values()].reduce((a, t) => a + t.input + t.output, 0) || 1;
  const legend: ActivityLegendRow[] = [...totals.entries()]
    .map(([code, t]) => ({
      model: names.get(code) ?? code,
      input: t.input,
      output: t.output,
      pct: ((t.input + t.output) / grand) * 100,
    }))
    .sort((a, b) => b.pct - a.pct);

  const peakIdx = hours.indexOf(Math.max(...hours));
  const heatFrom = from ?? new Date(sortedDays[0]);

  return {
    metrics: {
      sessions: conversationDates.filter(inRange).length,
      messages: used.length,
      tokens,
      activeDays: sortedDays.length,
      peakHour: `${String(peakIdx).padStart(2, "0")}:00`,
      topModel: legend[0]?.model ?? null,
    },
    bars,
    labels,
    legend,
    heat: buildHeat(callsByDay, heatFrom, now),
  };
}

/** Actividad de IA del usuario, ya agregada para los tres rangos de la UI. */
export async function getActivity(userId: string): Promise<ActivityData> {
  const supabase = createAdminClient();
  const [{ data: usage }, { data: convs }, { data: models }] = await Promise.all([
    supabase
      .from("ai_usage")
      .select("model_code, prompt_tokens, completion_tokens, created_at")
      .eq("user_id", userId)
      .order("created_at", { ascending: true })
      .limit(5000),
    supabase.from("ai_conversations").select("created_at").eq("user_id", userId),
    supabase.from("ai_models").select("code, name"),
  ]);

  const names = new Map((models ?? []).map((m) => [m.code as string, m.name as string]));
  const rows = (usage ?? []) as UsageRow[];
  const convDates = (convs ?? []).map((c) => c.created_at as string);

  return {
    todo: summarize(rows, convDates, names, "todo"),
    "30d": summarize(rows, convDates, names, "30d"),
    "7d": summarize(rows, convDates, names, "7d"),
  };
}
```

### 117. `lib/ai/superprompter-context.ts`
Instrucciones del Super Prompter.
```ts
/** System prompt del Super Prompter: convierte una idea corta en un prompt detallado. */
export function superPrompterSystem(target: "image" | "video"): string {
  const media = target === "video" ? "video" : "imagen";
  return `Eres el "Super Prompter" de TRIADA: un experto en escribir prompts detallados para generar ${media} con IA.

Tu trabajo: tomar la idea corta del usuario y devolver UN prompt mejorado y detallado en español, listo para pegar en un generador de ${media}.

Reglas:
- Devuelve SOLO el prompt final, sin explicaciones, sin comillas, sin "Aquí tienes:", sin markdown.
- Sé específico: sujeto, estilo visual, composición, iluminación, colores, ambiente${target === "video" ? ", movimiento de cámara, duración/ritmo" : ""}.
- Máximo 80 palabras.
- Respeta la intención original del usuario, solo la enriqueces.`;
}
```

### 118. `lib/ai/triada-context.ts`
System prompts: asistente de propósito general con contexto de marca solo cuando aplica.
```ts
/**
 * Contexto base del Hub Triada que se antepone (como system prompt) a toda
 * conversación.
 *
 * Deliberadamente NO restringe los temas: la IA es una herramienta de uso
 * general y debe responder lo que se le pida. El conocimiento de TRIADA está
 * aquí para cuando la conversación sí trate de la marca o del negocio, no para
 * desviar cada respuesta hacia ella.
 */
export const TRIADA_SYSTEM_PROMPT = `Eres el asistente de IA de TRIADA Business School. Eres un asistente de proposito general: responde cualquier tema que te pidan (programacion, redaccion, estudio, ideas, analisis, vida diaria) con la misma calidad que cualquier IA de primer nivel.

Como trabajas:
- Haz exactamente lo que se te pide. No redirijas la conversacion hacia TRIADA ni menciones la marca si no viene al caso.
- Se practico y accionable: pasos concretos, ejemplos y plantillas. Evita rodeos y relleno.
- Responde en el idioma del usuario.
- Si no sabes algo, dilo en vez de inventarlo.

Contexto de marca (usalo SOLO si la conversacion trata de TRIADA, del negocio, de los rangos, del plan o de la academia):
- TRIADA Business School es una academia digital y comunidad de negocios de habla hispana (LATAM) que enseña a crear, hacer crecer y proteger dinero usando inteligencia artificial.
- La "Triada Financiera" son 3 etapas: CAPITALIZACION (crear dinero con IA y marketing digital), MULTIPLICACION (hacer crecer el dinero operando en mercados financieros) y RENTABILIZACION (preservar y proteger el capital).
- Es un ecosistema de tres partes: educacion (cursos de IA, marketing, automatizaciones, vibecoding, trading), tecnologia (esta plataforma con herramientas de IA) y un modelo de distribucion por recomendacion (multinivel binario con rangos y comisiones).
- Valores: innovacion, confianza (nunca se prometen ganancias garantizadas), excelencia y comunidad.
- Tono de marca: directo, estrategico, aspiracional pero aterrizado. Explica los terminos tecnicos en lenguaje sencillo.
- Vocabulario: la marca habla de "distribucion", "distribuidor" y "organizacion". NUNCA uses la palabra "referido".
- No inventes cifras exactas de precios, comisiones, puntos ni requisitos de rangos: si te los piden, sugiere revisarlos en el back office. No prometas resultados economicos ni uses lenguaje de "hazte rico rapido".`;

/** Instrucción extra para la sección Código. */
export const CODE_SYSTEM_PROMPT = `Eres un asistente de programacion experto. Responde con codigo correcto y ejecutable.

- Entrega el codigo en bloques con el lenguaje indicado.
- Explica brevemente las decisiones no obvias; no narres lo que el codigo ya dice.
- Si faltan datos para resolverlo bien (version, framework, entorno), pregunta antes de asumir.
- Señala riesgos de seguridad o rendimiento cuando los veas.`;
```


---

# PARTE 12 — DOMINIO Y MOTOR QUE TOCA AL AFILIADO
Reglas que se disparan cuando el afiliado compra, se registra o es evaluado.

### 119. `lib/domain/keys.ts`
Las 3 llaves. Funciones puras y testeadas.
```ts
/**
 * LAS 3 LLAVES DE ACTIVACIÓN — lógica central de TRIADA.
 *
 * Tres interruptores booleanos INDEPENDIENTES (spec sección 3). Esta es la
 * customización que más software implementa mal, por eso vive aislada y testeada.
 *
 *   has_position   → compró algún producto/programa → tiene posición y genera puntos.
 *   can_distribute → IBO activo y al día → puede referir (link de distribución).
 *   can_earn       → servicio activo ≥100 pts (Quantum+) Y can_distribute → cobra red.
 *
 * Estas funciones son PURAS: no tocan la base de datos. Reciben el estado ya
 * leído y devuelven las llaves. La capa de persistencia vive en `activation.ts`.
 */

/**
 * Umbral de puntos de un servicio para habilitar el cobro de comisiones de red.
 * Quantum (100 pts) o superior. Es un umbral estructural del plan; si algún día
 * cambia, puede moverse a `compensation_config`.
 */
export const EARN_MIN_POINTS = 100;

/** Estado de la cuenta necesario para calcular las llaves. */
export interface AccountState {
  /** ¿Tiene al menos una compra pagada (producto físico O programa)? */
  ownsAnyProduct: boolean;
  /** ¿Tiene una membresía IBO activa y vigente (no expirada)? */
  iboActive: boolean;
  /** Puntos del servicio activo de mayor valor vigente (0 si no tiene servicio activo). */
  activeServicePoints: number;
}

/** Las 3 llaves resultantes. */
export interface ActivationKeys {
  has_position: boolean;
  can_distribute: boolean;
  can_earn: boolean;
}

/**
 * Calcula las 3 llaves a partir del estado de la cuenta.
 *
 * Reglas (independientes entre sí):
 * - has_position: cualquier compra da posición. Un producto físico la activa,
 *   pero NO activa las otras dos llaves por sí solo.
 * - can_distribute: SOLO si el IBO está activo.
 * - can_earn: requiere servicio ≥ umbral (Quantum+) Y can_distribute.
 *   Si baja a Nova (50 pts) o se desactiva, can_earn vuelve a false.
 */
export function computeKeys(
  state: AccountState,
  earnMinPoints: number = EARN_MIN_POINTS
): ActivationKeys {
  const has_position = state.ownsAnyProduct;
  const can_distribute = state.iboActive;
  const can_earn = can_distribute && state.activeServicePoints >= earnMinPoints;

  return { has_position, can_distribute, can_earn };
}
```

### 120. `lib/domain/activation.ts`
Activación al comprar: orden, membresía, llaves, créditos de IA y bono de afiliación.
```ts
/**
 * Capa de persistencia de las 3 llaves + simulación de activación.
 *
 * ⚠️ SOLO SERVIDOR: usa el cliente admin (service_role). Nunca importar desde
 * un componente 'use client'.
 *
 * `refreshUserKeys` lee el estado real del usuario (compras y membresías) y
 * recalcula/guarda las 3 llaves con la lógica pura de `keys.ts`.
 *
 * `simulateActivation` crea una compra "pagada" de prueba (sin pasarela real;
 * la pasarela USDT es de la Fase E) para poder ver las llaves prenderse.
 * Es también el germen del checkout real.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { computeKeys, type AccountState, type ActivationKeys } from "./keys";
import { processPaidOrder } from "@/lib/engine/process-order";
import { grantCredits, AFFILIATION_CREDIT_PCT } from "@/lib/ai/credits";

/** Lee de la BD el estado de activación de un usuario. */
export async function getActivationState(userId: string): Promise<AccountState> {
  const supabase = createAdminClient();
  const nowIso = new Date().toISOString();

  // ¿Tiene alguna compra pagada? → posición en el árbol.
  const { count: paidOrders } = await supabase
    .from("orders")
    .select("*", { count: "exact", head: true })
    .eq("user_id", userId)
    .eq("payment_status", "paid");

  // Membresías activas y vigentes (expires_at nulo = sin vencimiento).
  const { data: memberships } = await supabase
    .from("memberships")
    .select("type, points, status, expires_at")
    .eq("user_id", userId)
    .eq("status", "active");

  const vigentes = (memberships ?? []).filter(
    (m) => m.expires_at === null || m.expires_at > nowIso
  );

  const iboActive = vigentes.some((m) => m.type === "ibo");
  const activeServicePoints = vigentes
    .filter((m) => m.type === "service")
    .reduce((max, m) => Math.max(max, m.points ?? 0), 0);

  return {
    ownsAnyProduct: (paidOrders ?? 0) > 0,
    iboActive,
    activeServicePoints,
  };
}

/** Recalcula y PERSISTE las 3 llaves de un usuario. Devuelve las llaves. */
export async function refreshUserKeys(userId: string): Promise<ActivationKeys> {
  const supabase = createAdminClient();
  const state = await getActivationState(userId);
  const keys = computeKeys(state);

  const { error } = await supabase
    .from("users")
    .update({
      has_position: keys.has_position,
      can_distribute: keys.can_distribute,
      can_earn: keys.can_earn,
    })
    .eq("id", userId);

  if (error) throw new Error(`No se pudieron actualizar las llaves: ${error.message}`);
  return keys;
}

/**
 * Simula una compra PAGADA (sin pasarela real) para pruebas de la Fase B.
 * Crea la orden y, si es servicio o IBO, la membresía correspondiente.
 * Luego recalcula las 3 llaves.
 */
export async function simulateActivation(
  userId: string,
  productCode: string
): Promise<ActivationKeys> {
  const supabase = createAdminClient();

  // 1. Buscar el producto del catálogo.
  const { data: product, error: prodErr } = await supabase
    .from("products")
    .select("*")
    .eq("code", productCode)
    .single();
  if (prodErr || !product) throw new Error(`Producto no encontrado: ${productCode}`);

  // ¿Es recompra? (ya tenía inscripciones pagadas antes → bono de afiliación reducido).
  const { count: priorEnrollments } = await supabase
    .from("orders")
    .select("id", { count: "exact", head: true })
    .eq("user_id", userId)
    .eq("order_type", "enrollment")
    .eq("payment_status", "paid");
  const isRenewal = (priorEnrollments ?? 0) > 0;

  // 2. Tipo de orden según el producto.
  const orderType =
    product.type === "physical" ? "product" : "enrollment";

  // 3. Crear la orden marcada como pagada (simulación).
  const { data: orderRow, error: orderErr } = await supabase
    .from("orders")
    .insert({
      user_id: userId,
      product_id: product.id,
      order_type: orderType,
      amount: product.enrollment_price,
      points_awarded: product.points,
      payment_status: "paid",
      payment_method: "SIMULADO",
    })
    .select("id")
    .single();
  if (orderErr || !orderRow) {
    throw new Error(`No se pudo crear la orden: ${orderErr?.message}`);
  }

  // 4. Si es servicio o IBO, crear/renovar la membresía vigente.
  if (product.type === "service" || product.type === "ibo") {
    const starts = new Date();
    const expires = new Date(starts);
    expires.setMonth(expires.getMonth() + (product.duration_months || 1));

    const { error: memErr } = await supabase.from("memberships").insert({
      user_id: userId,
      type: product.type === "ibo" ? "ibo" : "service",
      program_code: product.code,
      points: product.points,
      status: "active",
      starts_at: starts.toISOString(),
      expires_at: expires.toISOString(),
      auto_renew: true,
    });
    if (memErr) throw new Error(`No se pudo crear la membresía: ${memErr.message}`);
  }

  // 5. Procesar la orden por el motor: volumen por el árbol + bono directo.
  await processPaidOrder(orderRow.id);

  // 5b. Créditos IA del paquete (Fase F4): otorgar los créditos incluidos y,
  //     si tiene patrocinador, el bono de afiliación en créditos (additivo,
  //     NO toca la comisión en dinero).
  const monthlyCredits = Number(product.ai_credits_monthly ?? 0);
  if (product.type === "service" && monthlyCredits > 0) {
    await grantCredits(userId, monthlyCredits, "paquete", { package: product.code });

    const { data: buyer } = await supabase
      .from("users")
      .select("sponsor_id")
      .eq("id", userId)
      .maybeSingle();
    if (buyer?.sponsor_id) {
      // Recompra/renovación paga la mitad del bono de afiliación.
      const pct = isRenewal ? AFFILIATION_CREDIT_PCT / 2 : AFFILIATION_CREDIT_PCT;
      const bonus = Math.round(monthlyCredits * pct);
      if (bonus > 0) {
        await grantCredits(buyer.sponsor_id, bonus, "afiliacion", {
          from_user: userId,
          package: product.code,
          renovacion: isRenewal,
        });
      }
    }
  }

  // 6. Recalcular las 3 llaves con el nuevo estado.
  return refreshUserKeys(userId);
}
```

### 121. `lib/domain/placement.ts`
Colocación de un miembro en una posición exacta del binario.
```ts
/**
 * COLOCACIÓN MANUAL EN EL ÁRBOL BINARIO.
 *
 * ⚠️ SOLO SERVIDOR (cliente admin).
 *
 * Modelo "activar primero, colocar después": los miembros se registran con
 * patrocinador pero SIN posición. Luego el patrocinador (o el admin) los coloca
 * en una posición binaria exacta (placement_id + pierna).
 *
 * La AUTORIZACIÓN (quién puede colocar a quién y dónde) se valida en la capa de
 * acciones; aquí solo se ejecuta la colocación con sus validaciones de integridad.
 *
 * Recordatorio: el árbol binario (placement_id + binary_leg) puede diferir del
 * árbol de patrocinio (sponsor_id).
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { accumulateVolume } from "@/lib/engine/volume";
import { currentPeriod } from "@/lib/engine/period";
import type { BinaryLeg } from "@/types/database";

/**
 * Coloca a `memberId` bajo `placementId` en la pierna `leg`.
 * Valida que el miembro esté sin colocar y que el slot esté libre.
 */
export async function placeMember(
  memberId: string,
  placementId: string,
  leg: BinaryLeg
): Promise<void> {
  const supabase = createAdminClient();

  const { data: member } = await supabase
    .from("users")
    .select("id, placement_id")
    .eq("id", memberId)
    .maybeSingle();
  if (!member) throw new Error("Miembro no encontrado.");
  if (member.placement_id) throw new Error("Este miembro ya está colocado en el árbol.");

  // El slot destino debe estar libre.
  const { data: occupied } = await supabase
    .from("users")
    .select("id")
    .eq("placement_id", placementId)
    .eq("binary_leg", leg)
    .maybeSingle();
  if (occupied) throw new Error("Esa posición del árbol ya está ocupada.");

  await applyPlacement(memberId, placementId, leg);
}

/**
 * Colocación AUTOMÁTICA (spillover) bajo `sponsorId`: busca el primer espacio
 * libre recorriendo el subárbol por niveles (BFS), para un llenado balanceado.
 * Se usa cuando el patrocinador es estudiante (sin IBO): el referido entra solo.
 */
export async function findAutoPlacementSlot(
  sponsorId: string
): Promise<{ placementId: string; leg: BinaryLeg }> {
  const supabase = createAdminClient();
  const queue: string[] = [sponsorId];

  for (let i = 0; i < 100000 && queue.length > 0; i++) {
    const node = queue.shift()!;
    const { data: children } = await supabase
      .from("users")
      .select("id, binary_leg")
      .eq("placement_id", node);

    const hasLeft = (children ?? []).some((c) => c.binary_leg === "left");
    const hasRight = (children ?? []).some((c) => c.binary_leg === "right");

    if (!hasLeft) return { placementId: node, leg: "left" };
    if (!hasRight) return { placementId: node, leg: "right" };

    // Ambos ocupados: seguir bajando (primero izquierda, luego derecha).
    const left = (children ?? []).find((c) => c.binary_leg === "left");
    const right = (children ?? []).find((c) => c.binary_leg === "right");
    if (left) queue.push(left.id);
    if (right) queue.push(right.id);
  }
  throw new Error("No se encontró un espacio libre para la colocación automática.");
}

/** Asigna placement y sube los puntos del período del miembro a los uplines. */
async function applyPlacement(
  memberId: string,
  placementId: string,
  leg: BinaryLeg
): Promise<void> {
  const supabase = createAdminClient();
  const { error } = await supabase
    .from("users")
    .update({ placement_id: placementId, binary_leg: leg })
    .eq("id", memberId);
  if (error) throw new Error(`No se pudo colocar: ${error.message}`);

  // El miembro pudo activarse ANTES de ser colocado: en ese caso sus puntos del
  // período no subieron por el árbol (no tenía posición). Ahora que tiene
  // posición, subimos sus puntos personales del período a los uplines.
  const period = currentPeriod();
  const monthStart = `${period}-01T00:00:00.000Z`;
  const { data: orders } = await supabase
    .from("orders")
    .select("points_awarded, created_at")
    .eq("user_id", memberId)
    .eq("payment_status", "paid")
    .gte("created_at", monthStart);

  const points = (orders ?? []).reduce((s, o) => s + (o.points_awarded ?? 0), 0);
  if (points > 0) {
    await accumulateVolume(memberId, points, period);
  }
}
```

### 122. `lib/engine/config.ts`
Carga de constantes y rangos desde la base de datos.
```ts
/**
 * Carga la configuración del plan de compensación desde la base de datos.
 *
 * NINGUNA constante está hardcodeada en el motor: todas viven en
 * `compensation_config` y `ranks`, editables desde el admin sin tocar código.
 * Las funciones puras de `calculations.ts` reciben estos valores como parámetros.
 *
 * ⚠️ SOLO SERVIDOR (usa el cliente admin).
 */
import { createAdminClient } from "@/lib/supabase/admin";

/** Constantes globales del plan (tabla compensation_config). */
export interface EngineConfig {
  POINT_VALUE: number;
  DIRECT_L1_PCT: number;
  DIRECT_L2_PCT: number;
  BINARY_PCT: number;
  RANK_PCT: number;
  MIN_WITHDRAWAL: number;
  MATCHING_MIN_RANK: number;
  LIFESTYLE_MIN_RANK: number;
  HOLDING_DAYS: number;
  WEEKLY_SPLIT: number;
  PAYOUT_ALERT_THRESHOLD: number;
}

/** Regla de un rango (tabla ranks). */
export interface RankRule {
  level: number;
  name: string;
  zone?: string | null;
  points_per_leg: number;
  max_pct_line: number;
  min_directs_left: number;
  min_directs_right: number;
  base_check: number;
  matching_levels: number[];
  lifestyle_bonus: number;
  lifestyle_directs: number;
}

/** Lee compensation_config y devuelve el objeto tipado. */
export async function loadEngineConfig(): Promise<EngineConfig> {
  const supabase = createAdminClient();
  const { data, error } = await supabase
    .from("compensation_config")
    .select("key, value");
  if (error) throw new Error(`No se pudo cargar la configuración: ${error.message}`);

  const map = new Map<string, unknown>();
  for (const row of data ?? []) map.set(row.key, row.value);

  const num = (key: keyof EngineConfig): number => {
    const v = map.get(key);
    const n = typeof v === "string" ? Number(v) : (v as number);
    if (v === undefined || Number.isNaN(n)) {
      throw new Error(`Config faltante o inválida: ${key}`);
    }
    return n;
  };

  return {
    POINT_VALUE: num("POINT_VALUE"),
    DIRECT_L1_PCT: num("DIRECT_L1_PCT"),
    DIRECT_L2_PCT: num("DIRECT_L2_PCT"),
    BINARY_PCT: num("BINARY_PCT"),
    RANK_PCT: num("RANK_PCT"),
    MIN_WITHDRAWAL: num("MIN_WITHDRAWAL"),
    MATCHING_MIN_RANK: num("MATCHING_MIN_RANK"),
    LIFESTYLE_MIN_RANK: num("LIFESTYLE_MIN_RANK"),
    HOLDING_DAYS: num("HOLDING_DAYS"),
    WEEKLY_SPLIT: num("WEEKLY_SPLIT"),
    PAYOUT_ALERT_THRESHOLD: num("PAYOUT_ALERT_THRESHOLD"),
  };
}

/** Lee la tabla ranks ordenada ascendente (1..13). */
export async function loadRanks(): Promise<RankRule[]> {
  const supabase = createAdminClient();
  const { data, error } = await supabase
    .from("ranks")
    .select("*")
    .order("level", { ascending: true });
  if (error) throw new Error(`No se pudieron cargar los rangos: ${error.message}`);

  return (data ?? []).map((r) => ({
    level: r.level,
    name: r.name,
    zone: r.zone ?? null,
    points_per_leg: r.points_per_leg,
    max_pct_line: Number(r.max_pct_line),
    min_directs_left: r.min_directs_left,
    min_directs_right: r.min_directs_right,
    base_check: Number(r.base_check),
    // matching_levels viene como JSONB (array) o string.
    matching_levels: Array.isArray(r.matching_levels)
      ? r.matching_levels.map(Number)
      : JSON.parse(r.matching_levels ?? "[]").map(Number),
    lifestyle_bonus: Number(r.lifestyle_bonus),
    lifestyle_directs: r.lifestyle_directs,
  }));
}
```

### 123. `lib/engine/period.ts`
Períodos 'YYYY-MM'.
```ts
/** Devuelve el período de cierre en formato 'YYYY-MM' (UTC). */
export function currentPeriod(d: Date = new Date()): string {
  const y = d.getUTCFullYear();
  const m = String(d.getUTCMonth() + 1).padStart(2, "0");
  return `${y}-${m}`;
}

/** Devuelve el período anterior a uno dado ('2026-06' → '2026-05'). */
export function previousPeriod(period: string): string {
  const [y, m] = period.split("-").map(Number);
  const d = new Date(Date.UTC(y, m - 2, 1)); // m-2 = mes anterior (0-indexado)
  return currentPeriod(d);
}
```

### 124. `lib/engine/calculations.ts`
Funciones PURAS: directo, binario, rango, liquidación de pierna menor, `qualifyRank`, matching y estilo de vida.
```ts
/**
 * ============================================================================
 * MOTOR DE COMISIONES — NÚCLEO PURO DE CÁLCULO
 * ============================================================================
 *
 * Estas funciones son PURAS: no tocan la base de datos, no tienen efectos
 * secundarios. Reciben números + la configuración del plan (parámetros) y
 * devuelven montos. Esto las hace 100% testeables y auditables — son el
 * corazón del sistema que el desarrollador externo debe revisar.
 *
 * Implementan exactamente el pseudocódigo de la spec (commission-engine.md).
 * Las 5 vías:
 *   Vía 1 — Bono de Distribución Directa (20% N1 / 10% N2 sobre inscripción)
 *   Vía 2 — Bono Binario (15% sobre pierna menor)
 *   Vía 3 — Bono de Rango (10% sobre pierna menor)
 *   Vía 4 — Bono de Igualación / Matching (rango 6+)
 *   Vía 5 — Bonos de Estilo de Vida (rango 10+)
 *
 * NOTA: el dinero se redondea a 2 decimales solo en los resultados expuestos.
 * ============================================================================
 */
import type { EngineConfig, RankRule } from "./config";

/** Redondeo financiero a 2 decimales (centavos). */
export function round2(n: number): number {
  return Math.round((n + Number.EPSILON) * 100) / 100;
}

// ----------------------------------------------------------------------------
// VÍA 1 — BONO DE DISTRIBUCIÓN DIRECTA
// ----------------------------------------------------------------------------
/**
 * Calcula el bono directo de niveles 1 y 2 sobre el valor de inscripción
 * COMISIONABLE (en NEXUS, ya viene sin los $500 de fondo).
 * Se paga por evento al confirmarse el pago de la inscripción.
 */
export function directBonus(
  comisionableAmount: number,
  cfg: Pick<EngineConfig, "DIRECT_L1_PCT" | "DIRECT_L2_PCT">
): { l1: number; l2: number } {
  return {
    l1: round2(comisionableAmount * cfg.DIRECT_L1_PCT),
    l2: round2(comisionableAmount * cfg.DIRECT_L2_PCT),
  };
}

// ----------------------------------------------------------------------------
// VÍAS 2 y 3 — BINARIO Y RANGO (sobre la pierna menor, en puntos)
// ----------------------------------------------------------------------------
export function binaryBonus(
  lesserPoints: number,
  cfg: Pick<EngineConfig, "POINT_VALUE" | "BINARY_PCT">
): number {
  return round2(lesserPoints * cfg.POINT_VALUE * cfg.BINARY_PCT);
}

export function rankBonus(
  lesserPoints: number,
  cfg: Pick<EngineConfig, "POINT_VALUE" | "RANK_PCT">
): number {
  return round2(lesserPoints * cfg.POINT_VALUE * cfg.RANK_PCT);
}

// ----------------------------------------------------------------------------
// CARRY-OVER — liquidación de un período binario
// ----------------------------------------------------------------------------
export interface Settlement {
  lesser: number; // volumen de la pierna menor (lo que se paga)
  greater: number; // volumen de la pierna mayor
  paidLegVolume: number; // = lesser
  leftCarryover: number; // arrastre al siguiente período
  rightCarryover: number;
}

/**
 * Liquida un período: se paga sobre la pierna menor; la mayor arrastra el
 * sobrante (carry-over); la menor queda en 0.
 *
 * `leftTotal`/`rightTotal` ya incluyen el carry-over del período anterior.
 */
export function settlePeriod(leftTotal: number, rightTotal: number): Settlement {
  const lesser = Math.min(leftTotal, rightTotal);
  const greater = Math.max(leftTotal, rightTotal);
  const newCarry = greater - lesser;

  // El sobrante se arrastra en la pierna que era la mayor.
  const leftIsGreaterOrEqual = leftTotal >= rightTotal;

  return {
    lesser,
    greater,
    paidLegVolume: lesser,
    leftCarryover: leftIsGreaterOrEqual ? newCarry : 0,
    rightCarryover: leftIsGreaterOrEqual ? 0 : newCarry,
  };
}

// ----------------------------------------------------------------------------
// CÁLCULO DE RANGO — las 4 condiciones simultáneas
// ----------------------------------------------------------------------------
export interface RankInput {
  leftTotal: number; // puntos pierna izquierda (con carry-over)
  rightTotal: number; // puntos pierna derecha (con carry-over)
  /**
   * Fracción (0..1) de los puntos de la pierna menor que provienen de la
   * MAYOR línea de patrocinio directa. Debe ser ≤ max_pct_line del rango.
   */
  maxLineFraction: number;
  directsLeftActive: number; // directos activos en la pierna izquierda
  directsRightActive: number; // directos activos en la pierna derecha
  canEarn: boolean; // condición 4: servicio ≥100 pts + IBO
}

/**
 * Devuelve el rango (objeto) más alto cuyas 4 condiciones se cumplen, o null.
 * Recorre ascendente; como los requisitos endurecen con el nivel, se corta al
 * primer rango que falle.
 */
export function qualifyRank(
  input: RankInput,
  ranks: RankRule[]
): RankRule | null {
  // Condición 4: sin can_earn no hay rango (no cobra red).
  if (!input.canEarn) return null;

  let qualified: RankRule | null = null;
  const ordered = [...ranks].sort((a, b) => a.level - b.level);

  for (const rank of ordered) {
    // Condición 1: ambas piernas ≥ puntos del rango.
    if (input.leftTotal < rank.points_per_leg) break;
    if (input.rightTotal < rank.points_per_leg) break;
    // Condición 2: máx % por línea de patrocinio.
    if (input.maxLineFraction > rank.max_pct_line) break;
    // Condición 3: directos mínimos activos por pierna.
    if (input.directsLeftActive < rank.min_directs_left) break;
    if (input.directsRightActive < rank.min_directs_right) break;

    qualified = rank;
  }

  return qualified;
}

// ----------------------------------------------------------------------------
// CHEQUE DEL PERÍODO (binario + rango) con compuertas can_earn / inactividad
// ----------------------------------------------------------------------------
export interface PeriodCheck {
  rankLevel: number;
  binary: number;
  rank: number;
  settlement: Settlement;
}

/**
 * Calcula el cheque base (binario + rango) de un usuario para el período y su
 * liquidación de carry-over. Aplica las compuertas:
 *  - Si `inactive` → no cobra y PIERDE todo el acumulado (carry-over a 0).
 *  - Si `!canEarn` → no cobra; el volumen igual liquida y arrastra normal.
 */
export function computePeriodCheck(
  input: RankInput & { inactive: boolean },
  ranks: RankRule[],
  cfg: EngineConfig
): PeriodCheck {
  // Inactividad: pierde TODO el acumulado, no cobra.
  if (input.inactive) {
    return {
      rankLevel: 0,
      binary: 0,
      rank: 0,
      settlement: {
        lesser: 0,
        greater: 0,
        paidLegVolume: 0,
        leftCarryover: 0,
        rightCarryover: 0,
      },
    };
  }

  const settlement = settlePeriod(input.leftTotal, input.rightTotal);
  const qualified = qualifyRank(input, ranks);

  // can_earn = false → no cobra binario ni rango, pero el volumen liquida/arrastra.
  if (!input.canEarn) {
    return { rankLevel: 0, binary: 0, rank: 0, settlement };
  }

  return {
    rankLevel: qualified?.level ?? 0,
    binary: binaryBonus(settlement.lesser, cfg),
    rank: rankBonus(settlement.lesser, cfg),
    settlement,
  };
}

// ----------------------------------------------------------------------------
// VÍA 4 — MATCHING (igualación sobre el binario de la downline de patrocinio)
// ----------------------------------------------------------------------------
/**
 * `binariesByGeneration[g]` = lista de montos de binario de los distribuidores
 * de la generación g+1 de la línea de patrocinio. `matchingLevels[g]` = % para
 * esa generación. Devuelve el total de matching.
 */
export function matchingBonus(
  binariesByGeneration: number[][],
  matchingLevels: number[]
): number {
  let total = 0;
  for (let g = 0; g < matchingLevels.length; g++) {
    const pct = matchingLevels[g];
    const gen = binariesByGeneration[g] ?? [];
    for (const binary of gen) total += binary * pct;
  }
  return round2(total);
}

// ----------------------------------------------------------------------------
// VÍA 5 — ESTILO DE VIDA (bono fijo en rangos altos)
// ----------------------------------------------------------------------------
/**
 * Devuelve el bono de estilo de vida si se cumplen: rango con bono > 0,
 * mantener el rango ≥ 3 meses consecutivos, y N directos activos.
 */
export function lifestyleBonus(
  rank: RankRule,
  monthsMaintained: number,
  directsActive: number
): number {
  if (rank.lifestyle_bonus <= 0) return 0;
  if (monthsMaintained < 3) return 0;
  if (directsActive < rank.lifestyle_directs) return 0;
  return round2(rank.lifestyle_bonus);
}

// ----------------------------------------------------------------------------
// PAGO SEMANAL — el cheque mensual validado se divide en N pagos
// ----------------------------------------------------------------------------
export function weeklyPayout(
  monthlyValidatedTotal: number,
  cfg: Pick<EngineConfig, "WEEKLY_SPLIT">
): number {
  if (cfg.WEEKLY_SPLIT <= 0) return 0;
  return round2(monthlyValidatedTotal / cfg.WEEKLY_SPLIT);
}
```

### 125. `lib/engine/volume.ts`
Sube los puntos de una compra por el árbol binario.
```ts
/**
 * ACUMULACIÓN DE VOLUMEN por el árbol binario.
 *
 * ⚠️ SOLO SERVIDOR (cliente admin).
 *
 * Cuando se paga un servicio (inscripción o renovación) o un producto físico
 * con puntos, esos puntos suben por el árbol binario hacia la raíz, sumándose
 * a la pierna (left/right) que corresponde en cada ancestro.
 *
 * Recordatorio: se usa el árbol BINARIO (placement_id + binary_leg), NO el de
 * patrocinio. Los $500 de fondo de NEXUS ya están excluidos (products.points
 * no los incluye).
 */
import { createAdminClient } from "@/lib/supabase/admin";
import type { BinaryLeg } from "@/types/database";

/** Suma puntos a una pierna del ledger de un usuario en un período. */
async function addVolume(
  userId: string,
  period: string,
  leg: BinaryLeg,
  points: number
): Promise<void> {
  const supabase = createAdminClient();

  const { data: existing } = await supabase
    .from("volume_ledger")
    .select("id, left_volume, right_volume")
    .eq("user_id", userId)
    .eq("period", period)
    .maybeSingle();

  if (!existing) {
    await supabase.from("volume_ledger").insert({
      user_id: userId,
      period,
      left_volume: leg === "left" ? points : 0,
      right_volume: leg === "right" ? points : 0,
    });
    return;
  }

  await supabase
    .from("volume_ledger")
    .update({
      left_volume: existing.left_volume + (leg === "left" ? points : 0),
      right_volume: existing.right_volume + (leg === "right" ? points : 0),
    })
    .eq("id", existing.id);
}

/**
 * Sube `points` por el árbol binario desde `buyerId` hasta la raíz.
 * En cada ancestro suma a la pierna que ocupa el hijo correspondiente.
 */
export async function accumulateVolume(
  buyerId: string,
  points: number,
  period: string
): Promise<void> {
  if (points <= 0) return;
  const supabase = createAdminClient();

  // Empezamos en el comprador y subimos por placement_id.
  let current: { placement_id: string | null; binary_leg: BinaryLeg | null } | null =
    (
      await supabase
        .from("users")
        .select("placement_id, binary_leg")
        .eq("id", buyerId)
        .maybeSingle()
    ).data;

  // Tope de seguridad contra ciclos por datos corruptos.
  for (let depth = 0; depth < 100000 && current?.placement_id && current.binary_leg; depth++) {
    const parentId = current.placement_id;
    const leg = current.binary_leg;

    await addVolume(parentId, period, leg, points);

    current = (
      await supabase
        .from("users")
        .select("placement_id, binary_leg")
        .eq("id", parentId)
        .maybeSingle()
    ).data;
  }
}
```

### 126. `lib/engine/process-order.ts`
Procesa una orden pagada: puntos, bonos directos y volumen.
```ts
/**
 * PROCESAMIENTO DE UNA ORDEN PAGADA (por evento).
 *
 * ⚠️ SOLO SERVIDOR (cliente admin).
 *
 * Al confirmarse el pago de una orden se disparan DOS circuitos separados
 * (bases distintas, mismo evento):
 *   1. VOLUMEN: los puntos del producto suben por el árbol binario.
 *      (Aplica a inscripción de servicio, renovación y productos físicos.)
 *   2. BONO DIRECTO (Vía 1): 20% N1 / 10% N2 sobre el valor COMISIONABLE de la
 *      inscripción. Solo para inscripciones de SERVICIO (no IBO ni físicos).
 *
 * Recordatorio del usuario: la inscripción de un servicio SÍ genera puntos
 * (incluye el mes corriente), además del bono directo.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { loadEngineConfig } from "./config";
import { directBonus } from "./calculations";
import { accumulateVolume } from "./volume";
import { currentPeriod } from "./period";

export async function processPaidOrder(orderId: string): Promise<void> {
  const supabase = createAdminClient();

  const { data: order, error: orderErr } = await supabase
    .from("orders")
    .select("id, user_id, product_id, order_type, amount, payment_status")
    .eq("id", orderId)
    .single();
  if (orderErr || !order) throw new Error(`Orden no encontrada: ${orderId}`);
  if (order.payment_status !== "paid") return; // solo procesamos pagadas

  const { data: product } = await supabase
    .from("products")
    .select("type, points, fund_amount")
    .eq("id", order.product_id)
    .single();
  if (!product) throw new Error("Producto de la orden no encontrado.");

  const period = currentPeriod();

  // --- Circuito 1: VOLUMEN (sube por el binario) ---
  if (product.points > 0) {
    await accumulateVolume(order.user_id, product.points, period);
  }

  // --- Circuito 2: BONO DIRECTO (solo inscripción de servicio) ---
  if (order.order_type === "enrollment" && product.type === "service") {
    await payDirectBonus(order.user_id, Number(order.amount), Number(product.fund_amount), period);
  }
}

/** Paga el bono directo de niveles 1 y 2 sobre el valor comisionable. */
async function payDirectBonus(
  buyerId: string,
  amount: number,
  fundAmount: number,
  period: string
): Promise<void> {
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();

  // NEXUS: el capital de fondo NO es comisionable.
  const comisionable = Math.max(0, amount - fundAmount);
  const { l1, l2 } = directBonus(comisionable, cfg);

  // Patrocinador (N1) y patrocinador del patrocinador (N2).
  const { data: buyer } = await supabase
    .from("users")
    .select("sponsor_id")
    .eq("id", buyerId)
    .single();
  const sponsorId = buyer?.sponsor_id;
  if (!sponsorId) return;

  const { data: sponsor } = await supabase
    .from("users")
    .select("id, sponsor_id, can_earn")
    .eq("id", sponsorId)
    .single();

  // Nivel 1
  if (sponsor?.can_earn && l1 > 0) {
    await supabase.from("commissions").insert({
      user_id: sponsor.id,
      source_user_id: buyerId,
      period,
      type: "direct_l1",
      amount: l1,
      status: "pending",
    });
  }

  // Nivel 2
  if (sponsor?.sponsor_id) {
    const { data: grand } = await supabase
      .from("users")
      .select("id, can_earn")
      .eq("id", sponsor.sponsor_id)
      .single();
    if (grand?.can_earn && l2 > 0) {
      await supabase.from("commissions").insert({
        user_id: grand.id,
        source_user_id: buyerId,
        period,
        type: "direct_l2",
        amount: l2,
        status: "pending",
      });
    }
  }
}
```

### 127. `lib/engine/monthly-close.ts`
Cierre mensual: rango, binario, rango, matching y estilo de vida.
```ts
/**
 * ============================================================================
 * CIERRE MENSUAL — orquestación (cron / botón admin)
 * ============================================================================
 *
 * ⚠️ SOLO SERVIDOR (cliente admin).
 *
 * Corre el último día del mes, GENERAL para todos (no por fecha de ingreso).
 * Usa el núcleo puro de `calculations.ts`; aquí solo se lee/escribe en la BD.
 *
 * Pasos:
 *   1) Por cada usuario: liquidar volumen (pierna menor + carry-over), calcular
 *      rango (4 condiciones) y pagar binario (15%) + rango (10%). Inactivos
 *      pierden todo el acumulado. Sin can_earn no cobran (pero el volumen liquida).
 *   2) Matching (Vía 4) para rango ≥ 6.
 *   3) Estilo de vida (Vía 5) para rango ≥ 10.
 *
 * NOTA SOBRE LA CONDICIÓN 2 (máx % por línea de patrocinio):
 *   Atribuir el volumen de la pierna menor a cada línea directa requiere un
 *   cálculo recursivo sobre el árbol que el modelo de datos actual no precalcula.
 *   En este prototipo se usa `maxLineFraction = 0` (la condición no frena el
 *   ascenso). ⚠️ PENDIENTE para el desarrollador: implementar la atribución real
 *   por línea (idealmente con una función SQL recursiva) y confirmar la regla
 *   exacta con el negocio.
 * ============================================================================
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { loadEngineConfig, loadRanks, type RankRule } from "./config";
import { computePeriodCheck, matchingBonus, lifestyleBonus } from "./calculations";
import { currentPeriod, previousPeriod } from "./period";
import type { BinaryLeg } from "@/types/database";

export interface CloseSummary {
  period: string;
  usersProcessed: number;
  commissionsCreated: number;
}

export async function monthlyClose(
  period: string = currentPeriod()
): Promise<CloseSummary> {
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();
  const ranks = await loadRanks();
  const ranksByLevel = new Map<number, RankRule>(ranks.map((r) => [r.level, r]));
  const prev = previousPeriod(period);

  const { data: users } = await supabase
    .from("users")
    .select("id, status, can_earn, current_rank, sponsor_id");

  let commissionsCreated = 0;

  // ---- PASO 1: liquidación + binario + rango + carry-over ----
  for (const user of users ?? []) {
    const ledger = await getLedger(user.id, period);
    const prevLedger = await getLedger(user.id, prev);

    const leftTotal = (ledger?.left_volume ?? 0) + (prevLedger?.left_carryover ?? 0);
    const rightTotal = (ledger?.right_volume ?? 0) + (prevLedger?.right_carryover ?? 0);

    const directs = await countDirectsActiveByLeg(user.id);
    const inactive = user.status === "inactive";

    const check = computePeriodCheck(
      {
        leftTotal,
        rightTotal,
        maxLineFraction: 0, // ver nota de arriba (PENDIENTE)
        directsLeftActive: directs.left,
        directsRightActive: directs.right,
        canEarn: user.can_earn,
        inactive,
      },
      ranks,
      cfg
    );

    // Comisiones de binario y rango.
    if (check.binary > 0) {
      await insertCommission(user.id, null, period, "binary", check.binary);
      commissionsCreated++;
    }
    if (check.rank > 0) {
      await insertCommission(user.id, null, period, "rank", check.rank);
      commissionsCreated++;
    }

    // Rango del período + historial.
    await supabase.from("users").update({ current_rank: check.rankLevel }).eq("id", user.id);
    await supabase
      .from("rank_history")
      .upsert(
        { user_id: user.id, period, level: check.rankLevel },
        { onConflict: "user_id,period" }
      );

    // Persistir liquidación/carry-over en el ledger de ESTE período.
    await upsertLedgerSettlement(user.id, period, check.settlement);
  }

  // ---- PASO 2: Matching (Vía 4), rango ≥ MATCHING_MIN_RANK ----
  // Releemos los rangos recién calculados en el paso 1 (matching depende de ellos).
  const { data: refreshed } = await supabase
    .from("users")
    .select("id, current_rank");

  for (const u of refreshed ?? []) {
    const level = u.current_rank ?? 0;
    if (level >= cfg.MATCHING_MIN_RANK) {
      const created = await applyMatching(u.id, level, period, ranksByLevel);
      commissionsCreated += created;
    }
  }

  // ---- PASO 3: Estilo de vida (Vía 5), rango ≥ LIFESTYLE_MIN_RANK ----
  for (const u of refreshed ?? []) {
    const level = u.current_rank ?? 0;
    if (level >= cfg.LIFESTYLE_MIN_RANK) {
      const created = await applyLifestyle(u.id, level, period, ranksByLevel);
      commissionsCreated += created;
    }
  }

  return { period, usersProcessed: users?.length ?? 0, commissionsCreated };
}

// ----------------------------------------------------------------------------
// Helpers de BD
// ----------------------------------------------------------------------------

async function getLedger(userId: string, period: string) {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("volume_ledger")
    .select("left_volume, right_volume, left_carryover, right_carryover")
    .eq("user_id", userId)
    .eq("period", period)
    .maybeSingle();
  return data;
}

async function upsertLedgerSettlement(
  userId: string,
  period: string,
  settlement: { leftCarryover: number; rightCarryover: number; paidLegVolume: number }
) {
  const supabase = createAdminClient();
  const { data: existing } = await supabase
    .from("volume_ledger")
    .select("id")
    .eq("user_id", userId)
    .eq("period", period)
    .maybeSingle();

  if (existing) {
    await supabase
      .from("volume_ledger")
      .update({
        left_carryover: settlement.leftCarryover,
        right_carryover: settlement.rightCarryover,
        paid_leg_volume: settlement.paidLegVolume,
      })
      .eq("id", existing.id);
  } else {
    await supabase.from("volume_ledger").insert({
      user_id: userId,
      period,
      left_carryover: settlement.leftCarryover,
      right_carryover: settlement.rightCarryover,
      paid_leg_volume: settlement.paidLegVolume,
    });
  }
}

async function insertCommission(
  userId: string,
  sourceUserId: string | null,
  period: string,
  type: "binary" | "rank" | "matching" | "lifestyle",
  amount: number
) {
  const supabase = createAdminClient();
  await supabase.from("commissions").insert({
    user_id: userId,
    source_user_id: sourceUserId,
    period,
    type,
    amount,
    status: "pending",
  });
}

/**
 * Cuenta los directos (patrocinados) ACTIVOS de un usuario, separados por la
 * pierna binaria bajo la que caen (condición 3 del rango).
 */
async function countDirectsActiveByLeg(
  userId: string
): Promise<{ left: number; right: number }> {
  const supabase = createAdminClient();
  const { data: directs } = await supabase
    .from("users")
    .select("id")
    .eq("sponsor_id", userId)
    .eq("can_earn", true);

  let left = 0;
  let right = 0;
  for (const d of directs ?? []) {
    const leg = await findLegUnder(userId, d.id);
    if (leg === "left") left++;
    else if (leg === "right") right++;
  }
  return { left, right };
}

/**
 * Determina en qué pierna de `ancestorId` cae `descendantId` subiendo por el
 * árbol binario hasta encontrar al hijo directo del ancestro.
 */
async function findLegUnder(
  ancestorId: string,
  descendantId: string
): Promise<BinaryLeg | null> {
  const supabase = createAdminClient();
  let current: { id: string; placement_id: string | null; binary_leg: BinaryLeg | null } | null =
    (
      await supabase
        .from("users")
        .select("id, placement_id, binary_leg")
        .eq("id", descendantId)
        .maybeSingle()
    ).data;

  for (let depth = 0; depth < 100000 && current?.placement_id; depth++) {
    if (current.placement_id === ancestorId) return current.binary_leg;
    current = (
      await supabase
        .from("users")
        .select("id, placement_id, binary_leg")
        .eq("id", current.placement_id)
        .maybeSingle()
    ).data;
  }
  return null;
}

/** Devuelve los IDs de la generación `n` de la línea de PATROCINIO de un usuario. */
async function getSponsorGeneration(userId: string, n: number): Promise<string[]> {
  const supabase = createAdminClient();
  let frontier = [userId];
  for (let i = 0; i < n; i++) {
    if (frontier.length === 0) return [];
    const { data } = await supabase
      .from("users")
      .select("id")
      .in("sponsor_id", frontier);
    frontier = (data ?? []).map((u) => u.id);
  }
  return frontier;
}

/** Vía 4 — Matching sobre el binario de la línea de patrocinio. */
async function applyMatching(
  userId: string,
  level: number,
  period: string,
  ranksByLevel: Map<number, RankRule>
): Promise<number> {
  const rule = ranksByLevel.get(level);
  const levels = rule?.matching_levels ?? [];
  if (levels.length === 0) return 0;

  const supabase = createAdminClient();
  const binariesByGeneration: number[][] = [];

  for (let g = 0; g < levels.length; g++) {
    const gen = await getSponsorGeneration(userId, g + 1);
    if (gen.length === 0) {
      binariesByGeneration.push([]);
      continue;
    }
    const { data: comms } = await supabase
      .from("commissions")
      .select("amount")
      .eq("period", period)
      .eq("type", "binary")
      .in("user_id", gen);
    binariesByGeneration.push((comms ?? []).map((c) => Number(c.amount)));
  }

  const total = matchingBonus(binariesByGeneration, levels);
  if (total > 0) {
    await insertCommission(userId, null, period, "matching", total);
    return 1;
  }
  return 0;
}

/** Vía 5 — Bono de estilo de vida (rango alto mantenido 3 meses + N directos). */
async function applyLifestyle(
  userId: string,
  level: number,
  period: string,
  ranksByLevel: Map<number, RankRule>
): Promise<number> {
  const rule = ranksByLevel.get(level);
  if (!rule || rule.lifestyle_bonus <= 0) return 0;

  const supabase = createAdminClient();

  // Meses consecutivos manteniendo al menos este rango (incluyendo el actual).
  const { data: history } = await supabase
    .from("rank_history")
    .select("period, level")
    .eq("user_id", userId);
  const byPeriod = new Map<string, number>((history ?? []).map((h) => [h.period, h.level]));

  let monthsMaintained = 0;
  let p = period;
  while ((byPeriod.get(p) ?? 0) >= level) {
    monthsMaintained++;
    p = previousPeriod(p);
  }

  // Directos activos (total).
  const { count: directsActive } = await supabase
    .from("users")
    .select("*", { count: "exact", head: true })
    .eq("sponsor_id", userId)
    .eq("can_earn", true);

  const amount = lifestyleBonus(rule, monthsMaintained, directsActive ?? 0);
  if (amount > 0) {
    await insertCommission(userId, null, period, "lifestyle", amount);
    return 1;
  }
  return 0;
}
```

### 128. `lib/engine/payout.ts`
Holding y pago semanal a la billetera.
```ts
/**
 * ============================================================================
 * PAGOS — holding de validación + pago semanal
 * ============================================================================
 *
 * ⚠️ SOLO SERVIDOR (cliente admin).
 *
 * Flujo:
 *   1) runHolding(): tras ~7 días (HOLDING_DAYS) las comisiones 'pending' pasan
 *      a 'validated' (período de validación para cubrir reembolsos).
 *   2) runWeeklyPayout(period): el cheque mensual validado se divide en
 *      WEEKLY_SPLIT (4) pagos semanales. Cada corrida abona una cuota a la
 *      billetera. Al completar las 4, las comisiones del período pasan a 'paid'.
 *
 * El mínimo de $10 (MIN_WITHDRAWAL) aplica al SOLICITAR un retiro, no al abonar
 * a la billetera (ver withdrawals).
 * ============================================================================
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { loadEngineConfig } from "./config";
import { weeklyPayout, round2 } from "./calculations";

/** Pasa de 'pending' a 'validated' las comisiones que ya cumplieron el holding. */
export async function runHolding(asOf: Date = new Date()): Promise<number> {
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();

  const cutoff = new Date(asOf);
  cutoff.setDate(cutoff.getDate() - cfg.HOLDING_DAYS);

  const { data, error } = await supabase
    .from("commissions")
    .update({ status: "validated" })
    .eq("status", "pending")
    .lte("created_at", cutoff.toISOString())
    .select("id");
  if (error) throw new Error(`Error en holding: ${error.message}`);

  return data?.length ?? 0;
}

export interface PayoutSummary {
  period: string;
  usersPaid: number;
  totalDisbursed: number;
}

/**
 * Corre un pago semanal para un período: abona una cuota (cheque mensual / 4)
 * a la billetera de cada usuario con comisiones validadas.
 */
export async function runWeeklyPayout(period: string): Promise<PayoutSummary> {
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();

  // Comisiones validadas del período, agrupadas por usuario.
  const { data: comms } = await supabase
    .from("commissions")
    .select("user_id, amount")
    .eq("period", period)
    .eq("status", "validated");

  const totalsByUser = new Map<string, number>();
  for (const c of comms ?? []) {
    totalsByUser.set(c.user_id, (totalsByUser.get(c.user_id) ?? 0) + Number(c.amount));
  }

  let usersPaid = 0;
  let totalDisbursed = 0;

  for (const [userId, monthlyTotal] of totalsByUser) {
    // Asegurar el calendario del usuario para el período.
    const { data: schedule } = await supabase
      .from("payout_schedule")
      .select("id, weeks_paid")
      .eq("user_id", userId)
      .eq("period", period)
      .maybeSingle();

    const weekly = weeklyPayout(round2(monthlyTotal), cfg);
    let scheduleId = schedule?.id;
    let weeksPaid = schedule?.weeks_paid ?? 0;

    if (!scheduleId) {
      const { data: created } = await supabase
        .from("payout_schedule")
        .insert({
          user_id: userId,
          period,
          monthly_total: round2(monthlyTotal),
          weekly_amount: weekly,
          weeks_paid: 0,
        })
        .select("id")
        .single();
      scheduleId = created?.id;
    }

    // Ya se pagaron las 4 semanas: nada que hacer.
    if (weeksPaid >= cfg.WEEKLY_SPLIT) continue;

    // Abonar una cuota semanal a la billetera.
    await creditWallet(userId, weekly);
    weeksPaid += 1;
    usersPaid++;
    totalDisbursed += weekly;

    await supabase
      .from("payout_schedule")
      .update({ weeks_paid: weeksPaid })
      .eq("id", scheduleId);

    // Si se completaron las cuotas, marcar las comisiones del período como pagadas.
    if (weeksPaid >= cfg.WEEKLY_SPLIT) {
      await supabase
        .from("commissions")
        .update({ status: "paid" })
        .eq("user_id", userId)
        .eq("period", period)
        .eq("status", "validated");
    }
  }

  return { period, usersPaid, totalDisbursed: round2(totalDisbursed) };
}

/** Abona un monto a la billetera del usuario (saldo + total ganado). */
async function creditWallet(userId: string, amount: number): Promise<void> {
  const supabase = createAdminClient();
  const { data: wallet } = await supabase
    .from("wallet")
    .select("id, balance, total_earned")
    .eq("user_id", userId)
    .maybeSingle();

  if (!wallet) {
    await supabase.from("wallet").insert({
      user_id: userId,
      balance: amount,
      total_earned: amount,
    });
    return;
  }

  await supabase
    .from("wallet")
    .update({
      balance: round2(Number(wallet.balance) + amount),
      total_earned: round2(Number(wallet.total_earned) + amount),
    })
    .eq("id", wallet.id);
}
```


---

# PARTE 13 — BASE DE DATOS
Migraciones en orden. La 019 (catálogo de 380 modelos de IA, ~45 KB de datos generados) se omite aquí: se regenera con `scripts/gen-catalogo.mjs`.

### 129. `db/migrations/001_schema.sql`
Esquema base: users, products, memberships, orders, volume_ledger, commissions, wallet, withdrawals, ranks, compensation_config.
```sql
-- ============================================================================
-- TRIADA Business School — Back Office MLM
-- Migración 001: Esquema completo de base de datos
-- Motor: PostgreSQL (Supabase). Pegar en: Dashboard > SQL Editor > New query.
-- ============================================================================
--
-- NOTAS DE DISEÑO (léelas, desarrollador externo):
--
-- 1. AUTH: el stack usa Supabase Auth. Las credenciales viven en `auth.users`
--    (gestionadas por Supabase), por eso `public.users` NO almacena password_hash
--    (a diferencia del modelo de datos crudo). `public.users.id` referencia a
--    `auth.users.id`: cada miembro = una cuenta de auth.
--
-- 2. DOS ÁRBOLES: `sponsor_id` (patrocinio → bono directo + matching) y
--    `placement_id` + `binary_leg` (binario → volumen). PUEDEN DIFERIR (spillover).
--
-- 3. LAS 3 LLAVES: has_position / can_distribute / can_earn son interruptores
--    INDEPENDIENTES. Su lógica de activación vive en la app (Fase B), no en la BD.
--
-- 4. RLS: se habilita Row Level Security en todas las tablas (postura segura por
--    defecto en Supabase). Las POLÍTICAS de acceso se añaden en la Fase B (auth).
--    Mientras tanto, solo la service_role key (motor/admin) puede leer/escribir.
--
-- 5. DINERO: DECIMAL(14,2). PUNTOS/VOLUMEN: INTEGER. Nunca float para dinero.
-- ============================================================================

-- ----------------------------------------------------------------------------
-- TIPOS ENUM
-- ----------------------------------------------------------------------------
create type binary_leg_t        as enum ('left', 'right');
create type user_status_t       as enum ('active', 'inactive', 'suspended');
create type membership_type_t   as enum ('ibo', 'service');
create type membership_status_t as enum ('active', 'expired', 'cancelled');
create type product_type_t      as enum ('service', 'physical', 'ibo');
create type order_type_t        as enum ('enrollment', 'renewal', 'product');
create type payment_status_t    as enum ('pending', 'paid', 'refunded');
create type commission_type_t   as enum ('direct_l1', 'direct_l2', 'binary', 'rank', 'matching', 'lifestyle');
create type commission_status_t as enum ('pending', 'validated', 'paid', 'rejected');
create type withdrawal_status_t as enum ('pending', 'approved', 'paid', 'rejected');

-- ----------------------------------------------------------------------------
-- FUNCIÓN: trigger genérico de updated_at
-- ----------------------------------------------------------------------------
create or replace function set_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

-- ----------------------------------------------------------------------------
-- TABLA: products (catálogo de programas, productos físicos e IBO)
-- Se crea primero: orders la referencia.
-- ----------------------------------------------------------------------------
create table products (
  id                uuid primary key default gen_random_uuid(),
  code              text unique not null,        -- NOVA, QUANTUM, GORRA, IBO...
  name              text not null,
  type              product_type_t not null,
  enrollment_price  decimal(14,2) not null,      -- precio de inscripción / primera compra
  renewal_price     decimal(14,2),               -- renovación (null si no aplica)
  points            integer not null default 0,  -- puntos que otorga
  duration_months   integer not null default 1,  -- 1, 3, 6, 12 (servicios)
  activates_earning boolean not null default false, -- true si >= 100 pts (Quantum+)
  fund_amount       decimal(14,2) not null default 0, -- capital de fondo NO comisionable (Nexus=500)
  is_active         boolean not null default true,
  created_at        timestamptz not null default now()
);

-- ----------------------------------------------------------------------------
-- TABLA: users (distribuidores y clientes)
-- ----------------------------------------------------------------------------
create table users (
  id              uuid primary key references auth.users(id) on delete cascade,
  username        text unique,
  email           text unique not null,
  full_name       text,
  country         text,
  phone           text,
  -- Árbol de patrocinio (quién refirió):
  sponsor_id      uuid references users(id),
  -- Árbol binario (posición física):
  placement_id    uuid references users(id),
  binary_leg      binary_leg_t,
  -- Las 3 llaves de activación (interruptores INDEPENDIENTES):
  has_position    boolean not null default false, -- llave 1: compró algo → posición + puntos
  can_distribute  boolean not null default false, -- llave 2: IBO activo → puede referir
  can_earn        boolean not null default false, -- llave 3: servicio >=100pts + IBO → cobra red
  referral_code   text unique not null,           -- para su link de distribución
  current_rank    integer not null default 0,     -- 0 = sin rango, 1-13
  kyc_verified    boolean not null default false,
  is_admin        boolean not null default false, -- rol admin (acceso al back office)
  status          user_status_t not null default 'active',
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now()
);

create trigger trg_users_updated_at
  before update on users
  for each row execute function set_updated_at();

-- ----------------------------------------------------------------------------
-- TABLA: memberships (estado de IBO y servicios por usuario)
-- ----------------------------------------------------------------------------
create table memberships (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references users(id) on delete cascade,
  type          membership_type_t not null,
  program_code  text references products(code),   -- NOVA, QUANTUM... (si type=service)
  points        integer not null default 0,
  status        membership_status_t not null default 'active',
  starts_at     timestamptz not null default now(),
  expires_at    timestamptz,                       -- fin del período cubierto
  auto_renew    boolean not null default true,
  created_at    timestamptz not null default now()
);

-- ----------------------------------------------------------------------------
-- TABLA: orders (compras)
-- ----------------------------------------------------------------------------
create table orders (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null references users(id) on delete cascade,
  product_id      uuid not null references products(id),
  order_type      order_type_t not null,
  amount          decimal(14,2) not null,
  points_awarded  integer not null default 0,
  is_promotional  boolean not null default false,  -- Mes Nitro
  payment_status  payment_status_t not null default 'pending',
  payment_method  text,                            -- USDT, etc.
  tx_hash         text,                            -- hash de transacción cripto
  created_at      timestamptz not null default now()
);

-- ----------------------------------------------------------------------------
-- TABLA: volume_ledger (libro de volumen/puntos por período binario)
-- Un registro por (usuario, período de cierre, ej. '2026-06').
-- ----------------------------------------------------------------------------
create table volume_ledger (
  id               uuid primary key default gen_random_uuid(),
  user_id          uuid not null references users(id) on delete cascade,
  period           text not null,                  -- 'YYYY-MM' (mes de cierre)
  left_volume      integer not null default 0,     -- puntos acumulados pierna izquierda
  right_volume     integer not null default 0,     -- puntos acumulados pierna derecha
  left_carryover   integer not null default 0,     -- arrastre del cierre anterior
  right_carryover  integer not null default 0,
  paid_leg_volume  integer not null default 0,     -- volumen de pierna menor pagado
  created_at       timestamptz not null default now(),
  unique (user_id, period)
);

-- ----------------------------------------------------------------------------
-- TABLA: promotional_points (puntos temporales — Mes Nitro)
-- ----------------------------------------------------------------------------
create table promotional_points (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references users(id) on delete cascade,
  bonus_points  integer not null,                  -- ej. +50 para llegar a 100
  reason        text not null,                     -- 'mes_nitro'
  expires_at    timestamptz not null,              -- 30 días desde activación
  active        boolean not null default true,
  created_at    timestamptz not null default now()
);

-- ----------------------------------------------------------------------------
-- TABLA: commissions (comisiones generadas — las 5 vías)
-- ----------------------------------------------------------------------------
create table commissions (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid not null references users(id) on delete cascade, -- quién la gana
  source_user_id uuid references users(id),         -- de quién proviene (null en binario/rango)
  period         text not null,                      -- 'YYYY-MM'
  type           commission_type_t not null,
  amount         decimal(14,2) not null,
  status         commission_status_t not null default 'pending',
  created_at     timestamptz not null default now()
);

-- ----------------------------------------------------------------------------
-- TABLA: wallet (billetera por usuario — solo comisiones)
-- IMPORTANTE: el Fondo de Rentabilización NUNCA se mezcla aquí.
-- ----------------------------------------------------------------------------
create table wallet (
  id              uuid primary key default gen_random_uuid(),
  user_id         uuid not null unique references users(id) on delete cascade,
  balance         decimal(14,2) not null default 0,
  total_earned    decimal(14,2) not null default 0,
  total_withdrawn decimal(14,2) not null default 0,
  updated_at      timestamptz not null default now()
);

create trigger trg_wallet_updated_at
  before update on wallet
  for each row execute function set_updated_at();

-- ----------------------------------------------------------------------------
-- TABLA: withdrawals (solicitudes de retiro — USDT)
-- ----------------------------------------------------------------------------
create table withdrawals (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid not null references users(id) on delete cascade,
  amount         decimal(14,2) not null,            -- mínimo $10 (validar en app)
  wallet_address text not null,                      -- dirección USDT del usuario
  status         withdrawal_status_t not null default 'pending',
  tx_hash        text,
  requested_at   timestamptz not null default now(),
  processed_at   timestamptz
);

-- ----------------------------------------------------------------------------
-- TABLA: ranks (configuración de los 13 rangos — editable por admin)
-- ----------------------------------------------------------------------------
create table ranks (
  level             integer primary key,             -- 1-13
  name              text not null,
  zone              text,                             -- EMPRESARIO / PREMIER / DIAMANTE / LEYENDA
  points_per_leg    integer not null,                -- mínimo en AMBAS piernas
  max_pct_line      decimal(4,2) not null,           -- 1.00, 0.80, 0.70...
  min_directs_left  integer not null,
  min_directs_right integer not null,
  base_check        decimal(14,2) not null,          -- cheque base de referencia
  matching_levels   jsonb not null default '[]',     -- [0.50, 0.10, 0.08...]
  lifestyle_bonus   decimal(14,2) not null default 0,
  lifestyle_directs integer not null default 0
);

-- ----------------------------------------------------------------------------
-- TABLA: compensation_config (constantes del plan — EDITABLE SIN CÓDIGO)
-- Clave-valor en JSONB. El motor de comisiones lee de aquí, NO hardcodea.
-- ----------------------------------------------------------------------------
create table compensation_config (
  key         text primary key,                       -- POINT_VALUE, BINARY_PCT...
  value       jsonb not null,                          -- numérico o texto
  description text,
  updated_at  timestamptz not null default now()
);

create trigger trg_config_updated_at
  before update on compensation_config
  for each row execute function set_updated_at();

-- ============================================================================
-- ÍNDICES RECOMENDADOS
-- ============================================================================
create index idx_users_sponsor       on users(sponsor_id);
create index idx_users_placement      on users(placement_id);
create index idx_users_referral_code  on users(referral_code);
create index idx_volume_ledger_period on volume_ledger(user_id, period);
create index idx_commissions_period   on commissions(user_id, period);
create index idx_commissions_status   on commissions(status);
create index idx_memberships_status   on memberships(user_id, status);
create index idx_orders_user          on orders(user_id);

-- ============================================================================
-- ROW LEVEL SECURITY (postura segura por defecto)
-- Políticas de acceso por rol se añaden en la Fase B (auth).
-- Por ahora: solo service_role (motor/admin) accede; anon/authenticated bloqueados.
-- ============================================================================
alter table users               enable row level security;
alter table memberships         enable row level security;
alter table products            enable row level security;
alter table orders              enable row level security;
alter table volume_ledger       enable row level security;
alter table promotional_points  enable row level security;
alter table commissions         enable row level security;
alter table wallet              enable row level security;
alter table withdrawals         enable row level security;
alter table ranks               enable row level security;
alter table compensation_config enable row level security;

-- FIN MIGRACIÓN 001
```

### 130. `db/migrations/002_rls_policies.sql`
Políticas de seguridad por fila: cada afiliado solo ve lo suyo.
```sql
-- ============================================================================
-- TRIADA — Migración 002: Políticas de seguridad (RLS)
-- Ejecutar DESPUÉS de 001_schema.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
--
-- Modelo de acceso:
--  - Catálogo (products, ranks, compensation_config): lectura para autenticados.
--  - Datos personales: cada usuario ve SOLO lo suyo; el admin ve todo.
--  - ESCRITURAS: se hacen desde el servidor con la service_role key (que salta
--    RLS) mediante server actions validadas. Por eso aquí solo definimos SELECT.
--  - Lecturas de red/árbol (downline): también vía servidor con service_role.
--
-- La función is_admin() es SECURITY DEFINER: corre como owner y salta RLS por
-- dentro, evitando recursión al consultar la propia tabla users.
-- ============================================================================

-- ----------------------------------------------------------------------------
-- Función auxiliar: ¿el usuario actual es admin?
-- ----------------------------------------------------------------------------
create or replace function public.is_admin()
returns boolean
language sql
security definer
stable
set search_path = public
as $$
  select coalesce((select is_admin from public.users where id = auth.uid()), false);
$$;

-- ----------------------------------------------------------------------------
-- CATÁLOGO — lectura para cualquier usuario autenticado
-- ----------------------------------------------------------------------------
create policy "products_read"  on products            for select to authenticated using (true);
create policy "ranks_read"     on ranks               for select to authenticated using (true);
create policy "config_read"    on compensation_config for select to authenticated using (true);

-- ----------------------------------------------------------------------------
-- USERS — cada quien ve su propia fila; el admin ve todas
-- ----------------------------------------------------------------------------
create policy "users_select_own_or_admin" on users
  for select to authenticated
  using (id = auth.uid() or public.is_admin());

-- El usuario puede actualizar campos de su propio perfil (datos de contacto).
-- (Las 3 llaves, rango, etc. solo las cambia el servidor con service_role.)
create policy "users_update_own" on users
  for update to authenticated
  using (id = auth.uid())
  with check (id = auth.uid());

-- ----------------------------------------------------------------------------
-- DATOS PERSONALES — solo del dueño (o admin). Solo lectura desde el cliente.
-- ----------------------------------------------------------------------------
create policy "memberships_own_or_admin" on memberships
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

create policy "orders_own_or_admin" on orders
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

create policy "volume_own_or_admin" on volume_ledger
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

create policy "promo_own_or_admin" on promotional_points
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

create policy "commissions_own_or_admin" on commissions
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

create policy "wallet_own_or_admin" on wallet
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

create policy "withdrawals_own_or_admin" on withdrawals
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

-- FIN MIGRACIÓN 002
```

### 131. `db/migrations/003_rank_history.sql`
Historial de rangos por período.
```sql
-- ============================================================================
-- TRIADA — Migración 003: Historial de rangos
-- Ejecutar DESPUÉS de 002_rls_policies.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
--
-- Guarda el rango alcanzado por cada usuario en cada cierre mensual. Sirve para:
--  - La regla "mantener el rango 3 meses consecutivos" del bono de estilo de vida.
--  - La pantalla "Mi Rango > Historial" del distribuidor (Fase D).
-- ============================================================================

create table rank_history (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references users(id) on delete cascade,
  period     text not null,                 -- 'YYYY-MM'
  level      integer not null,              -- rango alcanzado (0-13)
  created_at timestamptz not null default now(),
  unique (user_id, period)
);

create index idx_rank_history_user on rank_history(user_id, period);

alter table rank_history enable row level security;

create policy "rank_history_own_or_admin" on rank_history
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

-- FIN MIGRACIÓN 003
```

### 132. `db/migrations/004_payout_schedule.sql`
Calendario de pagos y holding.
```sql
-- ============================================================================
-- TRIADA — Migración 004: Calendario de pagos semanales
-- Ejecutar DESPUÉS de 003_rank_history.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
--
-- El cheque mensual validado de cada usuario se divide en N pagos semanales
-- (WEEKLY_SPLIT = 4). Esta tabla registra el progreso de esos pagos por período.
-- Cuando se completan las 4 semanas, las comisiones del período pasan a 'paid'.
-- ============================================================================

create table payout_schedule (
  id            uuid primary key default gen_random_uuid(),
  user_id       uuid not null references users(id) on delete cascade,
  period        text not null,                  -- 'YYYY-MM'
  monthly_total decimal(14,2) not null,         -- suma de comisiones validadas del mes
  weekly_amount decimal(14,2) not null,         -- monthly_total / WEEKLY_SPLIT
  weeks_paid    integer not null default 0,     -- semanas ya pagadas (0..WEEKLY_SPLIT)
  created_at    timestamptz not null default now(),
  updated_at    timestamptz not null default now(),
  unique (user_id, period)
);

create index idx_payout_schedule_user on payout_schedule(user_id, period);

create trigger trg_payout_schedule_updated_at
  before update on payout_schedule
  for each row execute function set_updated_at();

alter table payout_schedule enable row level security;

create policy "payout_schedule_own_or_admin" on payout_schedule
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

-- FIN MIGRACIÓN 004
```

### 133. `db/migrations/006_promotions.sql`
Promociones temporales.
```sql
-- ============================================================================
-- TRIADA — Migración 006: Promociones
-- Ejecutar DESPUÉS de 005_audit_log.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
--
-- Promociones temporales activables por fecha (Mes Nitro, Descuento de Bienvenida).
-- El motor/registro consultan si una promoción está activa para aplicar su efecto.
-- ============================================================================

create table promotions (
  key        text primary key,            -- 'mes_nitro', 'descuento_bienvenida'
  name       text not null,
  active     boolean not null default false,
  starts_at  timestamptz,
  ends_at    timestamptz,
  config     jsonb not null default '{}', -- parámetros (ej. bonus_points, pct_descuento)
  updated_at timestamptz not null default now()
);

create trigger trg_promotions_updated_at
  before update on promotions
  for each row execute function set_updated_at();

alter table promotions enable row level security;

create policy "promotions_read" on promotions
  for select to authenticated using (true);

-- Seed de las dos promociones base.
insert into promotions (key, name, active, config) values
  ('mes_nitro', 'Mes Nitro', false, '{"bonus_points": 50, "duration_days": 30}'),
  ('descuento_bienvenida', 'Descuento de Bienvenida', false, '{"pct": 0.20}')
on conflict (key) do nothing;

-- FIN MIGRACIÓN 006
```

### 134. `db/migrations/007_user_profile_fields.sql`
Campos de perfil.
```sql
-- ============================================================================
-- TRIADA — Migración 007: Campos de perfil del registro
-- Ejecutar DESPUÉS de 006_promotions.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
--
-- Agrega los datos que ahora se piden en el registro:
-- fecha de nacimiento, ciudad y código postal. (country y phone ya existían.)
-- ============================================================================

alter table users add column if not exists birth_date  date;
alter table users add column if not exists city        text;
alter table users add column if not exists postal_code text;

-- FIN MIGRACIÓN 007
```

### 135. `db/migrations/008_user_gender_address.sql`
Género y dirección.
```sql
-- ============================================================================
-- TRIADA — Migración 008: Género y dirección
-- Ejecutar DESPUÉS de 007_user_profile_fields.sql. Pegar en: SQL Editor.
-- ============================================================================

alter table users add column if not exists gender  text;   -- 'M' | 'F' | 'X' (no definido)
alter table users add column if not exists address text;

-- FIN MIGRACIÓN 008
```

### 136. `db/migrations/009_wallet_addresses.sql`
Direcciones USDT del afiliado.
```sql
-- ============================================================================
-- TRIADA — Migración 009: Lista de carteras (direcciones USDT guardadas)
-- Ejecutar DESPUÉS de 008_user_gender_address.sql. Pegar en: SQL Editor.
-- ============================================================================
--
-- Cada usuario puede guardar varias direcciones de cobro (USDT TRC-20 / BEP-20)
-- y marcar una como "más usada" (predeterminada) para sus retiros.
-- ============================================================================

create table wallet_addresses (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references users(id) on delete cascade,
  address    text not null,
  currency   text not null,                 -- 'USDT.TRC-20' | 'USDT.BEP-20'
  is_default boolean not null default false,
  created_at timestamptz not null default now()
);

create index idx_wallet_addresses_user on wallet_addresses(user_id);

alter table wallet_addresses enable row level security;

create policy "wallet_addresses_own_or_admin" on wallet_addresses
  for select to authenticated
  using (user_id = auth.uid() or public.is_admin());

-- FIN MIGRACIÓN 009
```

### 137. `db/migrations/010_avatar.sql`
Foto de perfil (bucket de Storage).
```sql
-- ============================================================================
-- TRIADA — Migración 010: Foto de perfil
-- Ejecutar DESPUÉS de 009_wallet_addresses.sql. Pegar en: SQL Editor.
-- ============================================================================
-- El bucket de Storage "avatars" se crea automáticamente desde el código
-- (público), por lo que aquí solo agregamos la columna con la URL de la foto.
-- ============================================================================

alter table users add column if not exists avatar_url text;

-- FIN MIGRACIÓN 010
```

### 138. `db/migrations/011_ai_hub.sql`
Hub de IA: modelos, saldo de créditos y uso.
```sql
-- ============================================================================
-- TRIADA — Migración 011: Hub de IA (Central de IA) — Fase F1
-- Ejecutar DESPUÉS de 010_avatar.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- Chat multi-modelo por créditos (gateway: OpenRouter). El navegador NUNCA
-- llama al gateway; una ruta del servidor valida saldo → llama → registra uso
-- → descuenta créditos. El catálogo de modelos es editable (como ranks/config).
-- ============================================================================

-- Catálogo de modelos disponibles (editable por admin).
create table if not exists ai_models (
  id          uuid primary key default gen_random_uuid(),
  code        text unique not null,          -- id de OpenRouter (ej. anthropic/claude-opus-5.5)
  name        text not null,                 -- nombre visible
  provider    text not null,                 -- Anthropic, OpenAI, DeepSeek, TRIADA...
  credit_in   numeric not null default 0,    -- créditos por 1K tokens de ENTRADA
  credit_out  numeric not null default 0,    -- créditos por 1K tokens de SALIDA
  tier        text not null default '',      -- indicador visual: '', '$', '$$', '$$$', '$$$$', 'Gratis'
  category    text not null default 'chat',  -- chat | image | video ... (para fases futuras)
  is_default  boolean not null default false,
  is_active   boolean not null default true,
  sort        integer not null default 0,
  created_at  timestamptz not null default now()
);

-- Saldo de créditos por usuario.
create table if not exists ai_credits (
  user_id    uuid primary key references users(id) on delete cascade,
  balance    numeric not null default 0,
  updated_at timestamptz not null default now()
);

-- Registro de uso por llamada (para métricas y cobro).
create table if not exists ai_usage (
  id                uuid primary key default gen_random_uuid(),
  user_id           uuid not null references users(id) on delete cascade,
  model_code        text not null,
  prompt_tokens     integer not null default 0,
  completion_tokens integer not null default 0,
  credits_spent     numeric not null default 0,
  created_at        timestamptz not null default now()
);
create index if not exists idx_ai_usage_user on ai_usage(user_id, created_at);

alter table ai_models  enable row level security;
alter table ai_credits enable row level security;
alter table ai_usage   enable row level security;

create policy "ai_models_read"        on ai_models  for select to authenticated using (true);
create policy "ai_credits_own_or_adm" on ai_credits for select to authenticated using (user_id = auth.uid() or public.is_admin());
create policy "ai_usage_own_or_adm"   on ai_usage   for select to authenticated using (user_id = auth.uid() or public.is_admin());

-- ----------------------------------------------------------------------------
-- Semilla de modelos (IDs y precios reales de OpenRouter, 2026-09).
-- credit_in / credit_out: créditos por 1K tokens (margen ya incluido).
-- "TRIADA Flash" = router de modelos gratis (openrouter/free): no gasta créditos.
-- ----------------------------------------------------------------------------
insert into ai_models (code, name, provider, credit_in, credit_out, tier, is_default, sort) values
  ('openrouter/free',                    'TRIADA Flash',    'TRIADA',   0, 0,  'Gratis', true,  1),
  ('deepseek/deepseek-chat-v3.1',        'DeepSeek V3.1',   'DeepSeek', 1, 2,  '$',      false, 2),
  ('meta-llama/llama-3.3-70b-instruct',  'Llama 3.3 70B',   'Meta',     1, 1,  '$',      false, 3),
  ('google/gemma-4-31b-it:free',         'Gemma 4 31B',     'Google',   0, 0,  'Gratis', false, 4),
  ('qwen/qwen3.8-27b:free',              'Qwen3.8 27B',     'Qwen',     0, 0,  'Gratis', false, 5),
  ('deepseek/deepseek-r1-0528',          'DeepSeek R1',     'DeepSeek', 1, 5,  '$$',     false, 6),
  ('openai/gpt-4o-2024-11-20',           'GPT-4o',          'OpenAI',   5, 20, '$$$',    false, 7),
  ('x-ai/grok-4.7',                      'Grok 4.7',        'xAI',      4, 10, '$$$',    false, 8),
  ('anthropic/claude-opus-5.5',          'Claude Opus 5.5', 'Anthropic',8, 40, '$$$$',   false, 9)
on conflict (code) do update set
  name = excluded.name, provider = excluded.provider, credit_in = excluded.credit_in,
  credit_out = excluded.credit_out, tier = excluded.tier, is_default = excluded.is_default,
  sort = excluded.sort;

-- Saldo inicial de prueba: 10.000 créditos a todos los usuarios actuales.
-- (En Fase F4 los créditos vendrán del paquete comprado y de la tienda.)
insert into ai_credits (user_id, balance)
select id, 10000 from users
on conflict (user_id) do nothing;

-- FIN MIGRACIÓN 011
```

### 139. `db/migrations/012_ai_assistants_conversations.sql`
Asistentes y conversaciones persistentes.
```sql
-- ============================================================================
-- TRIADA — Migración 012: Asistentes (F2) + persistencia de conversaciones
-- Ejecutar DESPUÉS de 011_ai_hub.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- - ai_assistants: "Mis Ayudantes" = presets de chat con cerebro (system_prompt).
-- - ai_conversations / ai_messages: respaldo de cada chat del usuario (lista de
--   proyectos, renombrables; sin título = "Sin proyecto").
-- ============================================================================

create table if not exists ai_assistants (
  id            uuid primary key default gen_random_uuid(),
  code          text unique not null,
  name          text not null,
  description   text not null default '',
  icon          text not null default 'sparkles',  -- nombre de icono (mapa en UI)
  category      text not null default 'General',
  system_prompt text not null,                      -- el "cerebro" del asistente
  model_code    text,                               -- modelo sugerido (opcional)
  sort          integer not null default 0,
  is_active     boolean not null default true,
  created_at    timestamptz not null default now()
);

create table if not exists ai_conversations (
  id             uuid primary key default gen_random_uuid(),
  user_id        uuid not null references users(id) on delete cascade,
  title          text,                    -- null = "Sin proyecto"
  assistant_code text,
  model_code     text,
  created_at     timestamptz not null default now(),
  updated_at     timestamptz not null default now()
);
create index if not exists idx_ai_conv_user on ai_conversations(user_id, updated_at desc);

create table if not exists ai_messages (
  id              uuid primary key default gen_random_uuid(),
  conversation_id uuid not null references ai_conversations(id) on delete cascade,
  role            text not null,           -- user | assistant
  content         text not null,
  created_at      timestamptz not null default now()
);
create index if not exists idx_ai_msg_conv on ai_messages(conversation_id, created_at);

alter table ai_assistants    enable row level security;
alter table ai_conversations enable row level security;
alter table ai_messages      enable row level security;

create policy "ai_assistants_read" on ai_assistants for select to authenticated using (true);
create policy "ai_conv_own_or_adm" on ai_conversations for select to authenticated using (user_id = auth.uid() or public.is_admin());
create policy "ai_msg_own_or_adm" on ai_messages for select to authenticated using (
  exists (select 1 from ai_conversations c where c.id = conversation_id and (c.user_id = auth.uid() or public.is_admin()))
);

-- Semilla de asistentes (el contexto base de TRIADA se antepone en el servidor).
insert into ai_assistants (code, name, description, icon, category, system_prompt, sort) values
  ('mentor-triada', 'Mentor TRIADA', 'Define metas y arma tu plan de crecimiento en el modelo TRIADA.', 'compass', 'Negocio',
   'Actua como un mentor de negocios de TRIADA. Ayuda al distribuidor a definir metas claras, armar un plan de accion por etapas y avanzar en su organizacion y rangos. Haz preguntas para entender su situacion antes de aconsejar y entrega pasos concretos y medibles.', 1),
  ('copy-ads', 'Copywriter de Anuncios', 'Anuncios que convierten para Meta, Google y TikTok Ads.', 'megaphone', 'Marketing',
   'Actua como un copywriter experto en anuncios de respuesta directa para Meta Ads, Google Ads y TikTok Ads. Escribe hooks, cuerpos y llamados a la accion persuasivos. Ofrece siempre 2 o 3 variantes y explica el angulo psicologico de cada una.', 2),
  ('estratega-marketing', 'Estratega de Marketing', 'Disena tu embudo y plan de marketing digital.', 'target', 'Marketing',
   'Actua como un estratega de marketing digital. Disena embudos de venta, calendarios de contenido y estrategias de adquisicion para promover los programas de TRIADA de forma etica, sin promesas de dinero facil. Prioriza tacticas accionables segun el presupuesto del usuario.', 3),
  ('creador-contenido', 'Creador de Contenido', 'Ideas, ganchos y guiones para tus redes.', 'clapperboard', 'Contenido',
   'Actua como un creador de contenido para redes sociales. Genera ideas, ganchos y guiones cortos para Reels, TikTok y Shorts, ademas de captions con llamado a la accion. Adapta el tono a la audiencia del usuario.', 4),
  ('analista-competencia', 'Analista de Competencia', 'Investiga tu nicho y encuentra tu ventaja.', 'search', 'Analisis',
   'Actua como un analista de mercado y competencia. Ayuda a investigar nichos, comparar competidores y encontrar oportunidades de diferenciacion. Estructura el analisis en fortalezas, debilidades y huecos de mercado.', 5),
  ('experto-prospeccion', 'Experto en Prospeccion', 'Mensajes y scripts para invitar sin sonar a spam.', 'users', 'Ventas',
   'Actua como un experto en prospeccion para marketing de recomendacion. Escribe mensajes de invitacion naturales y respetuosos (nunca spam), scripts para responder objeciones y secuencias de seguimiento. La marca habla de distribucion y organizacion, nunca de referido.', 6)
on conflict (code) do update set
  name = excluded.name, description = excluded.description, icon = excluded.icon,
  category = excluded.category, system_prompt = excluded.system_prompt, sort = excluded.sort;

-- FIN MIGRACIÓN 012
```

### 140. `db/migrations/013_creative_hub.sql`
Creaciones (imagen, audio, video).
```sql
-- ============================================================================
-- TRIADA — Migración 013: Creative Hub (F3) — generación de imágenes
-- Ejecutar DESPUÉS de 012. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- Reutiliza el gateway OpenRouter (modelos de imagen de Google/OpenAI) y el
-- sistema de créditos. Las creaciones se guardan en Storage + tabla ai_creations.
-- Para modelos de imagen usamos credit_out como "créditos por imagen" (plano).
-- ============================================================================

create table if not exists ai_creations (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references users(id) on delete cascade,
  kind       text not null default 'image',    -- image | video | audio ... (futuro)
  prompt     text not null default '',
  model_code text,
  url        text not null,
  created_at timestamptz not null default now()
);
create index if not exists idx_ai_creations_user on ai_creations(user_id, created_at desc);

alter table ai_creations enable row level security;
create policy "ai_creations_own_or_adm" on ai_creations
  for select to authenticated using (user_id = auth.uid() or public.is_admin());

-- Modelos de IMAGEN (category='image'). credit_out = créditos por imagen.
insert into ai_models (code, name, provider, credit_in, credit_out, tier, category, is_default, sort) values
  ('google/gemini-3.1-flash-image', 'TRIADA Imagen', 'Google', 0, 15, '$',   'image', true,  101),
  ('google/gemini-3-pro-image',     'Imagen Pro',    'Google', 0, 40, '$$',  'image', false, 102),
  ('openai/gpt-5-image',            'GPT Image',     'OpenAI', 0, 60, '$$$', 'image', false, 103)
on conflict (code) do update set
  name = excluded.name, provider = excluded.provider, credit_out = excluded.credit_out,
  tier = excluded.tier, category = excluded.category, sort = excluded.sort;

-- FIN MIGRACIÓN 013
```

### 141. `db/migrations/014_fixed_cost_router.sql`
Costo fijo por tarea y ledger de créditos.
```sql
-- ============================================================================
-- TRIADA — Migración 014: Costo fijo (Abacus-híbrido) + Router LLM
-- Ejecutar DESPUÉS de 013. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- Estrategia: costo FIJO por tipo de tarea (no per-token) con margen 3x ya
-- embebido. Router semántico elige modelo económico vs premium ANTES de llamar.
--
-- UNIFICADO: el saldo sigue en `ai_credits` (NO se crea otra tabla de balance).
-- Se agregan la matriz de tareas y un ledger de transacciones (consumo/recarga/
-- afiliación) que servirá para la tienda de créditos (F4).
-- Equivalencia: $1 USD = 1.000 créditos TRIADA.
-- ============================================================================

-- Matriz de costos fijos + modelo destino por tarea (editable sin código).
create table if not exists ia_tasks_config (
  id                text primary key,          -- llave lógica: economy-text, premium-text...
  task_name         text not null,
  category          text not null,             -- texto | imagen | agente | audio
  fixed_credit_cost numeric not null,          -- costo plano (1, 15, 100, 500)
  model_code        text                       -- id OpenRouter destino (router)
);

-- Ledger inmutable de movimientos de créditos.
create table if not exists ia_credits_transactions (
  id         uuid primary key default gen_random_uuid(),
  user_id    uuid not null references users(id) on delete cascade,
  type       text not null,                    -- consumo | recarga | afiliacion
  amount     numeric not null,                 -- delta (negativo=consumo, positivo=recarga)
  metadata   jsonb not null default '{}'::jsonb, -- router_selected_model, prompt_preview, task_id
  created_at timestamptz not null default now()
);
create index if not exists idx_ia_tx_user on ia_credits_transactions(user_id, created_at desc);

alter table ia_tasks_config          enable row level security;
alter table ia_credits_transactions  enable row level security;
create policy "ia_tasks_read" on ia_tasks_config for select to authenticated using (true);
create policy "ia_tx_own_or_adm" on ia_credits_transactions
  for select to authenticated using (user_id = auth.uid() or public.is_admin());

-- Semilla de tareas. model_code elegido para funcionar SIN saldo de OpenRouter
-- ahora (modelos free); premium-image necesita saldo (imágenes no son free).
-- Editable luego: apunta premium-text a Claude/GPT cuando cargues créditos.
insert into ia_tasks_config (id, task_name, category, fixed_credit_cost, model_code) values
  ('economy-text',     'Consulta Automática Económica',        'texto',  1,   'openrouter/free'),
  ('premium-text',     'Razonamiento y Código Avanzado',       'texto',  15,  'z-ai/glm-5.2:free'),
  ('premium-image',    'Generación de Imagen de Alta Definición','imagen', 100, 'google/gemini-3.1-flash-image'),
  ('premium-video',    'Generación de Video con IA',           'video',  300, null),
  ('autonomous-agent', 'Corrida de Agente / Automatización',   'agente', 500, null)
on conflict (id) do update set
  task_name = excluded.task_name, category = excluded.category,
  fixed_credit_cost = excluded.fixed_credit_cost, model_code = excluded.model_code;

-- FIN MIGRACIÓN 014
```

### 142. `db/migrations/015_paquetes_creditos.sql`
Paquetes con créditos mensuales y nivel.
```sql
-- ============================================================================
-- TRIADA — Migración 015: Reestructuración de paquetes + créditos por paquete
-- Ejecutar DESPUÉS de 014. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- 4 paquetes base: Básico $20, Estándar $50, Nova $125, Quantum $200.
-- + derivados de Nova (trimestral/semestral/anual, pago adelantado).
-- Elimina: NEXUS y los Quantum trimestral/semestral/anual.
-- Cada paquete otorga créditos IA mensuales; can_earn se activa a >=100 pts
-- (Nova y Quantum). tier_level: 1=Básico 2=Estándar 3=Nova 4=Quantum (gating).
-- ============================================================================

alter table products add column if not exists ai_credits_monthly integer not null default 0;
alter table products add column if not exists tier_level        integer not null default 0;

-- Eliminar paquetes descontinuados (sin membresías que los referencien en fresh DB).
delete from products where code in ('NEXUS', 'Q_TRIMESTRAL', 'Q_SEMESTRAL', 'Q_ANUAL');

-- Paquetes de servicio (activates_earning = puntos >= 100).
insert into products (code, name, type, enrollment_price, renewal_price, points, duration_months, activates_earning, fund_amount, ai_credits_monthly, tier_level) values
  ('BASICO',     'Básico',          'service', 20,   15,   20,  1,  false, 0, 2500,  1),
  ('ESTANDAR',   'Estándar',        'service', 50,   40,   50,  1,  false, 0, 7500,  2),
  ('NOVA',       'Nova',            'service', 125,  97,   100, 1,  true,  0, 20000, 3),
  ('QUANTUM',    'Quantum',         'service', 200,  147,  150, 1,  true,  0, 35000, 4),
  ('NOVA_TRIM',  'Nova Trimestral', 'service', 345,  345,  100, 3,  true,  0, 20000, 3),
  ('NOVA_SEM',   'Nova Semestral',  'service', 660,  660,  100, 6,  true,  0, 20000, 3),
  ('NOVA_ANUAL', 'Nova Anual',      'service', 1200, 1200, 100, 12, true,  0, 20000, 3)
on conflict (code) do update set
  name = excluded.name, type = excluded.type, enrollment_price = excluded.enrollment_price,
  renewal_price = excluded.renewal_price, points = excluded.points,
  duration_months = excluded.duration_months, activates_earning = excluded.activates_earning,
  ai_credits_monthly = excluded.ai_credits_monthly, tier_level = excluded.tier_level,
  is_active = true;

-- FIN MIGRACIÓN 015
```

### 143. `db/migrations/016_credit_packs.sql`
Paquetes de recarga.
```sql
-- ============================================================================
-- TRIADA — Migración 016: Paquetes de recarga de créditos (Tienda)
-- Ejecutar DESPUÉS de 015. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- Recargas prepago (NO expiran). Con bono por volumen. Margen >55% (servir un
-- crédito cuesta ~1/3). El bono de afiliación por recarga se otorga en código.
-- ============================================================================

create table if not exists ia_credit_packs (
  id          text primary key,
  name        text not null,
  price_usd   numeric not null,
  credits     integer not null,
  bonus_label text not null default '',
  sort        integer not null default 0,
  is_active   boolean not null default true
);

alter table ia_credit_packs enable row level security;
do $$ begin
  if not exists (select 1 from pg_policies where tablename='ia_credit_packs' and policyname='ia_packs_read') then
    create policy "ia_packs_read" on ia_credit_packs for select to authenticated using (true);
  end if;
end $$;

insert into ia_credit_packs (id, name, price_usd, credits, bonus_label, sort) values
  ('pack-10',  'Recarga $10',  10,  10000,  '',     1),
  ('pack-20',  'Recarga $20',  20,  22000,  '+10%', 2),
  ('pack-50',  'Recarga $50',  50,  60000,  '+20%', 3),
  ('pack-100', 'Recarga $100', 100, 130000, '+30%', 4)
on conflict (id) do update set
  name = excluded.name, price_usd = excluded.price_usd, credits = excluded.credits,
  bonus_label = excluded.bonus_label, sort = excluded.sort;

-- FIN MIGRACIÓN 016
```

### 144. `db/migrations/017_unified_ai_hub.sql`
Catálogo con capacidades, adjuntos y Tendencias públicas.
```sql
-- ============================================================================
-- TRIADA — Migración 017: Centro de IA unificado (catálogo con capacidades,
-- adjuntos, tendencias públicas)
-- Ejecutar DESPUÉS de 016. Pegar en: Dashboard > SQL Editor.
-- ============================================================================

-- 1. ai_models pasa a ser el catálogo REAL navegable (antes vestigial).
--    Cada fila = un modelo específico de OpenRouter con su costo fijo propio
--    (cuando el usuario lo elige a mano) y qué tipos de adjunto acepta.
alter table ai_models add column if not exists accepts_image boolean not null default false;
alter table ai_models add column if not exists accepts_pdf boolean not null default false;
alter table ai_models add column if not exists accepts_audio boolean not null default false;
alter table ai_models add column if not exists credit_cost numeric not null default 0;
alter table ai_models add column if not exists max_attachment_mb integer not null default 8;

-- Limpiar catálogo viejo y resembrar con IDs verificados contra OpenRouter hoy
-- (input_modalities reales) + costo fijo por selección manual.
delete from ai_models;

insert into ai_models (code, name, provider, tier, category, credit_cost, accepts_image, accepts_pdf, accepts_audio, is_default, is_active, sort) values
  -- Chat — económicos / gratis
  ('openrouter/free',                              'TRIADA Flash',   'TRIADA',   'Gratis', 'chat', 0,  true,  false, false, true,  true, 1),
  ('nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free','Nemotron Omni','NVIDIA','Gratis', 'chat', 0,  true,  false, true,  false, true, 2),
  ('google/gemma-4-31b-it:free',                   'Gemma 4 31B',    'Google',   'Gratis', 'chat', 0,  true,  false, false, false, true, 3),
  ('qwen/qwen3.8-27b:free',                        'Qwen3.8 27B',    'Qwen',     'Gratis', 'chat', 0,  true,  false, false, false, true, 4),
  ('deepseek/deepseek-chat-v3.1',                  'DeepSeek V3.1',  'DeepSeek', '$',      'chat', 3,  false, false, false, false, true, 5),
  ('meta-llama/llama-3.3-70b-instruct',             'Llama 3.3 70B',  'Meta',     '$',      'chat', 3,  false, false, false, false, true, 6),
  -- Chat — premium
  ('deepseek/deepseek-r1-0528',                    'DeepSeek R1',    'DeepSeek', '$$',     'chat', 10, false, false, false, false, true, 7),
  ('openai/gpt-4o-2024-11-20',                      'GPT-4o',         'OpenAI',   '$$$',    'chat', 25, true,  true,  false, false, true, 8),
  ('x-ai/grok-4.7',                                 'Grok 4.7',       'xAI',      '$$$',    'chat', 25, true,  true,  false, false, true, 9),
  ('anthropic/claude-opus-5.5',                     'Claude Opus 5.5','Anthropic','$$$$',   'chat', 50, true,  true,  false, false, true, 10),
  -- Imagen
  ('google/gemini-3.1-flash-image',                 'TRIADA Imagen',  'Google',   '$',      'image', 100, true, false, false, true,  true, 101),
  ('google/gemini-3-pro-image',                     'Imagen Pro',     'Google',   '$$',     'image', 150, true, false, false, false, true, 102),
  ('openai/gpt-5-image',                            'GPT Image',      'OpenAI',   '$$$',    'image', 250, true, true,  false, false, true, 103);

-- 2. Corregir la tarea premium-text del router (apuntaba a un modelo que ya
--    no existe en OpenRouter: z-ai/glm-5.2:free).
update ia_tasks_config
  set model_code = 'nvidia/nemotron-3-nano-omni-30b-a3b-reasoning:free'
  where id = 'premium-text';

-- 3. Tendencias (galería pública, opt-in) + costo registrado por creación.
alter table ai_creations add column if not exists is_public boolean not null default false;
alter table ai_creations add column if not exists credits_spent numeric not null default 0;
alter table ai_creations add column if not exists views integer not null default 0;

drop policy if exists "ai_creations_own_or_adm" on ai_creations;
create policy "ai_creations_own_or_public_or_adm" on ai_creations
  for select to authenticated
  using (user_id = auth.uid() or is_public = true or public.is_admin());

-- 4. Adjuntos en los mensajes del chat.
alter table ai_messages add column if not exists attachments jsonb not null default '[]'::jsonb;

-- FIN MIGRACIÓN 017
```

### 145. `db/migrations/018_super_prompter.sql`
Tarea del Super Prompter.
```sql
-- ============================================================================
-- TRIADA — Migración 018: Tarea "Super Prompter" (mejora de prompts)
-- Ejecutar DESPUÉS de 017. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- Reutiliza el modelo económico del chat (gratis) para expandir una idea corta
-- en un prompt detallado, listo para Imágenes/Editor/Video/Audio.
-- ============================================================================

insert into ia_tasks_config (id, task_name, category, fixed_credit_cost, model_code) values
  ('prompt-enhance', 'Super Prompter (mejora de prompt)', 'texto', 1, 'openrouter/free')
on conflict (id) do update set
  task_name = excluded.task_name, category = excluded.category,
  fixed_credit_cost = excluded.fixed_credit_cost, model_code = excluded.model_code;

-- FIN MIGRACIÓN 018
```

### 146. `db/migrations/020_tienda_ecommerce.sql`
Tienda: campos de presentación, pedidos, cupones y RLS.
```sql
-- ============================================================================
-- TRIADA — Migración 020: Tienda estilo e-commerce (catálogo con imagen,
-- carrito, pedidos y cupones) + gestión desde el panel de administración.
-- Ejecutar DESPUÉS de 019. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
-- IMPORTANTE: la tabla `orders` NO se toca. Sigue siendo el libro que alimenta
-- el motor de comisiones (una fila por producto comprado). La tienda vive
-- ENCIMA: `store_orders` guarda el pedido con su carrito y, al marcarse pagado,
-- se "explota" en una fila de `orders` por artículo. Así el checkout nuevo no
-- altera el cálculo de puntos, binario ni rangos.
-- ============================================================================

-- ----------------------------------------------------------------------------
-- 1. Catálogo: campos de presentación (imagen, descripción, insignia...).
--    La imagen admite jpg/png/svg/webp y se edita desde el panel admin.
-- ----------------------------------------------------------------------------
alter table products add column if not exists slug             text;
alter table products add column if not exists description      text;
alter table products add column if not exists image_url        text;
alter table products add column if not exists badge            text;
alter table products add column if not exists accent           text not null default '#C2B280';
alter table products add column if not exists sort             integer not null default 0;
alter table products add column if not exists compare_at_price decimal(14,2);

-- Slug único derivado del código (para URLs /tienda/<slug>).
update products set slug = lower(code) where slug is null;
create unique index if not exists idx_products_slug on products(slug);

-- ----------------------------------------------------------------------------
-- 2. Pedidos de tienda (carrito completo).
-- ----------------------------------------------------------------------------
create table if not exists store_orders (
  id              uuid primary key default gen_random_uuid(),
  order_number    text unique not null,
  user_id         uuid not null references users(id) on delete cascade,
  -- [{ product_id, code, name, price, qty, points, image_url }]
  items           jsonb not null default '[]'::jsonb,
  subtotal        decimal(14,2) not null default 0,
  discount        decimal(14,2) not null default 0,
  shipping        decimal(14,2) not null default 0,
  total           decimal(14,2) not null default 0,
  coupon_code     text,
  -- received | paid | preparing | shipped | delivered | cancelled
  status          text not null default 'received',
  status_history  jsonb not null default '[]'::jsonb,
  payment_method  text,
  payment_status  text not null default 'pending',  -- pending | paid | failed
  payment_ref     text,
  notes           text,
  -- true cuando ya se generaron las filas de `orders` (evita doble comisión).
  commissions_applied boolean not null default false,
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now()
);
create index if not exists idx_store_orders_user on store_orders(user_id, created_at desc);

-- ----------------------------------------------------------------------------
-- 3. Cupones de descuento.
-- ----------------------------------------------------------------------------
create table if not exists store_coupons (
  code          text primary key,
  type          text not null default 'percent',   -- percent | fixed
  value         decimal(14,2) not null,
  min_subtotal  decimal(14,2) not null default 0,
  max_uses      integer not null default 100,
  uses          integer not null default 0,
  active        boolean not null default true,
  expires_at    timestamptz,
  created_at    timestamptz not null default now()
);

-- ----------------------------------------------------------------------------
-- 4. RLS: cada quien ve sus pedidos; el catálogo y los cupones son de lectura
--    pública para usuarios autenticados. La escritura va por el cliente admin.
-- ----------------------------------------------------------------------------
alter table store_orders  enable row level security;
alter table store_coupons enable row level security;

do $$ begin
  if not exists (select 1 from pg_policies where tablename='store_orders' and policyname='store_orders_own_or_adm') then
    create policy "store_orders_own_or_adm" on store_orders
      for select to authenticated
      using (user_id = auth.uid() or public.is_admin());
  end if;
  if not exists (select 1 from pg_policies where tablename='store_coupons' and policyname='store_coupons_read') then
    create policy "store_coupons_read" on store_coupons
      for select to authenticated using (true);
  end if;
end $$;

-- ----------------------------------------------------------------------------
-- 5. Datos de presentación para el catálogo actual.
--    Las imágenes son de muestra: se cambian desde el panel de administración.
-- ----------------------------------------------------------------------------
update products set
  description = coalesce(description, 'Membresía de distribución: activa tu enlace y tu posición en la organización.'),
  image_url   = coalesce(image_url, '/muestras/producto-ibo.jpg'),
  badge       = 'Requisito',
  accent      = '#5B8DEF',
  sort        = 1
where code = 'IBO';

update products set
  description = coalesce(description, 'Punto de entrada al ecosistema TRIADA. Acceso a la academia y 2.500 créditos de IA al mes.'),
  image_url   = coalesce(image_url, '/muestras/producto-basico.jpg'),
  accent      = '#9AA7BD', sort = 10
where code = 'BASICO';

update products set
  description = coalesce(description, 'Más formación y 7.500 créditos de IA al mes. Desbloquea la generación de video cuando esté disponible.'),
  image_url   = coalesce(image_url, '/muestras/producto-estandar.jpg'),
  accent      = '#43C59E', sort = 20
where code = 'ESTANDAR';

update products set
  description = coalesce(description, 'Academia Nova completa, 20.000 créditos de IA al mes y activación del cobro de comisiones.'),
  image_url   = coalesce(image_url, '/muestras/producto-nova.jpg'),
  badge       = 'Más vendido',
  accent      = '#B98CE8', sort = 30
where code = 'NOVA';

update products set
  description = coalesce(description, 'El paquete completo: Academia Quantum, 35.000 créditos de IA al mes y el máximo de puntos por compra.'),
  image_url   = coalesce(image_url, '/muestras/producto-quantum.jpg'),
  badge       = 'Premium',
  accent      = '#D4AF37', sort = 40
where code = 'QUANTUM';

update products set
  description = coalesce(description, 'Nova con pago adelantado: mismo beneficio mensual con mejor precio por mes.'),
  image_url   = coalesce(image_url, '/muestras/producto-nova.jpg'),
  accent      = '#B98CE8', sort = 50
where code in ('NOVA_TRIM', 'NOVA_SEM', 'NOVA_ANUAL');

update products set
  image_url = coalesce(image_url, '/muestras/producto-merch.jpg'),
  description = coalesce(description, 'Producto oficial TRIADA. Suma puntos y volumen a tu organización.'),
  accent = '#E8845B', sort = 90
where type = 'physical';

-- Cupón de ejemplo (se administra desde el panel).
insert into store_coupons (code, type, value, min_subtotal, max_uses)
values ('TRIADA10', 'percent', 10, 50, 500)
on conflict (code) do nothing;

-- FIN MIGRACIÓN 020
```

### 147. `db/seeds/002_seed.sql`
Semilla de los 13 rangos y la configuración del plan.
```sql
-- ============================================================================
-- TRIADA Business School — Seed inicial
-- Ejecutar DESPUÉS de 001_schema.sql. Pegar en: Dashboard > SQL Editor.
-- Idempotente: usa ON CONFLICT para poder re-ejecutarse sin duplicar.
-- ============================================================================

-- ----------------------------------------------------------------------------
-- RANKS — los 13 rangos (datos exactos de la spec, sección 8)
-- matching_levels: % del binario de la downline por generación (Vía 4).
-- lifestyle_bonus / lifestyle_directs: Vía 5 (rangos 10+).
-- ----------------------------------------------------------------------------
insert into ranks (level, name, zone, points_per_leg, max_pct_line, min_directs_left, min_directs_right, base_check, matching_levels, lifestyle_bonus, lifestyle_directs) values
  (1,  'PIONERO',            'EMPRESARIO', 200,    1.00, 2, 1, 100,    '[]',                                0,    0),
  (2,  'IMPULSOR',           'EMPRESARIO', 400,    0.80, 2, 1, 200,    '[]',                                0,    0),
  (3,  'ÉLITE',              'EMPRESARIO', 1000,   0.70, 2, 2, 500,    '[]',                                0,    0),
  (4,  'EXECUTIVE',          'PREMIER',    2000,   0.70, 2, 2, 1000,   '[]',                                0,    0),
  (5,  'VISIONARIO',         'PREMIER',    4000,   0.65, 3, 2, 2000,   '[]',                                0,    0),
  (6,  'CONQUISTADOR',       'PREMIER',    7000,   0.65, 3, 3, 3500,   '[0.50]',                            0,    0),
  (7,  'SOBERANO',           'DIAMANTE',   10000,  0.60, 4, 4, 5000,   '[0.50,0.10]',                       0,    0),
  (8,  'DIAMANTE',           'DIAMANTE',   20000,  0.60, 5, 4, 10000,  '[0.50,0.10,0.08]',                  0,    0),
  (9,  'DIAMANTE IMPERIAL',  'DIAMANTE',   40000,  0.55, 5, 5, 20000,  '[0.50,0.10,0.08,0.06]',             0,    0),
  (10, 'DIAMANTE MONARCA',   'DIAMANTE',   70000,  0.55, 6, 5, 35000,  '[0.50,0.10,0.08,0.06,0.05]',        500,  8),
  (11, 'LEYENDA',            'LEYENDA',    120000, 0.50, 6, 6, 60000,  '[0.50,0.10,0.08,0.06,0.05]',        1000, 9),
  (12, 'LEYENDA TRIADA',     'LEYENDA',    240000, 0.50, 6, 6, 120000, '[0.50,0.10,0.08,0.06,0.05]',        2000, 10),
  (13, 'LEYENDA CORONA',     'LEYENDA',    500000, 0.50, 6, 6, 250000, '[0.50,0.10,0.08,0.06,0.05]',        4000, 12)
on conflict (level) do update set
  name = excluded.name, zone = excluded.zone, points_per_leg = excluded.points_per_leg,
  max_pct_line = excluded.max_pct_line, min_directs_left = excluded.min_directs_left,
  min_directs_right = excluded.min_directs_right, base_check = excluded.base_check,
  matching_levels = excluded.matching_levels, lifestyle_bonus = excluded.lifestyle_bonus,
  lifestyle_directs = excluded.lifestyle_directs;

-- ----------------------------------------------------------------------------
-- PRODUCTS — catálogo (precios y puntos exactos de la spec, sección 4)
-- ----------------------------------------------------------------------------

-- Programas de servicio
-- NEXUS: enrollment 897 = 397 comisionable + 500 de fondo (fund_amount, excluido del volumen).
insert into products (code, name, type, enrollment_price, renewal_price, points, duration_months, activates_earning, fund_amount) values
  ('NOVA',          'Nova',              'service', 125,  97,   50,   1,  false, 0),
  ('QUANTUM',       'Quantum',           'service', 200,  147,  100,  1,  true,  0),
  ('NEXUS',         'Nexus',             'service', 897,  397,  400,  3,  true,  500),
  ('Q_TRIMESTRAL',  'Quantum Trimestral','service', 450,  397,  400,  3,  true,  0),
  ('Q_SEMESTRAL',   'Quantum Semestral', 'service', 820,  747,  700,  6,  true,  0),
  ('Q_ANUAL',       'Quantum Anual',     'service', 1397, 1297, 1200, 12, true,  0)
on conflict (code) do update set
  name = excluded.name, type = excluded.type, enrollment_price = excluded.enrollment_price,
  renewal_price = excluded.renewal_price, points = excluded.points,
  duration_months = excluded.duration_months, activates_earning = excluded.activates_earning,
  fund_amount = excluded.fund_amount;

-- Productos físicos (solo dan has_position; NO activan distribución ni cobro)
insert into products (code, name, type, enrollment_price, renewal_price, points, duration_months, activates_earning, fund_amount) values
  ('GORRA',          'Gorra',            'physical', 25,  null, 5,  0, false, 0),
  ('CAMISETA',       'Camiseta',         'physical', 35,  null, 6,  0, false, 0),
  ('BUZO',           'Buzo/Hoodie',      'physical', 45,  null, 8,  0, false, 0),
  ('KIT_MARCA',      'Kit de Marca',     'physical', 90,  null, 16, 0, false, 0),
  ('KIT_MENTALIDAD', 'Kit de Mentalidad','physical', 100, null, 18, 0, false, 0)
on conflict (code) do update set
  name = excluded.name, type = excluded.type, enrollment_price = excluded.enrollment_price,
  renewal_price = excluded.renewal_price, points = excluded.points,
  duration_months = excluded.duration_months, activates_earning = excluded.activates_earning,
  fund_amount = excluded.fund_amount;

-- IBO (membresía de distribución: activa can_distribute)
insert into products (code, name, type, enrollment_price, renewal_price, points, duration_months, activates_earning, fund_amount) values
  ('IBO', 'IBO — Membresía de Distribución', 'ibo', 30, 20, 0, 1, false, 0)
on conflict (code) do update set
  name = excluded.name, type = excluded.type, enrollment_price = excluded.enrollment_price,
  renewal_price = excluded.renewal_price, points = excluded.points,
  duration_months = excluded.duration_months, activates_earning = excluded.activates_earning,
  fund_amount = excluded.fund_amount;

-- ----------------------------------------------------------------------------
-- COMPENSATION_CONFIG — constantes del plan (editables sin código)
-- El motor de comisiones (Fase C) lee SIEMPRE de aquí.
-- ----------------------------------------------------------------------------
insert into compensation_config (key, value, description) values
  ('POINT_VALUE',            '2.00',                 'Valor en USD de 1 punto de volumen comisionable.'),
  ('DIRECT_L1_PCT',          '0.20',                 'Vía 1 — Bono directo nivel 1 (% sobre inscripción).'),
  ('DIRECT_L2_PCT',          '0.10',                 'Vía 1 — Bono directo nivel 2 (% sobre inscripción).'),
  ('BINARY_PCT',             '0.15',                 'Vía 2 — Binario (% sobre volumen de pierna menor).'),
  ('RANK_PCT',               '0.10',                 'Vía 3 — Bono de rango (% sobre volumen de pierna menor).'),
  ('MIN_WITHDRAWAL',         '10.00',                'Mínimo acumulado en USD para solicitar retiro.'),
  ('MATCHING_MIN_RANK',      '6',                    'Rango mínimo para acceder al matching (Vía 4): Conquistador.'),
  ('LIFESTYLE_MIN_RANK',     '10',                   'Rango mínimo para bonos de estilo de vida (Vía 5).'),
  ('HOLDING_DAYS',           '7',                    'Días de validación (holding) antes de pagar una comisión.'),
  ('PAYOUT_DAY',             '"monday"',             'Día de pago semanal sugerido.'),
  ('CLOSE_DAY',              '"last_day_of_month"',  'Día de cierre mensual (general para todos).'),
  ('WEEKLY_SPLIT',           '4',                    'En cuántos pagos semanales se divide el cheque mensual.'),
  ('PAYOUT_ALERT_THRESHOLD', '0.30',                 'Umbral de % de payout que dispara alerta de salud financiera.')
on conflict (key) do update set
  value = excluded.value, description = excluded.description;

-- FIN SEED 002
```
