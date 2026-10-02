# PANEL DE ADMINISTRACIÓN — TRIADA Business School
## Documento técnico COMPLETO: arquitectura, pantallas, funciones y código fuente

Generado desde el código real del proyecto (no transcrito a mano): cada bloque de
código es el contenido literal del archivo indicado.

## ÍNDICE

- PARTE 1 — VISIÓN GENERAL (arquitectura, seguridad, mapa de pantallas, flujos, límites)
- PARTE 2 — SEGURIDAD Y ARRANQUE
- PARTE 3 — SHELL Y NAVEGACIÓN
- PARTE 4 — ESCRITURA: SERVER ACTIONS
- PARTE 5 — LECTURA: CAPA DE DATOS
- PARTE 6 — PANTALLAS (VISTAS)
- PARTE 7 — COMPONENTES DEL ADMIN
- PARTE 8 — DOMINIO: COLOCACIÓN Y ACTIVACIÓN
- PARTE 9 — MOTOR DE COMISIONES
- PARTE 10 — BASE DE DATOS

---

# PARTE 1 — VISIÓN GENERAL

## 1.1 Qué es

El admin es la parte del back office que solo ve el rol `is_admin`. Desde ahí se gestiona
todo el negocio MLM binario: miembros, árbol, pedidos de tienda, KYC, cierres mensuales,
pagos semanales, retiros USDT, reportes, configuración del plan de compensación,
productos y promociones. Ruta base: `/admin` (grupo de rutas `app/(admin)/`).

## 1.2 Stack

- Next.js 14 (App Router) + TypeScript + Tailwind CSS
- Supabase: PostgreSQL + Auth + Storage + Row Level Security
- **Server Components** para LEER datos, **Server Actions** para ESCRIBIR (sin API REST propia)
- lucide-react (iconos), i18n propio ES/EN/PT (el texto interno del admin está en español)
- Vitest: 34 tests del motor de comisiones

## 1.3 Seguridad en 3 capas

1. `middleware.ts` refresca la sesión de Supabase en cada navegación.
2. `app/(admin)/layout.tsx` llama `requireAdmin()`: sin sesión → `/login`; sin `is_admin` → `/dashboard`.
3. **Cada Server Action vuelve a llamar `requireAdmin()`.** Un Server Action es un endpoint
   público: la guardia del layout NO lo protege. Nunca omitirla.

Los datos se leen/escriben con el cliente `service_role` (`createAdminClient`), que salta
RLS. Por eso SOLO se usa en servidor y jamás se importa desde un componente `"use client"`.

## 1.4 Patrón de trabajo (el molde de TODO el admin)

```
LEER:    page.tsx (Server Component async) → lib/data/admin.ts → createAdminClient() → render
ESCRIBIR: <form action={miAction}> → actions.ts:
            1) requireAdmin()   2) escribir en BD   3) logAudit()   4) revalidatePath()
```

Enlazar argumentos a una acción sin JavaScript en el cliente:
`<form action={toggleProductAction.bind(null, p.id, !p.is_active)}>`

## 1.5 Mapa de pantallas

| Ruta | Función | Lee de | Escribe con |
|---|---|---|---|
| `/admin` | Dashboard: facturación, comisiones, % payout con alerta, miembros, pendientes, gráficas | `getAdminDashboard`, `getPendingGlobal`, `getAdminCharts` | — |
| `/admin/miembros` | Lista con búsqueda/filtros y rango | `getMembers` | — |
| `/admin/miembros/[id]` | Ficha: programas, billetera, activar/desactivar/suspender, aprobar KYC | `getMemberDetail` | `setMemberStatusAction`, `approveKycAction` |
| `/admin/arbol` | Árbol binario de cualquier raíz + colocación manual de pendientes | `getTreeData`, `getTreeRoots`, `getPendingGlobal` | `adminPlaceMemberAction` |
| `/admin/pedidos` | Pedidos de tienda + cupones + cambio de estado | `getAllStoreOrders`, `getCoupons` | `updateStoreOrderStatusAction` |
| `/admin/kyc` | Cola de verificación de identidad | `getKycQueue` | `approveKycAction` |
| `/admin/cierres` | Cierre mensual y pago semanal | `getCommissionsForPeriod` | `runCloseAction`, `runPayoutAction` |
| `/admin/pagos` | Cola de retiros USDT | `getWithdrawalsQueue` | `setWithdrawalAction` |
| `/admin/reportes` | Salud del negocio: miembros por mes/rango | `getReports` | — |
| `/admin/configuracion` | Constantes del plan y tabla de 13 rangos (editables sin código) | `getCompensationConfig`, `getRanksRaw` | `updateConfigAction`, `updateRankAction` |
| `/admin/productos` | Catálogo + presentación en tienda (imagen, insignia, orden) | `getProductsRaw` | `upsertProductAction`, `updateProductMediaAction`, `toggleProductAction` |
| `/admin/promociones` | Campañas temporales (ej. Mes Nitro) | `getPromotions` | `savePromotionAction` |

## 1.6 Flujos de negocio clave

**Cierre mensual** (`runCloseAction` → `monthlyClose`). Por cada usuario: suma el volumen del
período + el arrastre (carry-over) anterior; liquida SIEMPRE sobre la **pierna menor**; calcula
el rango con 4 condiciones (puntos por pierna, % máx por línea, directos activos por pierna,
`can_earn`); paga binario (15 %) y rango (10 %); la pierna mayor arrastra. Después: matching
(rango ≥ 6) y estilo de vida (rango ≥ 10). Un **inactivo pierde TODO su carry-over**. Quien no
tiene `can_earn` no cobra, pero su volumen igual liquida y arrastra.

**Pago semanal** (`runPayoutAction`): `runHolding()` valida comisiones que cumplieron su
retención y `runWeeklyPayout()` las acredita a la billetera de cada miembro.

**Retiros** (`setWithdrawalAction`): *approved* solo cambia el estado; *paid* guarda el `tx_hash`
y suma a `wallet.total_withdrawn` (el saldo ya se descontó al solicitar); *rejected* **reembolsa**
el saldo reservado.

**Tienda → admin**: el distribuidor compra en `/tienda` → se crea una fila en `store_orders` y por
cada unidad se ejecuta `simulateActivation()`, que inserta en `orders` (el libro que lee el motor).
La tabla `orders` NO se modificó: la tienda vive encima de ella, así el checkout no altera puntos,
binario ni rangos. El admin gestiona el estado en `/admin/pedidos`; lo que edita en
`/admin/productos` aparece al instante en `/tienda` (`revalidatePath("/tienda")`).

**Configuración sin código**: `updateConfigAction` y `updateRankAction` escriben en
`compensation_config` y `ranks`. El motor lee TODO de ahí (`loadEngineConfig`, `loadRanks`),
así que cambiar un porcentaje o los puntos de un rango no requiere desplegar nada.

**Las 3 llaves** (independientes entre sí): `has_position` (compró algo), `can_distribute`
(IBO activo → puede referir), `can_earn` (servicio ≥ 100 pts Y can_distribute → cobra red).

## 1.7 Cómo agregar una pantalla nueva

1. `app/(admin)/admin/<nombre>/page.tsx` como Server Component `async` (el layout ya la protege).
2. Si lee datos: nueva función en `lib/data/admin.ts` con `createAdminClient()`.
3. Si escribe: nuevo Server Action en `actions.ts` con el molde
   `requireAdmin → escribir → logAudit → revalidatePath`.
4. Agregar el ítem al arreglo `GROUPS` de `AdminSidebar.tsx` y su clave en `lib/i18n/messages.ts`.

## 1.8 Acceso de prueba

`/login` con la cuenta admin de prueba, creada con `node scripts/bootstrap-users.mjs` (correo y
clave de prueba están en ese script). Sin `is_admin = true`, `/admin` redirige a `/dashboard`.

## 1.9 Límites conocidos (antes de producción)

- Pagos de tienda **simulados** (`simulateActivation`); la pasarela USDT real no está conectada.
- Condición 2 del rango (máx % por línea de patrocinio): el cierre usa `maxLineFraction = 0`, no frena ascensos todavía.
- Los cupones solo se **listan** en el admin; falta alta/edición.
- Texto interno del admin solo en español.
- Seguridad pendiente: gatear la compra simulada, validar tipo/tamaño del avatar, exigir contraseña actual al cambiarla, limitar intentos de login, políticas RLS de escritura.


---

# PARTE 2 — SEGURIDAD Y ARRANQUE
Las tres capas de protección y el cliente de base de datos del servidor.

### 1. `middleware.ts`
Capa 1: refresca la sesión de Supabase en cada navegación (excluye estáticos).
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

### 2. `lib/auth/session.ts`
Capa 2: une Supabase Auth con el perfil de `public.users`. `requireAdmin()` es la guardia que usan el layout y TODAS las acciones.
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

### 3. `lib/supabase/admin.ts`
Cliente `service_role`: salta RLS. Solo servidor.
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

### 4. `app/(admin)/layout.tsx`
Capa 3: envuelve todas las páginas de /admin/*. Exige rol admin y monta el shell con el sidebar.
```tsx
import { requireAdmin } from "@/lib/auth/session";
import { AdminSidebar } from "@/components/admin/AdminSidebar";
import { DashboardShell } from "@/components/dashboard/DashboardShell";
import { UserMenu } from "@/components/dashboard/UserMenu";
import { ThemeToggle } from "@/components/ui/ThemeToggle";
import { LanguageProvider } from "@/components/i18n/LanguageProvider";
import { LanguageSwitcher } from "@/components/i18n/LanguageSwitcher";
import { getLocale } from "@/lib/i18n/server";
import { messages } from "@/lib/i18n/messages";

/** Layout protegido del admin con sidebar. Exige rol is_admin. */
export default async function AdminLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const admin = await requireAdmin();
  const locale = getLocale();

  return (
    <LanguageProvider locale={locale} dict={messages[locale]}>
      <DashboardShell
        sidebar={<AdminSidebar />}
        right={
          <>
            <LanguageSwitcher />
            <ThemeToggle />
            <UserMenu name={admin.full_name ?? admin.email} code={admin.referral_code} avatarUrl={admin.avatar_url} />
          </>
        }
      >
        {children}
      </DashboardShell>
    </LanguageProvider>
  );
}
```


---

# PARTE 3 — SHELL Y NAVEGACIÓN
Estructura visual compartida: barra lateral colapsable, cabecera con idioma/tema/usuario.

### 5. `components/admin/AdminSidebar.tsx`
Menú lateral agrupado (Red · Operaciones · Informes · Ajustes). Para agregar una pantalla se añade una línea al arreglo `GROUPS`.
```tsx
"use client";

import Link from "next/link";
import { usePathname } from "next/navigation";
import {
  LayoutDashboard,
  Users,
  Share2,
  ShieldCheck,
  SlidersHorizontal,
  Package,
  ShoppingCart,
  Sparkles,
  CalendarCheck,
  Banknote,
  BarChart3,
  LogOut,
} from "lucide-react";
import { logoutAction } from "@/app/(auth)/actions";
import { Wordmark } from "@/components/ui/Wordmark";
import { useT } from "@/components/i18n/LanguageProvider";
import { cn } from "@/lib/utils";

type Item = { href: string; key: string; icon: typeof LayoutDashboard };
type Group = { titleKey: string | null; items: Item[] };

const GROUPS: Group[] = [
  { titleKey: null, items: [{ href: "/admin", key: "nav.dashboard", icon: LayoutDashboard }] },
  {
    titleKey: "group.red",
    items: [
      { href: "/admin/miembros", key: "nav.members", icon: Users },
      { href: "/admin/arbol", key: "nav.tree", icon: Share2 },
    ],
  },
  {
    titleKey: "group.operations",
    items: [
      { href: "/admin/pedidos", key: "nav.orders", icon: ShoppingCart },
      { href: "/admin/kyc", key: "nav.kyc", icon: ShieldCheck },
      { href: "/admin/cierres", key: "nav.closings", icon: CalendarCheck },
      { href: "/admin/pagos", key: "nav.payments", icon: Banknote },
    ],
  },
  {
    titleKey: "group.reports",
    items: [{ href: "/admin/reportes", key: "nav.reports", icon: BarChart3 }],
  },
  {
    titleKey: "group.settings",
    items: [
      { href: "/admin/configuracion", key: "nav.settings", icon: SlidersHorizontal },
      { href: "/admin/productos", key: "nav.products", icon: Package },
      { href: "/admin/promociones", key: "nav.promotions", icon: Sparkles },
    ],
  },
];

export function AdminSidebar() {
  const pathname = usePathname();
  const { t } = useT();

  return (
    <div className="flex h-full min-h-screen flex-col lg:sticky lg:top-0 lg:h-screen">
      <div className="px-6 py-6">
        <Link href="/admin"><Wordmark /></Link>
        <span className="mt-2 inline-block rounded-full bg-cobalt/15 px-2 py-0.5 text-[10px] font-semibold uppercase tracking-wide text-cobalt">
          {t("group.admin")}
        </span>
      </div>
      <nav className="flex-1 space-y-5 overflow-y-auto px-3 pb-4">
        {GROUPS.map((group, gi) => (
          <div key={group.titleKey ?? `g${gi}`}>
            {group.titleKey && (
              <p className="px-3 pb-2 text-[10px] font-semibold uppercase tracking-label text-muted/60">
                {t(group.titleKey)}
              </p>
            )}
            <div className="space-y-1">
              {group.items.map(({ href, key, icon: Icon }) => {
                const active = href === "/admin" ? pathname === href : pathname.startsWith(href);
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

### 6. `components/dashboard/DashboardShell.tsx`
Shell compartido (distribuidor y admin): sidebar con colapso persistente (localStorage), drawer en móvil y cabecera.
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

### 7. `components/ui/Card.tsx`
Componentes de tarjeta (`Card`, `StatCard`) que usan todas las pantallas.
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

### 8. `lib/utils.ts`
Utilidades: `cn()` para clases y `formatUSD()`.
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


---

# PARTE 4 — ESCRITURA: SERVER ACTIONS
Todas las operaciones que modifican datos. Molde: requireAdmin → escribir → logAudit → revalidatePath.

### 9. `app/(admin)/admin/actions.ts`
Acciones: estado de miembro, KYC, colocación manual, constantes del plan, rangos, productos, pedidos, promociones, cierre mensual, pago semanal y retiros. `logAudit` registra quién hizo qué en `audit_log`.
```ts
"use server";

import { revalidatePath } from "next/cache";
import { requireAdmin } from "@/lib/auth/session";
import { createAdminClient } from "@/lib/supabase/admin";
import { getTreeData, type TreeNode } from "@/lib/data/tree";
import { placeMember } from "@/lib/domain/placement";
import { monthlyClose, type CloseSummary } from "@/lib/engine/monthly-close";
import { runHolding, runWeeklyPayout } from "@/lib/engine/payout";
import { currentPeriod } from "@/lib/engine/period";
import type { BinaryLeg } from "@/types/database";

/** Registra una acción del admin en la auditoría. */
async function logAudit(
  adminId: string,
  action: string,
  targetUserId: string | null,
  detail: Record<string, unknown>
) {
  const supabase = createAdminClient();
  await supabase.from("audit_log").insert({
    admin_id: adminId,
    action,
    target_user_id: targetUserId,
    detail,
  });
}

/** Activa o desactiva un miembro. Desactivar tiene efecto en el próximo cierre
 *  (pierde su carry-over, según la regla de inactividad del motor). */
export async function setMemberStatusAction(
  userId: string,
  status: "active" | "inactive" | "suspended"
): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();

  const { data: before } = await supabase
    .from("users")
    .select("status")
    .eq("id", userId)
    .maybeSingle();

  await supabase.from("users").update({ status }).eq("id", userId);
  await logAudit(admin.id, "member_status", userId, {
    from: before?.status,
    to: status,
  });

  revalidatePath(`/admin/miembros/${userId}`);
  revalidatePath("/admin/miembros");
}

/** Aprueba el KYC de un miembro (desbloquea sus retiros). */
export async function approveKycAction(userId: string): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();

  await supabase.from("users").update({ kyc_verified: true }).eq("id", userId);
  await logAudit(admin.id, "kyc_approve", userId, { kyc_verified: true });

  revalidatePath("/admin/kyc");
  revalidatePath(`/admin/miembros/${userId}`);
}

/** Carga un subárbol para el árbol del admin (sin restricción de red). */
export async function adminLoadTreeAction(
  nodeId: string,
  tree: "binary" | "sponsor"
): Promise<TreeNode | null> {
  await requireAdmin();
  return getTreeData(nodeId, tree);
}

export interface PlaceResult {
  error?: string;
}

/**
 * Colocación (admin): ubica a un miembro pendiente (ya registrado) en una
 * posición exacta del árbol binario. El admin puede colocar a cualquiera.
 */
export async function adminPlaceMemberAction(
  memberId: string,
  placementId: string,
  leg: BinaryLeg
): Promise<PlaceResult> {
  const admin = await requireAdmin();
  try {
    await placeMember(memberId, placementId, leg);
  } catch (err) {
    return { error: err instanceof Error ? err.message : "No se pudo colocar." };
  }
  await logAudit(admin.id, "manual_placement", memberId, { placementId, leg });
  revalidatePath("/admin/arbol");
  return {};
}

// ---- Configuración del plan (sin código) ----

/** Actualiza una constante de compensation_config. */
export async function updateConfigAction(formData: FormData): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  const key = String(formData.get("key") ?? "");
  const raw = String(formData.get("value") ?? "").trim();
  // Número si parsea; si no, texto.
  const num = Number(raw);
  const value: number | string = raw !== "" && !Number.isNaN(num) ? num : raw;

  await supabase.from("compensation_config").update({ value }).eq("key", key);
  await logAudit(admin.id, "config_update", null, { key, value });
  revalidatePath("/admin/configuracion");
}

/** Actualiza una fila de la tabla de rangos. */
export async function updateRankAction(formData: FormData): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  const level = Number(formData.get("level"));
  const matching = String(formData.get("matching_levels") ?? "")
    .split(",")
    .map((s) => Number(s.trim()))
    .filter((n) => !Number.isNaN(n));

  const patch = {
    points_per_leg: Number(formData.get("points_per_leg")),
    max_pct_line: Number(formData.get("max_pct_line")),
    min_directs_left: Number(formData.get("min_directs_left")),
    min_directs_right: Number(formData.get("min_directs_right")),
    base_check: Number(formData.get("base_check")),
    matching_levels: matching,
    lifestyle_bonus: Number(formData.get("lifestyle_bonus")),
    lifestyle_directs: Number(formData.get("lifestyle_directs")),
  };
  await supabase.from("ranks").update(patch).eq("level", level);
  await logAudit(admin.id, "rank_update", null, { level, ...patch });
  revalidatePath("/admin/configuracion");
}

// ---- Productos ----

export async function upsertProductAction(formData: FormData): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  const code = String(formData.get("code") ?? "").trim().toUpperCase();
  if (!code) return;

  const text = (k: string) => {
    const v = String(formData.get(k) ?? "").trim();
    return v === "" ? null : v;
  };

  const row = {
    code,
    name: String(formData.get("name") ?? "").trim(),
    type: String(formData.get("type") ?? "service"),
    enrollment_price: Number(formData.get("enrollment_price")),
    renewal_price: formData.get("renewal_price") ? Number(formData.get("renewal_price")) : null,
    points: Number(formData.get("points")),
    duration_months: Number(formData.get("duration_months")),
    fund_amount: Number(formData.get("fund_amount") ?? 0),
    activates_earning: Number(formData.get("points")) >= 100,
    // Presentación en la tienda.
    slug: (text("slug") ?? code).toLowerCase(),
    description: text("description"),
    image_url: text("image_url"),
    badge: text("badge"),
    accent: text("accent") ?? "#C2B280",
    sort: Number(formData.get("sort") ?? 0),
    compare_at_price: formData.get("compare_at_price") ? Number(formData.get("compare_at_price")) : null,
  };
  await supabase.from("products").upsert(row, { onConflict: "code" });
  await logAudit(admin.id, "product_upsert", null, { code });
  revalidatePath("/admin/productos");
  revalidatePath("/tienda");
}

/** Edita solo los campos de presentación de un producto ya existente. */
export async function updateProductMediaAction(formData: FormData): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  const id = String(formData.get("id") ?? "");
  if (!id) return;

  const text = (k: string) => {
    const v = String(formData.get(k) ?? "").trim();
    return v === "" ? null : v;
  };

  await supabase
    .from("products")
    .update({
      image_url: text("image_url"),
      description: text("description"),
      badge: text("badge"),
      accent: text("accent") ?? "#C2B280",
      sort: Number(formData.get("sort") ?? 0),
    })
    .eq("id", id);

  await logAudit(admin.id, "product_media_update", null, { id });
  revalidatePath("/admin/productos");
  revalidatePath("/tienda");
}

/** Cambia el estado de un pedido de tienda. */
export async function updateStoreOrderStatusAction(orderId: string, status: string): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();

  const { data: current } = await supabase
    .from("store_orders")
    .select("status_history")
    .eq("id", orderId)
    .maybeSingle();

  const history = Array.isArray(current?.status_history) ? current!.status_history : [];
  history.push({ status, at: new Date().toISOString() });

  await supabase
    .from("store_orders")
    .update({ status, status_history: history, updated_at: new Date().toISOString() })
    .eq("id", orderId);

  await logAudit(admin.id, "store_order_status", null, { orderId, status });
  revalidatePath("/admin/pedidos");
}

export async function toggleProductAction(productId: string, isActive: boolean): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  await supabase.from("products").update({ is_active: isActive }).eq("id", productId);
  await logAudit(admin.id, "product_toggle", null, { productId, isActive });
  revalidatePath("/admin/productos");
}

// ---- Promociones ----

export async function savePromotionAction(formData: FormData): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  const key = String(formData.get("key") ?? "");
  const active = formData.get("active") === "on";
  const starts = String(formData.get("starts_at") ?? "");
  const ends = String(formData.get("ends_at") ?? "");

  await supabase
    .from("promotions")
    .update({
      active,
      starts_at: starts ? new Date(starts).toISOString() : null,
      ends_at: ends ? new Date(ends).toISOString() : null,
    })
    .eq("key", key);
  await logAudit(admin.id, "promotion_update", null, { key, active });
  revalidatePath("/admin/promociones");
}

// ---- Cierres y pagos ----

/** Ejecuta el cierre mensual del período en curso. */
export async function runCloseAction(): Promise<CloseSummary> {
  const admin = await requireAdmin();
  const summary = await monthlyClose(currentPeriod());
  await logAudit(admin.id, "monthly_close", null, summary as unknown as Record<string, unknown>);
  revalidatePath("/admin/cierres");
  return summary;
}

/** Valida comisiones (holding) y corre un pago semanal. */
export async function runPayoutAction(): Promise<{ validated: number; usersPaid: number; totalDisbursed: number }> {
  const admin = await requireAdmin();
  const period = currentPeriod();
  const validated = await runHolding();
  const payout = await runWeeklyPayout(period);
  await logAudit(admin.id, "weekly_payout", null, { validated, ...payout });
  revalidatePath("/admin/cierres");
  revalidatePath("/admin/pagos");
  return { validated, usersPaid: payout.usersPaid, totalDisbursed: payout.totalDisbursed };
}

/** Aprueba / paga / rechaza una solicitud de retiro. */
export async function setWithdrawalAction(formData: FormData): Promise<void> {
  const admin = await requireAdmin();
  const supabase = createAdminClient();
  const id = String(formData.get("id") ?? "");
  const status = String(formData.get("status") ?? "");
  const txHash = String(formData.get("tx_hash") ?? "").trim() || null;

  const { data: wd } = await supabase
    .from("withdrawals")
    .select("user_id, amount, status")
    .eq("id", id)
    .maybeSingle();
  if (!wd) return;

  if (status === "approved") {
    await supabase.from("withdrawals").update({ status: "approved" }).eq("id", id);
  } else if (status === "paid") {
    await supabase
      .from("withdrawals")
      .update({ status: "paid", tx_hash: txHash, processed_at: new Date().toISOString() })
      .eq("id", id);
    // Acreditar total_withdrawn (el saldo ya se descontó al solicitar).
    const { data: w } = await supabase.from("wallet").select("id, total_withdrawn").eq("user_id", wd.user_id).maybeSingle();
    if (w) {
      await supabase
        .from("wallet")
        .update({ total_withdrawn: Math.round((Number(w.total_withdrawn) + Number(wd.amount)) * 100) / 100 })
        .eq("id", w.id);
    }
  } else if (status === "rejected") {
    await supabase
      .from("withdrawals")
      .update({ status: "rejected", processed_at: new Date().toISOString() })
      .eq("id", id);
    // Reembolsar el saldo reservado.
    const { data: w } = await supabase.from("wallet").select("id, balance").eq("user_id", wd.user_id).maybeSingle();
    if (w) {
      await supabase
        .from("wallet")
        .update({ balance: Math.round((Number(w.balance) + Number(wd.amount)) * 100) / 100 })
        .eq("id", w.id);
    }
  }

  await logAudit(admin.id, "withdrawal_" + status, wd.user_id, { id, amount: wd.amount });
  revalidatePath("/admin/pagos");
}
```


---

# PARTE 5 — LECTURA: CAPA DE DATOS
Funciones que consultan la base para las pantallas del admin. Solo servidor.

### 10. `lib/data/admin.ts`
Dashboard, miembros, ficha, comisiones por período, retiros, reportes, gráficas, pendientes globales, KYC, configuración, productos, promociones y raíces del árbol.
```ts
/**
 * CAPA DE DATOS DEL ADMIN.
 *
 * ⚠️ SOLO SERVIDOR. Las páginas que la usan están protegidas por requireAdmin.
 * Usa el cliente admin (service_role): el admin ve todo el sistema.
 */
import { createAdminClient } from "@/lib/supabase/admin";
import { currentPeriod } from "@/lib/engine/period";
import { loadEngineConfig, loadRanks } from "@/lib/engine/config";

// ----------------------------------------------------------------------------
// DASHBOARD ADMIN
// ----------------------------------------------------------------------------
export interface AdminDashboard {
  billing: number;
  commissionsTotal: number;
  commissionsPaid: number;
  payoutPct: number;
  alertThreshold: number;
  alert: boolean;
  membersTotal: number;
  membersActive: number;
  membersInactive: number;
  newThisMonth: number;
  withdrawals: { pending: number; approved: number; paid: number; rejected: number };
}

export async function getAdminDashboard(): Promise<AdminDashboard> {
  const supabase = createAdminClient();
  const cfg = await loadEngineConfig();
  const period = currentPeriod();
  const monthStart = `${period}-01`;

  const [orders, comms, commsPaid, total, active, inactive, newM, wds] = await Promise.all([
    supabase.from("orders").select("amount").eq("payment_status", "paid"),
    supabase.from("commissions").select("amount"),
    supabase.from("commissions").select("amount").eq("status", "paid"),
    supabase.from("users").select("*", { count: "exact", head: true }),
    supabase.from("users").select("*", { count: "exact", head: true }).eq("status", "active"),
    supabase.from("users").select("*", { count: "exact", head: true }).eq("status", "inactive"),
    supabase.from("users").select("*", { count: "exact", head: true }).gte("created_at", monthStart),
    supabase.from("withdrawals").select("status"),
  ]);

  const billing = (orders.data ?? []).reduce((s, o) => s + Number(o.amount), 0);
  const commissionsTotal = (comms.data ?? []).reduce((s, c) => s + Number(c.amount), 0);
  const commissionsPaid = (commsPaid.data ?? []).reduce((s, c) => s + Number(c.amount), 0);
  const payoutPct = billing > 0 ? commissionsTotal / billing : 0;

  const wdCounts = { pending: 0, approved: 0, paid: 0, rejected: 0 };
  for (const w of wds.data ?? []) {
    if (w.status in wdCounts) wdCounts[w.status as keyof typeof wdCounts]++;
  }

  return {
    billing,
    commissionsTotal,
    commissionsPaid,
    payoutPct,
    alertThreshold: cfg.PAYOUT_ALERT_THRESHOLD,
    alert: payoutPct > cfg.PAYOUT_ALERT_THRESHOLD,
    membersTotal: total.count ?? 0,
    membersActive: active.count ?? 0,
    membersInactive: inactive.count ?? 0,
    newThisMonth: newM.count ?? 0,
    withdrawals: wdCounts,
  };
}

// ----------------------------------------------------------------------------
// GESTIÓN DE MIEMBROS
// ----------------------------------------------------------------------------
export interface MemberRow {
  id: string;
  full_name: string | null;
  email: string;
  referral_code: string;
  country: string | null;
  current_rank: number;
  status: string;
  can_distribute: boolean;
  can_earn: boolean;
  kyc_verified: boolean;
}

export async function getMembers(opts: {
  search?: string;
  status?: string;
}): Promise<MemberRow[]> {
  const supabase = createAdminClient();
  let query = supabase
    .from("users")
    .select(
      "id, full_name, email, referral_code, country, current_rank, status, can_distribute, can_earn, kyc_verified"
    )
    .order("created_at", { ascending: false })
    .limit(200);

  if (opts.status) query = query.eq("status", opts.status);
  if (opts.search) {
    const s = opts.search.trim();
    query = query.or(
      `full_name.ilike.%${s}%,email.ilike.%${s}%,referral_code.ilike.%${s}%`
    );
  }

  const { data } = await query;
  return (data ?? []) as MemberRow[];
}

export interface MemberDetail {
  profile: MemberRow & { sponsor_id: string | null; phone: string | null; created_at: string };
  wallet: { balance: number; total_earned: number; total_withdrawn: number } | null;
  memberships: { type: string; program_code: string | null; points: number; expires_at: string | null }[];
  directs: number;
  commissionsTotal: number;
  rankName: string | null;
}

export async function getMemberDetail(id: string): Promise<MemberDetail | null> {
  const supabase = createAdminClient();
  const ranks = await loadRanks();

  const { data: profile } = await supabase
    .from("users")
    .select(
      "id, full_name, email, referral_code, country, phone, current_rank, status, can_distribute, can_earn, kyc_verified, sponsor_id, created_at"
    )
    .eq("id", id)
    .maybeSingle();
  if (!profile) return null;

  const [wallet, mems, directs, comms] = await Promise.all([
    supabase.from("wallet").select("balance, total_earned, total_withdrawn").eq("user_id", id).maybeSingle(),
    supabase.from("memberships").select("type, program_code, points, expires_at").eq("user_id", id).eq("status", "active"),
    supabase.from("users").select("*", { count: "exact", head: true }).eq("sponsor_id", id),
    supabase.from("commissions").select("amount").eq("user_id", id),
  ]);

  return {
    profile: profile as MemberDetail["profile"],
    wallet: wallet.data
      ? {
          balance: Number(wallet.data.balance),
          total_earned: Number(wallet.data.total_earned),
          total_withdrawn: Number(wallet.data.total_withdrawn),
        }
      : null,
    memberships: (mems.data ?? []).map((m) => ({
      type: m.type,
      program_code: m.program_code,
      points: m.points,
      expires_at: m.expires_at,
    })),
    directs: directs.count ?? 0,
    commissionsTotal: (comms.data ?? []).reduce((s, c) => s + Number(c.amount), 0),
    rankName: ranks.find((r) => r.level === profile.current_rank)?.name ?? null,
  };
}

// ----------------------------------------------------------------------------
// CIERRES Y COMISIONES
// ----------------------------------------------------------------------------
export async function getCommissionsForPeriod(period: string) {
  const supabase = createAdminClient();
  const { data: comms } = await supabase
    .from("commissions")
    .select("user_id, type, amount, status")
    .eq("period", period)
    .order("amount", { ascending: false })
    .limit(200);

  const ids = [...new Set((comms ?? []).map((c) => c.user_id))];
  const codeById = new Map<string, string>();
  if (ids.length > 0) {
    const { data: users } = await supabase.from("users").select("id, referral_code").in("id", ids);
    for (const u of users ?? []) codeById.set(u.id, u.referral_code);
  }
  return (comms ?? []).map((c) => ({
    code: codeById.get(c.user_id) ?? "—",
    type: c.type as string,
    amount: Number(c.amount),
    status: c.status as string,
  }));
}

// ----------------------------------------------------------------------------
// RETIROS (cola de pagos)
// ----------------------------------------------------------------------------
export async function getWithdrawalsQueue() {
  const supabase = createAdminClient();
  const { data: wds } = await supabase
    .from("withdrawals")
    .select("id, user_id, amount, wallet_address, status, tx_hash, requested_at")
    .order("requested_at", { ascending: false })
    .limit(200);

  const ids = [...new Set((wds ?? []).map((w) => w.user_id))];
  const userById = new Map<string, { code: string; name: string | null }>();
  if (ids.length > 0) {
    const { data: users } = await supabase.from("users").select("id, referral_code, full_name").in("id", ids);
    for (const u of users ?? []) userById.set(u.id, { code: u.referral_code, name: u.full_name });
  }
  return (wds ?? []).map((w) => ({
    id: w.id,
    code: userById.get(w.user_id)?.code ?? "—",
    name: userById.get(w.user_id)?.name ?? "—",
    amount: Number(w.amount),
    address: w.wallet_address,
    status: w.status as string,
    txHash: w.tx_hash as string | null,
    requestedAt: w.requested_at as string,
  }));
}

// ----------------------------------------------------------------------------
// REPORTES
// ----------------------------------------------------------------------------
export async function getReports() {
  const supabase = createAdminClient();
  const ranks = await loadRanks();

  const [orders, comms, users] = await Promise.all([
    supabase.from("orders").select("amount").eq("payment_status", "paid"),
    supabase.from("commissions").select("amount, status"),
    supabase.from("users").select("current_rank, created_at"),
  ]);

  const billing = (orders.data ?? []).reduce((s, o) => s + Number(o.amount), 0);
  const commissionsTotal = (comms.data ?? []).reduce((s, c) => s + Number(c.amount), 0);
  const payoutPct = billing > 0 ? commissionsTotal / billing : 0;

  // Miembros por rango (0..13).
  const byRank = new Map<number, number>();
  for (const u of users.data ?? []) byRank.set(u.current_rank ?? 0, (byRank.get(u.current_rank ?? 0) ?? 0) + 1);
  const membersByRank = [
    { level: 0, name: "Sin rango", count: byRank.get(0) ?? 0 },
    ...ranks.map((r) => ({ level: r.level, name: r.name, count: byRank.get(r.level) ?? 0 })),
  ];

  // Nuevos miembros por mes (últimos 6).
  const byMonth = new Map<string, number>();
  for (const u of users.data ?? []) {
    const m = (u.created_at as string).slice(0, 7);
    byMonth.set(m, (byMonth.get(m) ?? 0) + 1);
  }
  const newByMonth = [...byMonth.entries()]
    .map(([period, count]) => ({ period, count }))
    .sort((a, b) => a.period.localeCompare(b.period))
    .slice(-6);

  return { billing, commissionsTotal, payoutPct, membersByRank, newByMonth };
}

// ----------------------------------------------------------------------------
// GRÁFICOS Y LISTAS DEL DASHBOARD ADMIN
// ----------------------------------------------------------------------------
export interface AdminCharts {
  monthly: { period: string; income: number; commission: number }[];
  recentMembers: { name: string; code: string; date: string }[];
  topEarners: { name: string; code: string; amount: number }[];
}

export async function getAdminCharts(): Promise<AdminCharts> {
  const supabase = createAdminClient();

  const [orders, comms, members, allComms, users] = await Promise.all([
    supabase.from("orders").select("amount, created_at").eq("payment_status", "paid"),
    supabase.from("commissions").select("amount, period"),
    supabase.from("users").select("full_name, referral_code, created_at").order("created_at", { ascending: false }).limit(10),
    supabase.from("commissions").select("user_id, amount"),
    supabase.from("users").select("id, full_name, referral_code"),
  ]);

  // Ingreso (facturación) y comisión por mes.
  const incomeByMonth = new Map<string, number>();
  for (const o of orders.data ?? []) {
    const m = (o.created_at as string).slice(0, 7);
    incomeByMonth.set(m, (incomeByMonth.get(m) ?? 0) + Number(o.amount));
  }
  const commByMonth = new Map<string, number>();
  for (const c of comms.data ?? []) {
    commByMonth.set(c.period, (commByMonth.get(c.period) ?? 0) + Number(c.amount));
  }
  const periods = [...new Set([...incomeByMonth.keys(), ...commByMonth.keys()])].sort().slice(-12);
  const monthly = periods.map((p) => ({
    period: p,
    income: incomeByMonth.get(p) ?? 0,
    commission: commByMonth.get(p) ?? 0,
  }));

  // Top del equipo (por comisiones acumuladas).
  const nameById = new Map((users.data ?? []).map((u) => [u.id, u]));
  const earnByUser = new Map<string, number>();
  for (const c of allComms.data ?? []) {
    earnByUser.set(c.user_id, (earnByUser.get(c.user_id) ?? 0) + Number(c.amount));
  }
  const topEarners = [...earnByUser.entries()]
    .map(([id, amount]) => ({
      name: nameById.get(id)?.full_name ?? "—",
      code: nameById.get(id)?.referral_code ?? "",
      amount,
    }))
    .sort((a, b) => b.amount - a.amount)
    .slice(0, 5);

  const recentMembers = (members.data ?? []).map((u) => ({
    name: u.full_name ?? "—",
    code: u.referral_code,
    date: u.created_at as string,
  }));

  return { monthly, recentMembers, topEarners };
}

// ----------------------------------------------------------------------------
// PENDIENTES GLOBALES (por activar / por colocar) — para el admin
// ----------------------------------------------------------------------------
export interface PendingMember {
  id: string;
  code: string;
  name: string;
  sponsorName: string | null;
}

export async function getPendingGlobal(): Promise<{
  toPlace: PendingMember[];
  toActivate: PendingMember[];
}> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("users")
    .select("id, full_name, referral_code, placement_id, sponsor_id, can_earn");

  const byId = new Map((data ?? []).map((u) => [u.id, u]));
  const toPlace: PendingMember[] = [];
  const toActivate: PendingMember[] = [];

  for (const u of data ?? []) {
    const sponsorName = u.sponsor_id ? byId.get(u.sponsor_id)?.full_name ?? null : null;
    const m = { id: u.id, code: u.referral_code, name: u.full_name ?? "—", sponsorName };
    // "por colocar": tiene patrocinador pero no posición (no es raíz).
    if (!u.placement_id && u.sponsor_id) toPlace.push(m);
    if (!u.can_earn) toActivate.push(m);
  }
  return { toPlace, toActivate };
}

// ----------------------------------------------------------------------------
// COLA DE KYC
// ----------------------------------------------------------------------------
export interface KycRow {
  id: string;
  full_name: string | null;
  email: string;
  country: string | null;
  created_at: string;
}

// ----------------------------------------------------------------------------
// CONFIGURACIÓN DEL PLAN (constantes, rangos, productos, promociones)
// ----------------------------------------------------------------------------
export async function getCompensationConfig() {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("compensation_config")
    .select("key, value, description")
    .order("key");
  return (data ?? []).map((r) => ({
    key: r.key,
    value: typeof r.value === "string" ? r.value : JSON.stringify(r.value),
    description: r.description as string | null,
  }));
}

export async function getRanksRaw() {
  const supabase = createAdminClient();
  const { data } = await supabase.from("ranks").select("*").order("level");
  return data ?? [];
}

export async function getProductsRaw() {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("products")
    .select("*")
    .order("type")
    .order("enrollment_price");
  return data ?? [];
}

export async function getPromotions() {
  const supabase = createAdminClient();
  const { data } = await supabase.from("promotions").select("*").order("key");
  return data ?? [];
}

// ----------------------------------------------------------------------------
// RAÍCES DEL ÁRBOL (usuarios sin posición = topes del binario)
// ----------------------------------------------------------------------------
export interface TreeRoot {
  id: string;
  name: string;
  code: string;
  childCount: number;
}

export async function getTreeRoots(): Promise<TreeRoot[]> {
  const supabase = createAdminClient();
  const { data: roots } = await supabase
    .from("users")
    .select("id, full_name, referral_code")
    .is("placement_id", null);

  const out: TreeRoot[] = [];
  for (const r of roots ?? []) {
    const { count } = await supabase
      .from("users")
      .select("*", { count: "exact", head: true })
      .eq("placement_id", r.id);
    out.push({
      id: r.id,
      name: r.full_name ?? "—",
      code: r.referral_code,
      childCount: count ?? 0,
    });
  }
  // Primero los que tienen red.
  return out.sort((a, b) => b.childCount - a.childCount);
}

/** Usuarios sin KYC verificado (cola de verificación pendiente). */
export async function getKycQueue(): Promise<KycRow[]> {
  const supabase = createAdminClient();
  const { data } = await supabase
    .from("users")
    .select("id, full_name, email, country, created_at")
    .eq("kyc_verified", false)
    .order("created_at", { ascending: true })
    .limit(200);
  return (data ?? []) as KycRow[];
}
```

### 11. `lib/data/store.ts`
Catálogo de tienda, pedidos del usuario, todos los pedidos (admin) y cupones.
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

### 12. `lib/data/tree.ts`
Construye el subárbol binario o de patrocinio de una raíz hasta N niveles.
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


---

# PARTE 6 — PANTALLAS (VISTAS)
Cada archivo es una página Server Component.

### 13. `app/(admin)/admin/page.tsx`
**Dashboard.** Métricas principales, % payout con banner de alerta si supera el umbral, miembros activos, pendientes por colocar/activar, gráfica ingresos vs comisión, resumen de retiros y nuevos miembros.
```tsx
import Link from "next/link";
import { getAdminDashboard, getPendingGlobal, getAdminCharts } from "@/lib/data/admin";
import { Card } from "@/components/ui/Card";
import { formatUSD } from "@/lib/utils";
import { AlertTriangle, Wallet, CheckCircle2, Clock, TrendingUp, Users, User } from "lucide-react";

export const metadata = { title: "Admin — TRIADA" };

export default async function AdminHome() {
  const [d, pending, charts] = await Promise.all([
    getAdminDashboard(),
    getPendingGlobal(),
    getAdminCharts(),
  ]);
  const payoutLabel = `${(d.payoutPct * 100).toFixed(1)}%`;
  const pendienteComisiones = d.commissionsTotal - d.commissionsPaid;

  return (
    <div className="space-y-6">
      <div>
        <p className="label">Administración</p>
        <h1 className="mt-1 text-2xl font-extrabold text-cream">Hola, Administrador</h1>
      </div>

      {d.alert && (
        <div className="flex items-start gap-3 rounded-card border border-danger/40 bg-danger/10 p-4">
          <AlertTriangle size={20} className="mt-0.5 shrink-0 text-danger" />
          <div>
            <p className="text-sm font-semibold text-danger">Payout alto: {payoutLabel}</p>
            <p className="mt-1 text-xs text-muted">
              Supera el umbral ({(d.alertThreshold * 100).toFixed(0)}%). Revisa la salud del plan.
            </p>
          </div>
        </div>
      )}

      {/* Métricas principales */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <MiniStat icon={<TrendingUp size={18} />} label="Facturación" value={formatUSD(d.billing)} accent />
        <MiniStat icon={<Wallet size={18} />} label="Comisiones (total)" value={formatUSD(d.commissionsTotal)} />
        <MiniStat icon={<CheckCircle2 size={18} />} label="Comisiones pagadas" value={formatUSD(d.commissionsPaid)} />
        <MiniStat icon={<Clock size={18} />} label="Comisiones por pagar" value={formatUSD(pendienteComisiones)} />
      </div>

      {/* Payout + pendientes */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
        <Card>
          <p className="label">% Payout</p>
          <p className={`mt-2 font-display text-3xl font-extrabold ${d.alert ? "text-danger" : "text-gold"}`}>{payoutLabel}</p>
          <p className="mt-1 text-xs text-muted">objetivo ≈16% · umbral {(d.alertThreshold * 100).toFixed(0)}%</p>
        </Card>
        <Metric label="Miembros activos" value={d.membersActive} sub={`${d.membersTotal} en total`} />
        <Link href="/admin/arbol" className="block">
          <Metric label="Por colocar" value={pending.toPlace.length} sub="clic para ir al árbol" accent="cobalt" />
        </Link>
        <Metric label="Por activar" value={pending.toActivate.length} sub="sin cobro aún" accent="gold" />
      </div>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-3">
        {/* Ingresos vs Comisión */}
        <Card className="lg:col-span-2">
          <h2 className="mb-4 text-lg font-bold text-cream">Ingresos vs Comisión por mes</h2>
          <IncomeVsCommission data={charts.monthly} />
          <div className="mt-3 flex gap-4 text-xs text-muted">
            <span className="flex items-center gap-1"><span className="h-2 w-3 rounded bg-cobalt" /> Ingresos</span>
            <span className="flex items-center gap-1"><span className="h-2 w-3 rounded bg-gold" /> Comisión</span>
          </div>
        </Card>

        {/* Resumen de pagos (donut) */}
        <Card>
          <h2 className="mb-4 text-lg font-bold text-cream">Resumen de retiros</h2>
          <PaymentsDonut wd={d.withdrawals} />
        </Card>
      </div>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        {/* Rendimiento del equipo */}
        <Card>
          <h2 className="mb-3 text-base font-bold text-cream">Rendimiento del equipo</h2>
          {charts.topEarners.length === 0 ? (
            <p className="text-sm text-muted">Sin datos.</p>
          ) : (
            <ul className="space-y-3">
              {charts.topEarners.map((t, i) => (
                <li key={i} className="flex items-center justify-between">
                  <div className="flex items-center gap-2">
                    <span className="flex h-7 w-7 items-center justify-center rounded-full bg-navyDeep"><User size={13} className="text-muted" /></span>
                    <div>
                      <p className="text-sm text-cream">{t.name}</p>
                      <p className="text-[11px] text-muted">{t.code}</p>
                    </div>
                  </div>
                  <span className="text-sm font-semibold text-success">{formatUSD(t.amount)}</span>
                </li>
              ))}
            </ul>
          )}
        </Card>

        {/* Nuevos miembros */}
        <Card>
          <h2 className="mb-3 text-base font-bold text-cream">Nuevos miembros</h2>
          {charts.recentMembers.length === 0 ? (
            <p className="text-sm text-muted">Sin miembros.</p>
          ) : (
            <ul className="space-y-2">
              {charts.recentMembers.map((m, i) => (
                <li key={i} className="flex items-center justify-between text-sm">
                  <div className="flex items-center gap-2">
                    <span className="flex h-7 w-7 items-center justify-center rounded-full bg-navyDeep"><User size={13} className="text-muted" /></span>
                    <div>
                      <p className="text-cream">{m.name}</p>
                      <p className="text-[11px] text-muted">{m.code}</p>
                    </div>
                  </div>
                  <span className="text-[11px] text-muted">{new Date(m.date).toLocaleDateString("es")}</span>
                </li>
              ))}
            </ul>
          )}
        </Card>
      </div>
    </div>
  );
}

function MiniStat({ icon, label, value, accent = false }: { icon: React.ReactNode; label: string; value: string; accent?: boolean }) {
  return (
    <Card>
      <div className="flex items-center justify-between">
        <p className="label">{label}</p>
        <span className={`flex h-8 w-8 items-center justify-center rounded-lg ${accent ? "bg-gold/15 text-gold" : "bg-navyDeep text-cobalt"}`}>{icon}</span>
      </div>
      <p className={`mt-2 font-display text-2xl font-extrabold ${accent ? "text-gold" : "text-cream"}`}>{value}</p>
    </Card>
  );
}

function Metric({ label, value, sub, accent = "cream" }: { label: string; value: number; sub: string; accent?: "cream" | "gold" | "cobalt" }) {
  const color = { cream: "text-cream", gold: "text-gold", cobalt: "text-cobalt" }[accent];
  return (
    <Card>
      <p className="label">{label}</p>
      <p className={`mt-2 font-display text-3xl font-extrabold ${color}`}>{value}</p>
      <p className="mt-1 text-xs text-muted">{sub}</p>
    </Card>
  );
}

/** Barras agrupadas ingresos vs comisión. */
function IncomeVsCommission({ data }: { data: { period: string; income: number; commission: number }[] }) {
  if (data.length === 0) return <p className="text-sm text-muted">Sin datos.</p>;
  const max = Math.max(...data.flatMap((d) => [d.income, d.commission]), 1);
  return (
    <div className="flex items-end justify-between gap-2" style={{ height: 200 }}>
      {data.map((d, i) => (
        <div key={i} className="flex flex-1 flex-col items-center justify-end gap-1">
          <div className="flex items-end gap-0.5" style={{ height: 160 }}>
            <div className="w-2.5 rounded-t bg-cobalt" style={{ height: Math.max(2, (d.income / max) * 160) }} title={`Ingresos ${formatUSD(d.income)}`} />
            <div className="w-2.5 rounded-t bg-gold" style={{ height: Math.max(2, (d.commission / max) * 160) }} title={`Comisión ${formatUSD(d.commission)}`} />
          </div>
          <span className="text-[9px] text-muted">{d.period.slice(2)}</span>
        </div>
      ))}
    </div>
  );
}

/** Donut de retiros por estado. */
function PaymentsDonut({ wd }: { wd: { pending: number; approved: number; paid: number; rejected: number } }) {
  const segs = [
    { label: "Pagado", value: wd.paid, color: "#2ECC71" },
    { label: "Aprobado", value: wd.approved, color: "#0353A4" },
    { label: "Pendiente", value: wd.pending, color: "#C2B280" },
    { label: "Rechazado", value: wd.rejected, color: "#E74C3C" },
  ];
  const total = segs.reduce((s, x) => s + x.value, 0);
  const C = 100; // circunferencia normalizada
  let offset = 0;

  return (
    <div className="flex items-center gap-5">
      <svg viewBox="0 0 42 42" className="h-32 w-32 -rotate-90">
        <circle cx="21" cy="21" r="15.915" fill="none" stroke="#1A3570" strokeWidth="6" />
        {total > 0 &&
          segs.map((s, i) => {
            const len = (s.value / total) * C;
            const el = (
              <circle key={i} cx="21" cy="21" r="15.915" fill="none" stroke={s.color} strokeWidth="6"
                strokeDasharray={`${len} ${C - len}`} strokeDashoffset={-offset} />
            );
            offset += len;
            return el;
          })}
        <text x="21" y="22" textAnchor="middle" className="fill-cream" style={{ fontSize: 6, fontWeight: 800, transform: "rotate(90deg)", transformOrigin: "center" }}>
          {total}
        </text>
      </svg>
      <ul className="space-y-1.5 text-sm">
        {segs.map((s) => (
          <li key={s.label} className="flex items-center gap-2">
            <span className="h-2.5 w-2.5 rounded-full" style={{ backgroundColor: s.color }} />
            <span className="text-muted">{s.label}</span>
            <span className="ml-auto font-medium text-cream">{s.value}</span>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 14. `app/(admin)/admin/miembros/page.tsx`
**Lista de miembros** con búsqueda/filtros, insignia de rango y enlace a la ficha.
```tsx
import Link from "next/link";
import { getMembers } from "@/lib/data/admin";
import { Card } from "@/components/ui/Card";
import { RankBadge } from "@/components/ui/RankBadge";
import { rankNameFor } from "@/lib/ranks";

export const metadata = { title: "Miembros — TRIADA Admin" };

const STATUS_STYLE: Record<string, string> = {
  active: "bg-success/15 text-success",
  inactive: "bg-muted/15 text-muted",
  suspended: "bg-danger/15 text-danger",
};

export default async function MiembrosPage({
  searchParams,
}: {
  searchParams: { q?: string; status?: string };
}) {
  const members = await getMembers({
    search: searchParams.q,
    status: searchParams.status,
  });

  return (
    <div className="space-y-6">
      <div>
        <p className="label">Gestión de Miembros</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Miembros</h1>
      </div>

      {/* Búsqueda + filtro (form GET) */}
      <form className="flex flex-wrap gap-3">
        <input
          name="q"
          defaultValue={searchParams.q ?? ""}
          placeholder="Buscar por nombre, correo o código…"
          className="min-w-64 flex-1 rounded-md border border-border bg-navyDeep px-3 py-2 text-sm text-cream placeholder:text-muted/60 outline-none focus:border-gold"
        />
        <select
          name="status"
          defaultValue={searchParams.status ?? ""}
          className="rounded-md border border-border bg-navyDeep px-3 py-2 text-sm text-cream outline-none focus:border-gold"
        >
          <option value="">Todos los estados</option>
          <option value="active">Activos</option>
          <option value="inactive">Inactivos</option>
          <option value="suspended">Suspendidos</option>
        </select>
        <button className="rounded-md border border-gold px-4 py-2 text-sm font-semibold text-gold hover:bg-gold/10">
          Buscar
        </button>
      </form>

      <Card>
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">Miembro</th>
                <th className="pb-2 font-semibold">Código</th>
                <th className="pb-2 font-semibold">País</th>
                <th className="pb-2 font-semibold">Rango</th>
                <th className="pb-2 font-semibold">Llaves</th>
                <th className="pb-2 font-semibold">KYC</th>
                <th className="pb-2 font-semibold">Estado</th>
              </tr>
            </thead>
            <tbody>
              {members.map((m) => (
                <tr key={m.id} className="border-b border-border/50 hover:bg-card/50">
                  <td className="py-2.5">
                    <Link href={`/admin/miembros/${m.id}`} className="text-cream hover:text-gold">
                      {m.full_name ?? "—"}
                    </Link>
                    <p className="text-[11px] text-muted">{m.email}</p>
                  </td>
                  <td className="py-2.5 text-muted">{m.referral_code}</td>
                  <td className="py-2.5 text-muted">{m.country ?? "—"}</td>
                  <td className="py-2.5">
                    <span className="flex items-center gap-2">
                      <RankBadge level={m.current_rank} size="xs" />
                      <span className="text-cream">
                        {m.current_rank > 0 ? rankNameFor(m.current_rank) : "—"}
                      </span>
                    </span>
                  </td>
                  <td className="py-2.5 text-xs">
                    <span className={m.can_distribute ? "text-gold" : "text-muted/40"}>D</span>{" "}
                    <span className={m.can_earn ? "text-gold" : "text-muted/40"}>C</span>
                  </td>
                  <td className="py-2.5">
                    {m.kyc_verified ? (
                      <span className="text-success">✓</span>
                    ) : (
                      <span className="text-muted/50">—</span>
                    )}
                  </td>
                  <td className="py-2.5">
                    <span
                      className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${
                        STATUS_STYLE[m.status] ?? ""
                      }`}
                    >
                      {m.status}
                    </span>
                  </td>
                </tr>
              ))}
              {members.length === 0 && (
                <tr>
                  <td colSpan={7} className="py-6 text-center text-sm text-muted">
                    No se encontraron miembros.
                  </td>
                </tr>
              )}
            </tbody>
          </table>
        </div>
      </Card>
    </div>
  );
}
```

### 15. `app/(admin)/admin/miembros/[id]/page.tsx`
**Ficha de miembro**: datos, rango, programas activos, billetera; botones activar / desactivar / suspender y aprobar KYC (cada uno es un `<form action={...bind(null, id, ...)}>`).
```tsx
import Link from "next/link";
import { notFound } from "next/navigation";
import { getMemberDetail } from "@/lib/data/admin";
import { setMemberStatusAction, approveKycAction } from "@/app/(admin)/admin/actions";
import { Card, StatCard } from "@/components/ui/Card";
import { RankBadge } from "@/components/ui/RankBadge";
import { formatUSD } from "@/lib/utils";
import { ArrowLeft } from "lucide-react";

export const metadata = { title: "Ficha de miembro — TRIADA Admin" };

export default async function MiembroDetallePage({
  params,
}: {
  params: { id: string };
}) {
  const m = await getMemberDetail(params.id);
  if (!m) notFound();
  const p = m.profile;

  return (
    <div className="space-y-8">
      <Link href="/admin/miembros" className="inline-flex items-center gap-2 text-sm text-muted hover:text-gold">
        <ArrowLeft size={16} /> Volver a miembros
      </Link>

      <div className="flex flex-wrap items-start justify-between gap-4">
        <div className="flex items-start gap-4">
          <RankBadge level={p.current_rank} size="lg" />
          <div>
            <p className="label">Ficha de miembro</p>
            <h1 className="mt-1 text-3xl font-extrabold text-cream">{p.full_name ?? "—"}</h1>
            <p className="mt-1 text-sm text-muted">
              {p.email} · código <span className="text-gold">{p.referral_code}</span>
            </p>
            <p className="mt-1 text-sm text-muted">
              Rango: <span className="text-cream">{m.rankName ?? "Sin rango"}</span> · Estado:{" "}
              <span className="text-cream">{p.status}</span>
            </p>
          </div>
        </div>

        {/* Acciones del admin */}
        <div className="flex flex-wrap gap-2">
          {p.status !== "active" && (
            <form action={setMemberStatusAction.bind(null, p.id, "active")}>
              <button className="rounded-md border border-success px-3 py-2 text-xs font-semibold text-success hover:bg-success/10">
                Activar
              </button>
            </form>
          )}
          {p.status !== "inactive" && (
            <form action={setMemberStatusAction.bind(null, p.id, "inactive")}>
              <button className="rounded-md border border-muted px-3 py-2 text-xs font-semibold text-muted hover:bg-muted/10">
                Desactivar
              </button>
            </form>
          )}
          {p.status !== "suspended" && (
            <form action={setMemberStatusAction.bind(null, p.id, "suspended")}>
              <button className="rounded-md border border-danger px-3 py-2 text-xs font-semibold text-danger hover:bg-danger/10">
                Suspender
              </button>
            </form>
          )}
          {!p.kyc_verified && (
            <form action={approveKycAction.bind(null, p.id)}>
              <button className="rounded-md border border-gold px-3 py-2 text-xs font-semibold text-gold hover:bg-gold/10">
                Aprobar KYC
              </button>
            </form>
          )}
        </div>
      </div>

      {/* Llaves */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-3">
        <KeyChip on={p.can_distribute || p.can_earn} label="Posición" sub={p.can_distribute || p.can_earn ? "Sí" : "—"} />
        <KeyChip on={p.can_distribute} label="Distribución" sub={p.can_distribute ? "Activa" : "Inactiva"} />
        <KeyChip on={p.can_earn} label="Cobro" sub={p.can_earn ? "Activa" : "Inactiva"} />
      </div>

      {/* Métricas */}
      <div className="grid grid-cols-1 gap-4 sm:grid-cols-4">
        <StatCard label="Saldo" value={formatUSD(m.wallet?.balance ?? 0)} />
        <StatCard label="Total ganado" value={formatUSD(m.wallet?.total_earned ?? 0)} />
        <StatCard label="Comisiones (total)" value={formatUSD(m.commissionsTotal)} />
        <StatCard label="Directos" value={String(m.directs)} />
      </div>

      {/* Programas */}
      <Card>
        <h2 className="mb-3 text-lg font-bold text-cream">Programas activos</h2>
        {m.memberships.length === 0 ? (
          <p className="text-sm text-muted">Sin programas activos.</p>
        ) : (
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">Programa</th>
                <th className="pb-2 font-semibold">Tipo</th>
                <th className="pb-2 font-semibold">Puntos</th>
                <th className="pb-2 font-semibold">Vence</th>
              </tr>
            </thead>
            <tbody>
              {m.memberships.map((mm, i) => (
                <tr key={i} className="border-b border-border/50">
                  <td className="py-2.5 text-cream">{mm.program_code ?? "—"}</td>
                  <td className="py-2.5 text-muted">{mm.type === "ibo" ? "IBO" : "Servicio"}</td>
                  <td className="py-2.5 text-cream">{mm.points}</td>
                  <td className="py-2.5 text-muted">
                    {mm.expires_at ? new Date(mm.expires_at).toLocaleDateString("es") : "—"}
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

function KeyChip({ on, label, sub }: { on: boolean; label: string; sub: string }) {
  return (
    <div className={`rounded-card border p-4 ${on ? "border-gold/40 bg-gold/5" : "border-border bg-navyDeep"}`}>
      <p className="font-display text-sm font-semibold text-cream">{label}</p>
      <p className={`mt-1 text-xs ${on ? "text-success" : "text-muted"}`}>{sub}</p>
    </div>
  );
}
```

### 16. `app/(admin)/admin/arbol/page.tsx`
**Árbol genealógico del admin.** Selector de raíz, árbol interactivo y colocación manual de miembros pendientes en cualquier posición.
```tsx
import { getTreeData } from "@/lib/data/tree";
import { getTreeRoots, getPendingGlobal } from "@/lib/data/admin";
import { adminLoadTreeAction, adminPlaceMemberAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";
import { GenealogyTree } from "@/components/tree/GenealogyTree";

export const metadata = { title: "Árbol — TRIADA Admin" };

export default async function AdminArbolPage({
  searchParams,
}: {
  searchParams: { root?: string };
}) {
  const [roots, pending] = await Promise.all([getTreeRoots(), getPendingGlobal()]);
  const rootId = searchParams.root ?? roots[0]?.id;
  const root = rootId ? await getTreeData(rootId, "binary") : null;

  const pendingToPlace = pending.toPlace.map((m) => ({
    id: m.id,
    code: m.code,
    name: m.name,
  }));

  return (
    <div className="space-y-6">
      <div>
        <p className="label">Árbol Genealógico</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Organización</h1>
        <p className="mt-1 text-sm text-muted">
          Pasa el mouse para ver datos; haz clic en un nodo y luego en un{" "}
          <span className="text-cobalt">+</span> para colocar a un miembro pendiente.
        </p>
      </div>

      {roots.length > 1 && (
        <form className="flex items-center gap-2">
          <label className="text-sm text-muted">Raíz:</label>
          <select
            name="root"
            defaultValue={rootId}
            className="rounded-md border border-border bg-navyDeep px-3 py-2 text-sm text-cream outline-none focus:border-gold"
          >
            {roots.map((r) => (
              <option key={r.id} value={r.id}>
                {r.code} — {r.name} ({r.childCount})
              </option>
            ))}
          </select>
          <button className="rounded-md border border-gold px-3 py-2 text-xs font-semibold text-gold hover:bg-gold/10">
            Ver
          </button>
        </form>
      )}

      <Card>
        {root ? (
          <GenealogyTree
            initialRoot={root}
            rootId={rootId!}
            load={adminLoadTreeAction}
            place={adminPlaceMemberAction}
            pending={pendingToPlace}
          />
        ) : (
          <p className="text-sm text-muted">No hay miembros en el árbol todavía.</p>
        )}
      </Card>
    </div>
  );
}
```

### 17. `app/(admin)/admin/pedidos/page.tsx`
**Pedidos de la tienda.** Pedidos de todos los usuarios con ítems, descuento, envío y cupón; botones para cambiar el estado (cada cambio queda en `status_history`); tabla de cupones con usos.
```tsx
import { getAllStoreOrders, getCoupons } from "@/lib/data/store";
import { updateStoreOrderStatusAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";
import { formatUSD } from "@/lib/utils";

export const metadata = { title: "Pedidos — TRIADA Admin" };

const STATUSES = ["received", "paid", "preparing", "shipped", "delivered", "cancelled"] as const;

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
  const [orders, coupons] = await Promise.all([getAllStoreOrders(), getCoupons()]);

  const totalSold = orders
    .filter((o) => o.paymentStatus === "paid")
    .reduce((s, o) => s + o.total, 0);

  return (
    <div className="space-y-8">
      <div>
        <p className="label">Tienda</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Pedidos</h1>
        <p className="mt-1 text-sm text-muted">
          {orders.length} pedidos · {formatUSD(totalSold)} vendidos
        </p>
      </div>

      <Card>
        {orders.length === 0 ? (
          <p className="py-8 text-center text-sm text-muted">Aún no hay pedidos.</p>
        ) : (
          <div className="space-y-3">
            {orders.map((o) => (
              <div key={o.id} className="rounded-card border border-border bg-card p-3">
                <div className="flex flex-wrap items-center justify-between gap-2">
                  <div className="min-w-0">
                    <p className="font-mono text-sm text-cream">{o.orderNumber}</p>
                    <p className="text-xs text-muted">
                      {o.userName} · {o.userEmail} · {new Date(o.createdAt).toLocaleString()}
                    </p>
                  </div>
                  <div className="flex items-center gap-2">
                    <span className="text-sm font-bold text-cream">{formatUSD(o.total)}</span>
                    <span
                      className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${
                        STATUS_STYLE[o.status] ?? "bg-muted/15 text-muted"
                      }`}
                    >
                      {STATUS_LABEL[o.status] ?? o.status}
                    </span>
                  </div>
                </div>

                <ul className="mt-2 space-y-0.5 text-xs text-muted">
                  {o.items.map((i, idx) => (
                    <li key={idx}>
                      {i.qty}× {i.name} — {formatUSD(i.price * i.qty)} · {i.points * i.qty} pts
                    </li>
                  ))}
                </ul>

                {(o.discount > 0 || o.shipping > 0) && (
                  <p className="mt-1 text-[11px] text-muted">
                    Subtotal {formatUSD(o.subtotal)}
                    {o.discount > 0 && ` · Descuento −${formatUSD(o.discount)}`}
                    {o.couponCode && ` (${o.couponCode})`}
                    {o.shipping > 0 && ` · Envío ${formatUSD(o.shipping)}`}
                  </p>
                )}

                <div className="mt-2 flex flex-wrap gap-1.5 border-t border-border pt-2">
                  {STATUSES.filter((s) => s !== o.status).map((s) => (
                    <form key={s} action={updateStoreOrderStatusAction.bind(null, o.id, s)}>
                      <button className="rounded-md border border-border px-2 py-1 text-[10px] text-muted hover:border-gold hover:text-cream">
                        → {STATUS_LABEL[s]}
                      </button>
                    </form>
                  ))}
                </div>
              </div>
            ))}
          </div>
        )}
      </Card>

      <Card>
        <h2 className="mb-3 text-lg font-bold text-cream">Cupones</h2>
        {coupons.length === 0 ? (
          <p className="text-sm text-muted">No hay cupones.</p>
        ) : (
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">Código</th>
                <th className="pb-2 font-semibold">Descuento</th>
                <th className="pb-2 font-semibold">Mínimo</th>
                <th className="pb-2 font-semibold">Usos</th>
                <th className="pb-2 font-semibold">Estado</th>
              </tr>
            </thead>
            <tbody>
              {coupons.map((c) => (
                <tr key={c.code} className="border-b border-border/50">
                  <td className="py-2 font-mono text-cream">{c.code}</td>
                  <td className="py-2 text-cream">
                    {c.type === "percent" ? `${c.value}%` : formatUSD(c.value)}
                  </td>
                  <td className="py-2 text-muted">{formatUSD(c.minSubtotal)}</td>
                  <td className="py-2 text-muted">
                    {c.uses} / {c.maxUses}
                  </td>
                  <td className="py-2">
                    <span
                      className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${
                        c.active ? "bg-success/15 text-success" : "bg-muted/15 text-muted"
                      }`}
                    >
                      {c.active ? "Activo" : "Inactivo"}
                    </span>
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

### 18. `app/(admin)/admin/kyc/page.tsx`
**Cola de KYC.** Miembros sin verificar; Aprobar desbloquea sus retiros.
```tsx
import Link from "next/link";
import { getKycQueue } from "@/lib/data/admin";
import { approveKycAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";

export const metadata = { title: "KYC — TRIADA Admin" };

export default async function KycPage() {
  const queue = await getKycQueue();

  return (
    <div className="space-y-6">
      <div>
        <p className="label">Verificación KYC</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Cola de KYC</h1>
        <p className="mt-1 text-sm text-muted">
          Miembros sin verificar. Aprobar desbloquea sus retiros.
        </p>
      </div>

      <Card>
        {queue.length === 0 ? (
          <p className="text-sm text-muted">No hay verificaciones pendientes. ✓</p>
        ) : (
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">Miembro</th>
                <th className="pb-2 font-semibold">País</th>
                <th className="pb-2 font-semibold">Registrado</th>
                <th className="pb-2 font-semibold text-right">Acción</th>
              </tr>
            </thead>
            <tbody>
              {queue.map((u) => (
                <tr key={u.id} className="border-b border-border/50">
                  <td className="py-2.5">
                    <Link href={`/admin/miembros/${u.id}`} className="text-cream hover:text-gold">
                      {u.full_name ?? "—"}
                    </Link>
                    <p className="text-[11px] text-muted">{u.email}</p>
                  </td>
                  <td className="py-2.5 text-muted">{u.country ?? "—"}</td>
                  <td className="py-2.5 text-muted">
                    {new Date(u.created_at).toLocaleDateString("es")}
                  </td>
                  <td className="py-2.5 text-right">
                    <form action={approveKycAction.bind(null, u.id)} className="inline">
                      <button className="rounded-md border border-gold px-3 py-1.5 text-xs font-semibold text-gold hover:bg-gold/10">
                        Aprobar
                      </button>
                    </form>
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

### 19. `app/(admin)/admin/cierres/page.tsx`
**Cierres y comisiones.** Botones de cierre mensual y pago semanal + tabla de comisiones del período por vía (Directa N1/N2, Binario, Rango, Igualación, Estilo de vida).
```tsx
import { getCommissionsForPeriod } from "@/lib/data/admin";
import { currentPeriod } from "@/lib/engine/period";
import { Card } from "@/components/ui/Card";
import { CierreControls } from "@/components/admin/CierreControls";
import { formatUSD } from "@/lib/utils";

export const metadata = { title: "Cierres — TRIADA Admin" };

const TYPE_LABEL: Record<string, string> = {
  direct_l1: "Directa N1",
  direct_l2: "Directa N2",
  binary: "Binario",
  rank: "Rango",
  matching: "Igualación",
  lifestyle: "Estilo de vida",
};

export default async function CierresPage() {
  const period = currentPeriod();
  const comms = await getCommissionsForPeriod(period);
  const total = comms.reduce((s, c) => s + c.amount, 0);

  return (
    <div className="space-y-8">
      <div>
        <p className="label">Cierres y Comisiones</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Cierre {period}</h1>
        <p className="mt-1 text-sm text-muted">
          Ejecuta el cierre mensual (binario, rango, matching, estilo de vida) y los pagos.
        </p>
      </div>

      <Card premium>
        <CierreControls />
      </Card>

      <Card>
        <div className="mb-3 flex items-center justify-between">
          <h2 className="text-lg font-bold text-cream">Comisiones del período</h2>
          <span className="text-sm text-gold">{formatUSD(total)}</span>
        </div>
        {comms.length === 0 ? (
          <p className="text-sm text-muted">Aún no hay comisiones este período. Ejecuta el cierre.</p>
        ) : (
          <div className="overflow-x-auto">
            <table className="w-full text-sm">
              <thead>
                <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                  <th className="pb-2 font-semibold">Miembro</th>
                  <th className="pb-2 font-semibold">Vía</th>
                  <th className="pb-2 font-semibold">Monto</th>
                  <th className="pb-2 font-semibold">Estado</th>
                </tr>
              </thead>
              <tbody>
                {comms.map((c, i) => (
                  <tr key={i} className="border-b border-border/50">
                    <td className="py-2 text-cream">{c.code}</td>
                    <td className="py-2 text-muted">{TYPE_LABEL[c.type] ?? c.type}</td>
                    <td className="py-2 font-medium text-cream">{formatUSD(c.amount)}</td>
                    <td className="py-2 text-muted">{c.status}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </Card>
    </div>
  );
}
```

### 20. `app/(admin)/admin/pagos/page.tsx`
**Cola de retiros USDT.** Aprobar, marcar pagado (con tx_hash) o rechazar (devuelve el saldo).
```tsx
import { getWithdrawalsQueue } from "@/lib/data/admin";
import { setWithdrawalAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";
import { formatUSD } from "@/lib/utils";

export const metadata = { title: "Pagos — TRIADA Admin" };

const STATUS_STYLE: Record<string, string> = {
  pending: "bg-gold/15 text-gold",
  approved: "bg-cobalt/15 text-cobalt",
  paid: "bg-success/15 text-success",
  rejected: "bg-danger/15 text-danger",
};

export default async function PagosPage() {
  const wds = await getWithdrawalsQueue();

  return (
    <div className="space-y-6">
      <div>
        <p className="label">Pagos / Retiros USDT</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Cola de retiros</h1>
        <p className="mt-1 text-sm text-muted">
          Aprueba, paga (registra el hash) o rechaza (devuelve el saldo) las solicitudes.
        </p>
      </div>

      <Card>
        {wds.length === 0 ? (
          <p className="text-sm text-muted">No hay solicitudes de retiro.</p>
        ) : (
          <div className="space-y-4">
            {wds.map((w) => (
              <div key={w.id} className="rounded-card border border-border bg-navyDeep/40 p-4">
                <div className="flex flex-wrap items-center justify-between gap-3">
                  <div>
                    <p className="text-sm text-cream">
                      {w.name} · <span className="text-muted">{w.code}</span>
                    </p>
                    <p className="text-[11px] text-muted break-all">{w.address}</p>
                  </div>
                  <div className="text-right">
                    <p className="font-display text-lg font-extrabold text-gold">{formatUSD(w.amount)}</p>
                    <span className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${STATUS_STYLE[w.status] ?? ""}`}>
                      {w.status}
                    </span>
                  </div>
                </div>

                {(w.status === "pending" || w.status === "approved") && (
                  <form action={setWithdrawalAction} className="mt-3 flex flex-wrap items-center gap-2">
                    <input type="hidden" name="id" value={w.id} />
                    <input
                      name="tx_hash"
                      placeholder="tx_hash (al pagar)"
                      className="flex-1 rounded-md border border-border bg-navyDeep px-3 py-1.5 text-xs text-cream outline-none focus:border-gold"
                    />
                    {w.status === "pending" && (
                      <button name="status" value="approved" className="rounded-md border border-cobalt px-3 py-1.5 text-xs font-semibold text-cobalt hover:bg-cobalt/10">
                        Aprobar
                      </button>
                    )}
                    <button name="status" value="paid" className="rounded-md border border-success px-3 py-1.5 text-xs font-semibold text-success hover:bg-success/10">
                      Marcar pagado
                    </button>
                    <button name="status" value="rejected" className="rounded-md border border-danger px-3 py-1.5 text-xs font-semibold text-danger hover:bg-danger/10">
                      Rechazar
                    </button>
                  </form>
                )}
                {w.txHash && <p className="mt-2 text-[11px] text-muted break-all">tx: {w.txHash}</p>}
              </div>
            ))}
          </div>
        )}
      </Card>
    </div>
  );
}
```

### 21. `app/(admin)/admin/reportes/page.tsx`
**Reportes.** Salud del negocio: nuevos miembros por mes, miembros por rango y totales.
```tsx
import { getReports } from "@/lib/data/admin";
import { Card, StatCard } from "@/components/ui/Card";
import { RankBadge } from "@/components/ui/RankBadge";
import { formatUSD } from "@/lib/utils";

export const metadata = { title: "Reportes — TRIADA Admin" };

export default async function ReportesPage() {
  const r = await getReports();
  const maxNew = Math.max(...r.newByMonth.map((m) => m.count), 1);
  const maxRank = Math.max(...r.membersByRank.map((m) => m.count), 1);

  return (
    <div className="space-y-8">
      <div>
        <p className="label">Reportes</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Salud del negocio</h1>
      </div>

      <div className="grid grid-cols-1 gap-4 sm:grid-cols-3">
        <StatCard label="Facturación" value={formatUSD(r.billing)} />
        <StatCard label="Comisiones" value={formatUSD(r.commissionsTotal)} />
        <StatCard label="% Payout" value={`${(r.payoutPct * 100).toFixed(1)}%`} hint="objetivo ≈16%" />
      </div>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        {/* Crecimiento */}
        <Card>
          <h2 className="mb-4 text-lg font-bold text-cream">Nuevos miembros por mes</h2>
          {r.newByMonth.length === 0 ? (
            <p className="text-sm text-muted">Sin datos.</p>
          ) : (
            <div className="flex items-end justify-between gap-3" style={{ height: 150 }}>
              {r.newByMonth.map((m, i) => (
                <div key={i} className="flex flex-1 flex-col items-center justify-end">
                  <span className="mb-1 text-[10px] text-muted">{m.count}</span>
                  <div className="w-full rounded-t bg-cobalt" style={{ height: Math.max(4, (m.count / maxNew) * 120) }} />
                  <span className="mt-1 text-[10px] text-muted">{m.period.slice(5)}</span>
                </div>
              ))}
            </div>
          )}
        </Card>

        {/* Miembros por rango */}
        <Card>
          <h2 className="mb-4 text-lg font-bold text-cream">Miembros por rango</h2>
          <div className="space-y-2">
            {r.membersByRank.map((m) => (
              <div key={m.level} className="flex items-center gap-3">
                <RankBadge level={m.level} size="xs" />
                <span className="w-28 shrink-0 truncate text-xs text-muted">{m.name}</span>
                <div className="h-3 flex-1 overflow-hidden rounded-full bg-navyDeep">
                  <div className="h-full rounded-full bg-gold/80" style={{ width: `${(m.count / maxRank) * 100}%` }} />
                </div>
                <span className="w-8 text-right text-xs text-cream">{m.count}</span>
              </div>
            ))}
          </div>
        </Card>
      </div>
    </div>
  );
}
```

### 22. `app/(admin)/admin/configuracion/page.tsx`
**Configuración del plan.** Constantes de `compensation_config` y los 13 rangos de `ranks`, editables sin tocar código.
```tsx
import { getCompensationConfig, getRanksRaw } from "@/lib/data/admin";
import { updateConfigAction, updateRankAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";
import { RankBadge } from "@/components/ui/RankBadge";

export const metadata = { title: "Configuración — TRIADA Admin" };

const inputCls =
  "w-full rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold";

export default async function ConfiguracionPage() {
  const [config, ranks] = await Promise.all([getCompensationConfig(), getRanksRaw()]);

  return (
    <div className="space-y-8">
      <div>
        <p className="label">Plan de Compensación</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Configuración</h1>
        <p className="mt-1 text-sm text-muted">
          Edita las constantes y los rangos sin tocar código. El motor lee estos valores.
        </p>
      </div>

      {/* Constantes */}
      <Card>
        <h2 className="mb-4 text-lg font-bold text-cream">Constantes del plan</h2>
        <div className="grid grid-cols-1 gap-3 md:grid-cols-2">
          {config.map((c) => (
            <form key={c.key} action={updateConfigAction} className="flex items-end gap-2">
              <input type="hidden" name="key" value={c.key} />
              <label className="flex-1">
                <span className="block text-xs font-medium text-cream">{c.key}</span>
                {c.description && (
                  <span className="block text-[10px] text-muted">{c.description}</span>
                )}
                <input name="value" defaultValue={c.value} className={inputCls + " mt-1"} />
              </label>
              <button className="rounded-md border border-gold px-3 py-1.5 text-xs font-semibold text-gold hover:bg-gold/10">
                Guardar
              </button>
            </form>
          ))}
        </div>
      </Card>

      {/* Rangos */}
      <Card>
        <h2 className="mb-4 text-lg font-bold text-cream">Rangos (13 niveles)</h2>
        <div className="space-y-3 overflow-x-auto">
          {ranks.map((r) => (
            <form
              key={r.level}
              action={updateRankAction}
              className="flex flex-wrap items-end gap-2 border-b border-border/50 pb-3"
            >
              <input type="hidden" name="level" value={r.level} />
              <span className="flex w-36 shrink-0 items-center gap-2">
                <RankBadge level={r.level} size="sm" />
                <span className="min-w-0">
                  <span className="block text-[10px] text-muted">#{r.level}</span>
                  <span className="block truncate text-sm font-semibold text-cream">{r.name}</span>
                </span>
              </span>
              <Num label="Pts/pierna" name="points_per_leg" value={r.points_per_leg} />
              <Num label="% línea" name="max_pct_line" value={r.max_pct_line} step="0.01" />
              <Num label="Dir. I" name="min_directs_left" value={r.min_directs_left} w="w-16" />
              <Num label="Dir. D" name="min_directs_right" value={r.min_directs_right} w="w-16" />
              <Num label="Cheque" name="base_check" value={r.base_check} />
              <label className="w-40">
                <span className="block text-[10px] text-muted">Matching (csv)</span>
                <input
                  name="matching_levels"
                  defaultValue={(Array.isArray(r.matching_levels) ? r.matching_levels : []).join(",")}
                  className={inputCls}
                />
              </label>
              <Num label="Estilo $" name="lifestyle_bonus" value={r.lifestyle_bonus} />
              <Num label="Estilo dir." name="lifestyle_directs" value={r.lifestyle_directs} w="w-16" />
              <button className="rounded-md border border-gold px-3 py-1.5 text-xs font-semibold text-gold hover:bg-gold/10">
                Guardar
              </button>
            </form>
          ))}
        </div>
      </Card>
    </div>
  );
}

function Num({
  label,
  name,
  value,
  step,
  w = "w-24",
}: {
  label: string;
  name: string;
  value: number;
  step?: string;
  w?: string;
}) {
  return (
    <label className={w}>
      <span className="block text-[10px] text-muted">{label}</span>
      <input
        name={name}
        type="number"
        step={step ?? "1"}
        defaultValue={value}
        className="w-full rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold"
      />
    </label>
  );
}
```

### 23. `app/(admin)/admin/productos/page.tsx`
**Productos.** Tabla del catálogo, sección 'Presentación en la tienda' (imagen, descripción, insignia, acento, orden) y formulario de alta/edición por código.
```tsx
import { getProductsRaw } from "@/lib/data/admin";
import { upsertProductAction, toggleProductAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";
import { ProductMediaRow } from "@/components/admin/ProductMediaRow";
import { formatUSD } from "@/lib/utils";

export const metadata = { title: "Productos — TRIADA Admin" };

const inputCls =
  "w-full rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold";

export default async function ProductosPage() {
  const products = await getProductsRaw();

  return (
    <div className="space-y-8">
      <div>
        <p className="label">Catálogo</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Productos y programas</h1>
      </div>

      {/* Lista */}
      <Card>
        <div className="overflow-x-auto">
          <table className="w-full text-sm">
            <thead>
              <tr className="border-b border-border text-left text-[11px] uppercase tracking-wide text-muted">
                <th className="pb-2 font-semibold">Código</th>
                <th className="pb-2 font-semibold">Nombre</th>
                <th className="pb-2 font-semibold">Tipo</th>
                <th className="pb-2 font-semibold">Inscripción</th>
                <th className="pb-2 font-semibold">Renovación</th>
                <th className="pb-2 font-semibold">Puntos</th>
                <th className="pb-2 font-semibold">Fondo</th>
                <th className="pb-2 font-semibold">Estado</th>
              </tr>
            </thead>
            <tbody>
              {products.map((p) => (
                <tr key={p.id} className="border-b border-border/50">
                  <td className="py-2 text-cream">{p.code}</td>
                  <td className="py-2 text-cream">{p.name}</td>
                  <td className="py-2 text-muted">{p.type}</td>
                  <td className="py-2 text-cream">{formatUSD(Number(p.enrollment_price))}</td>
                  <td className="py-2 text-muted">{p.renewal_price ? formatUSD(Number(p.renewal_price)) : "—"}</td>
                  <td className="py-2 text-cream">{p.points}</td>
                  <td className="py-2 text-muted">{Number(p.fund_amount) > 0 ? formatUSD(Number(p.fund_amount)) : "—"}</td>
                  <td className="py-2">
                    <form action={toggleProductAction.bind(null, p.id, !p.is_active)}>
                      <button className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${p.is_active ? "bg-success/15 text-success" : "bg-muted/15 text-muted"}`}>
                        {p.is_active ? "Activo" : "Inactivo"}
                      </button>
                    </form>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </Card>

      {/* Presentación en la tienda: imagen, descripción, insignia */}
      <Card>
        <h2 className="mb-1 text-lg font-bold text-cream">Presentación en la tienda</h2>
        <p className="mb-4 text-xs text-muted">
          Lo que ve el distribuidor en /tienda. La imagen puede ser una URL externa o una ruta
          pública del proyecto (por ejemplo <span className="font-mono">/muestras/producto-nova.jpg</span>).
        </p>
        <div className="space-y-2">
          {products.map((p) => (
            <ProductMediaRow
              key={p.id}
              product={{
                id: p.id,
                code: p.code,
                name: p.name,
                imageUrl: p.image_url ?? null,
                description: p.description ?? null,
                badge: p.badge ?? null,
                accent: p.accent ?? "#C2B280",
                sort: Number(p.sort ?? 0),
              }}
            />
          ))}
        </div>
      </Card>

      {/* Crear / editar por código */}
      <Card premium>
        <h2 className="mb-3 text-lg font-bold text-cream">Crear o editar producto</h2>
        <p className="mb-4 text-xs text-muted">
          Si el código ya existe, se actualiza. Tipo: service / physical / ibo.
        </p>
        <form action={upsertProductAction} className="grid grid-cols-1 gap-3 sm:grid-cols-2 lg:grid-cols-4">
          <Field label="Código" name="code" placeholder="QUANTUM" required />
          <Field label="Nombre" name="name" placeholder="Quantum" required />
          <Field label="Tipo" name="type" placeholder="service" />
          <Field label="Inscripción ($)" name="enrollment_price" type="number" />
          <Field label="Renovación ($)" name="renewal_price" type="number" />
          <Field label="Puntos" name="points" type="number" />
          <Field label="Meses" name="duration_months" type="number" />
          <Field label="Fondo NO comisionable ($)" name="fund_amount" type="number" />
          <Field label="Imagen (URL o /ruta)" name="image_url" placeholder="/muestras/producto-nova.jpg" />
          <Field label="Insignia" name="badge" placeholder="Más vendido" />
          <Field label="Orden" name="sort" type="number" />
          <div className="flex items-end">
            <button className="w-full rounded-[4px] bg-gold px-4 py-2 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight">
              Guardar
            </button>
          </div>
        </form>
      </Card>
    </div>
  );
}

function Field({
  label,
  name,
  type = "text",
  placeholder,
  required,
}: {
  label: string;
  name: string;
  type?: string;
  placeholder?: string;
  required?: boolean;
}) {
  return (
    <label>
      <span className="block text-[10px] uppercase tracking-wide text-muted">{label}</span>
      <input name={name} type={type} step="0.01" placeholder={placeholder} required={required} className={inputCls + " mt-1"} />
    </label>
  );
}
```

### 24. `app/(admin)/admin/promociones/page.tsx`
**Promociones.** Activar/desactivar campañas temporales con fechas de inicio y fin.
```tsx
import { getPromotions } from "@/lib/data/admin";
import { savePromotionAction } from "@/app/(admin)/admin/actions";
import { Card } from "@/components/ui/Card";

export const metadata = { title: "Promociones — TRIADA Admin" };

function toDateInput(iso: string | null): string {
  if (!iso) return "";
  return new Date(iso).toISOString().slice(0, 10);
}

export default async function PromocionesPage() {
  const promos = await getPromotions();

  return (
    <div className="space-y-8">
      <div>
        <p className="label">Promociones</p>
        <h1 className="mt-1 text-3xl font-extrabold text-cream">Campañas temporales</h1>
        <p className="mt-1 text-sm text-muted">
          Actívalas por rango de fechas. El sistema aplica su efecto mientras estén vigentes.
        </p>
      </div>

      <div className="grid grid-cols-1 gap-6 lg:grid-cols-2">
        {promos.map((p) => (
          <Card key={p.key} premium={p.active}>
            <form action={savePromotionAction} className="space-y-4">
              <input type="hidden" name="key" value={p.key} />
              <div className="flex items-center justify-between">
                <div>
                  <h2 className="text-lg font-bold text-cream">{p.name}</h2>
                  <p className="text-[11px] text-muted">{p.key}</p>
                </div>
                <span className={`rounded-full px-2 py-0.5 text-[10px] font-semibold ${p.active ? "bg-success/15 text-success" : "bg-muted/15 text-muted"}`}>
                  {p.active ? "Activa" : "Inactiva"}
                </span>
              </div>

              <label className="flex items-center gap-2 text-sm text-cream">
                <input type="checkbox" name="active" defaultChecked={p.active} className="h-4 w-4 accent-[#C2B280]" />
                Activar promoción
              </label>

              <div className="grid grid-cols-2 gap-3">
                <label>
                  <span className="block text-[10px] uppercase tracking-wide text-muted">Desde</span>
                  <input type="date" name="starts_at" defaultValue={toDateInput(p.starts_at)} className="mt-1 w-full rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold" />
                </label>
                <label>
                  <span className="block text-[10px] uppercase tracking-wide text-muted">Hasta</span>
                  <input type="date" name="ends_at" defaultValue={toDateInput(p.ends_at)} className="mt-1 w-full rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold" />
                </label>
              </div>

              <p className="rounded-md border border-border bg-navyDeep px-3 py-2 text-xs text-muted">
                {p.key === "mes_nitro"
                  ? "Nuevos distribuidores con Nova + IBO reciben estatus de 100 pts su primer mes."
                  : "Descuento temporal en el precio de inscripción."}
              </p>

              <button className="w-full rounded-[4px] bg-gold px-4 py-2 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight">
                Guardar
              </button>
            </form>
          </Card>
        ))}
      </div>
    </div>
  );
}
```


---

# PARTE 7 — COMPONENTES DEL ADMIN
Piezas con interactividad en el cliente (`"use client"`) o reutilizables.

### 25. `components/admin/CierreControls.tsx`
Botones 'Ejecutar cierre mensual' y 'Validar + pago semanal'. Usa `useTransition` para mostrar el progreso y el resumen.
```tsx
"use client";

import { useState, useTransition } from "react";
import { runCloseAction, runPayoutAction } from "@/app/(admin)/admin/actions";

export function CierreControls() {
  const [msg, setMsg] = useState<string | null>(null);
  const [pending, startTransition] = useTransition();

  function close() {
    startTransition(async () => {
      const s = await runCloseAction();
      setMsg(`Cierre ${s.period}: ${s.usersProcessed} miembros, ${s.commissionsCreated} comisiones generadas.`);
    });
  }
  function payout() {
    startTransition(async () => {
      const s = await runPayoutAction();
      setMsg(`Pago: ${s.validated} comisiones validadas, ${s.usersPaid} pagados, $${s.totalDisbursed.toFixed(2)} desembolsados.`);
    });
  }

  return (
    <div>
      <div className="flex flex-wrap gap-3">
        <button
          onClick={close}
          disabled={pending}
          className="rounded-[4px] bg-gold px-6 py-2.5 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight disabled:opacity-50"
        >
          {pending ? "Procesando…" : "Ejecutar cierre mensual"}
        </button>
        <button
          onClick={payout}
          disabled={pending}
          className="rounded-[4px] border border-gold px-6 py-2.5 font-display text-xs font-semibold uppercase tracking-[0.1em] text-gold hover:bg-gold/10 disabled:opacity-50"
        >
          {pending ? "Procesando…" : "Validar + pago semanal"}
        </button>
      </div>
      {msg && (
        <p className="mt-3 rounded-md border border-success/30 bg-success/10 px-3 py-2 text-sm text-success">
          {msg}
        </p>
      )}
    </div>
  );
}
```

### 26. `components/admin/ProductMediaRow.tsx`
Fila de producto con edición en línea de imagen (URL o ruta pública; jpg/png/svg/webp), descripción, insignia, color de acento y orden.
```tsx
"use client";

import { useState } from "react";
import { ImageOff, Pencil, X } from "lucide-react";
import { updateProductMediaAction } from "@/app/(admin)/admin/actions";

export interface ProductMedia {
  id: string;
  code: string;
  name: string;
  imageUrl: string | null;
  description: string | null;
  badge: string | null;
  accent: string;
  sort: number;
}

const inputCls =
  "w-full rounded-md border border-border bg-navyDeep px-2 py-1.5 text-sm text-cream outline-none focus:border-gold";

/**
 * Fila del catálogo con edición en línea de la presentación en tienda:
 * imagen (jpg/png/svg/webp por URL o ruta pública), descripción, insignia,
 * color de acento y orden.
 */
export function ProductMediaRow({ product }: { product: ProductMedia }) {
  const [editing, setEditing] = useState(false);

  return (
    <div className="rounded-card border border-border bg-card p-3">
      <div className="flex items-start gap-3">
        <div className="h-16 w-16 shrink-0 overflow-hidden rounded-lg border border-border bg-navyDeep">
          {product.imageUrl ? (
            // eslint-disable-next-line @next/next/no-img-element
            <img src={product.imageUrl} alt={product.name} className="h-full w-full object-cover" />
          ) : (
            <div className="flex h-full w-full items-center justify-center text-muted">
              <ImageOff size={18} />
            </div>
          )}
        </div>

        <div className="min-w-0 flex-1">
          <div className="flex items-center gap-2">
            <p className="truncate text-sm font-semibold text-cream">{product.name}</p>
            <span className="rounded bg-navyDeep px-1.5 py-0.5 font-mono text-[10px] text-muted">
              {product.code}
            </span>
            {product.badge && (
              <span
                className="rounded-full px-2 py-0.5 text-[10px] font-bold text-goldInk"
                style={{ backgroundColor: product.accent }}
              >
                {product.badge}
              </span>
            )}
          </div>
          <p className="mt-1 line-clamp-2 text-[12px] text-muted">
            {product.description || "Sin descripción."}
          </p>
          <p className="mt-1 truncate text-[10px] text-muted/70">{product.imageUrl || "Sin imagen"}</p>
        </div>

        <button
          onClick={() => setEditing((e) => !e)}
          className="shrink-0 rounded-md border border-border p-1.5 text-muted hover:border-gold hover:text-cream"
          aria-label={editing ? "Cerrar" : "Editar"}
        >
          {editing ? <X size={14} /> : <Pencil size={14} />}
        </button>
      </div>

      {editing && (
        <form action={updateProductMediaAction} className="mt-3 grid grid-cols-1 gap-3 border-t border-border pt-3 sm:grid-cols-2">
          <input type="hidden" name="id" value={product.id} />
          <label className="sm:col-span-2">
            <span className="block text-[10px] uppercase tracking-wide text-muted">
              Imagen (URL o ruta: /muestras/archivo.png · admite jpg, png, svg, webp)
            </span>
            <input name="image_url" defaultValue={product.imageUrl ?? ""} className={inputCls + " mt-1"} />
          </label>
          <label className="sm:col-span-2">
            <span className="block text-[10px] uppercase tracking-wide text-muted">Descripción</span>
            <textarea
              name="description"
              rows={2}
              defaultValue={product.description ?? ""}
              className={inputCls + " mt-1 resize-none"}
            />
          </label>
          <label>
            <span className="block text-[10px] uppercase tracking-wide text-muted">Insignia</span>
            <input
              name="badge"
              defaultValue={product.badge ?? ""}
              placeholder="Más vendido"
              className={inputCls + " mt-1"}
            />
          </label>
          <label>
            <span className="block text-[10px] uppercase tracking-wide text-muted">Color de acento</span>
            <input name="accent" defaultValue={product.accent} className={inputCls + " mt-1"} />
          </label>
          <label>
            <span className="block text-[10px] uppercase tracking-wide text-muted">Orden</span>
            <input name="sort" type="number" defaultValue={product.sort} className={inputCls + " mt-1"} />
          </label>
          <div className="flex items-end">
            <button className="w-full rounded-[4px] bg-gold px-4 py-2 font-display text-xs font-semibold uppercase tracking-[0.1em] text-goldInk hover:bg-goldLight">
              Guardar
            </button>
          </div>
        </form>
      )}
    </div>
  );
}
```

### 27. `components/tree/GenealogyTree.tsx`
Árbol genealógico con conectores CSS y botón '+' para colocar pendientes. Lo usa `/admin/arbol`.
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

### 28. `components/ui/RankBadge.tsx`
Insignia PNG del rango (13 niveles) o marcador neutro 'sin rango'.
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

### 29. `lib/ranks.ts`
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


---

# PARTE 8 — DOMINIO: COLOCACIÓN Y ACTIVACIÓN
Reglas de negocio que usan las acciones del admin y la tienda.

### 30. `lib/domain/placement.ts`
Coloca a un miembro pendiente en una posición exacta del árbol binario (valida pierna libre y evita ciclos).
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

### 31. `lib/domain/keys.ts`
Las 3 llaves de activación: `has_position`, `can_distribute`, `can_earn`. Funciones puras y testeadas.
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

### 32. `lib/domain/activation.ts`
Activación al comprar: crea la orden pagada, la membresía, enciende las llaves, otorga créditos de IA y dispara comisiones directas.
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


---

# PARTE 9 — MOTOR DE COMISIONES
El núcleo del negocio. `calculations.ts` es puro (sin base de datos) y está cubierto por tests; el resto orquesta lecturas y escrituras.

### 33. `lib/engine/config.ts`
Carga `compensation_config` y `ranks` desde la base: ninguna constante está hardcodeada en el motor.
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

### 34. `lib/engine/period.ts`
Formato de período 'YYYY-MM' y período anterior.
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

### 35. `lib/engine/calculations.ts`
Funciones PURAS: bono directo, binario (15 %), rango (10 %), liquidación de pierna menor con carry-over, `qualifyRank` (las 4 condiciones), matching y estilo de vida.
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

### 36. `lib/engine/volume.ts`
Sube los puntos de una compra por el árbol binario hacia la raíz, sumando a la pierna que corresponde en cada ancestro.
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

### 37. `lib/engine/process-order.ts`
Procesa una orden pagada: puntos, bonos directos N1/N2 y acumulación de volumen.
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

### 38. `lib/engine/monthly-close.ts`
Cierre mensual: liquidación, rango, binario, rango, matching y estilo de vida para todos los usuarios.
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

### 39. `lib/engine/payout.ts`
Holding (validación de comisiones tras el periodo de retención) y pago semanal a la billetera.
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

# PARTE 10 — BASE DE DATOS
Esquema base, auditoría y tienda. Se pegan en el SQL Editor de Supabase en orden.

### 40. `db/migrations/001_schema.sql`
Esquema base: users (con las 3 llaves, rango, sponsor y placement), products, memberships, orders, volume_ledger, commissions, wallet, withdrawals, ranks, compensation_config.
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

### 41. `db/migrations/005_audit_log.sql`
Tabla `audit_log`: quién (admin), qué acción, sobre quién y detalle JSON. Solo el admin puede leerla (RLS).
```sql
-- ============================================================================
-- TRIADA — Migración 005: Registro de auditoría
-- Ejecutar DESPUÉS de 004_payout_schedule.sql. Pegar en: Dashboard > SQL Editor.
-- ============================================================================
--
-- Registra las acciones manuales del administrador (activar/desactivar miembros,
-- aprobar KYC, colocación manual, ajustes, etc.): quién, qué, sobre quién, cuándo.
-- ============================================================================

create table audit_log (
  id             uuid primary key default gen_random_uuid(),
  admin_id       uuid references users(id),
  action         text not null,              -- 'member_status', 'kyc_approve', 'manual_placement'...
  target_user_id uuid references users(id),
  detail         jsonb,                       -- datos del cambio (antes/después)
  created_at     timestamptz not null default now()
);

create index idx_audit_log_admin  on audit_log(admin_id, created_at);
create index idx_audit_log_target on audit_log(target_user_id);

alter table audit_log enable row level security;

-- Solo el admin puede leer la auditoría.
create policy "audit_log_admin_read" on audit_log
  for select to authenticated
  using (public.is_admin());

-- FIN MIGRACIÓN 005
```

### 42. `db/migrations/020_tienda_ecommerce.sql`
Tienda: campos de presentación en `products` (imagen, descripción, insignia…), `store_orders`, `store_coupons` y RLS.
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

### 43. `db/seeds/002_seed.sql`
Semilla de los 13 rangos con sus requisitos y la configuración del plan.
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
