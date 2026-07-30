# Carpeta `argocd/` — la capa de control de ArgoCD

Esta carpeta es lo que **ArgoCD lee para gestionar todo**. No contiene los manifiestos de tu
aplicación (esos están renderizados en la rama `deployment`) — contiene los objetos de ArgoCD
que dicen *qué* desplegar y *desde dónde*.

Está organizada como un **árbol de tres niveles** (igual que el `state-argocd` de Automya): un
app raíz arriba, dos apps que agrupan en medio, y las cosas de verdad abajo.

```
argocd/
├── root.Application.yaml                 ← 1) el "app raíz"
├── apps.Application.yaml                 ← 2) agrupa TODAS tus aplicaciones
├── services.Application.yaml             ← 2) agrupa TODOS los servicios del sistema
├── apps/
│   └── atm/
│       └── atm.ApplicationSet.yaml       ← 3) tus aplicaciones de negocio
└── sys-services/
    └── external-secrets/
        ├── external-secrets.Application.yaml   ← 3) el operador de secretos
        └── clustersecretstore-fake.yaml        ← 3) el store de mentira
```

---

## La jerarquía (lo importante)

Antes el app raíz se recorría **toda** la carpeta y aplicaba cada fichero directamente, así que en
ArgoCD salía todo plano (el ApplicationSet, external-secrets, los ambientes… todos al mismo
nivel). No había forma de distinguir "mis apps" de "los servicios del clúster".

Ahora hay **dos apps que agrupan en medio**:

```
root
├── apps        → gestiona la carpeta apps/          → ApplicationSet → dev / pro / staging
└── services    → gestiona la carpeta sys-services/  → external-secrets + store fake
```

- El **raíz** ya no se mete en las subcarpetas: solo aplica `apps` y `services`.
- **`apps`** se encarga de todo lo que hay en `apps/`.
- **`services`** se encarga de todo lo que hay en `sys-services/`.

Así, en ArgoCD, si abres la app **`apps`** ves colgando de ella el ApplicationSet y los tres
ambientes; y si abres **`services`** ves el operador y el store. Cada cosa en su rama del árbol.

> **Detalle de la interfaz:** en la rejilla de "Applications" siguen apareciendo las fichas de
> `eks-demo-default-dev/pro/staging` (ArgoCD nunca esconde las Applications del listado). Lo que
> cambia es que ahora **cuelgan** de `apps`, no del raíz. La agrupación se ve en el árbol de cada
> app, no en que desaparezcan del listado.

---

## Qué hace cada fichero

### `root.Application.yaml` — el app raíz (nivel 1)
Una Application de ArgoCD que apunta a **esta carpeta** (`path: argocd`) en la rama `main`, pero
con **`recurse: false`**: solo lee los ficheros que están *directamente* aquí (no baja a las
subcarpetas). Por eso solo aplica `apps.Application.yaml` y `services.Application.yaml`.

Es la única cosa que aplicas a mano (`kubectl apply`). A partir de ahí, ArgoCD gestiona el resto
solo.

### `apps.Application.yaml` — agrupador de aplicaciones (nivel 2)
Una Application llamada **`apps`** que apunta a la carpeta `argocd/apps` con **`recurse: true`**:
se recorre esa carpeta y aplica lo que haya dentro. Hoy solo está el ApplicationSet de `atm`; si
mañana añades otra aplicación en `apps/`, esta la recoge sola.

### `services.Application.yaml` — agrupador de servicios (nivel 2)
Igual que la anterior, pero llamada **`services`** y apuntando a `argocd/sys-services`. Gestiona
todo lo que sea "infraestructura del clúster": hoy el operador de external-secrets y el store; en
un sistema real irían aquí grafana, apisix, etc.

### `apps/atm/atm.ApplicationSet.yaml` — la app `atm` (nivel 3)
Un `ApplicationSet` que:
- mira la rama **`deployment`** del repo `app-atm`,
- busca las carpetas de ambiente (`kubernetes/*/*/*` → staging, pro, dev),
- y genera **una Application de ArgoCD por cada ambiente**, sola.

Cada Application generada sincroniza los manifiestos de su ambiente y los aplica al clúster (los
pods de tus microservicios). Añadir un ambiente = crear su carpeta; el `ApplicationSet` lo
detecta solo.

### `sys-services/external-secrets/external-secrets.Application.yaml` — el operador (nivel 3)
Una Application que instala el **operador de External Secrets** desde su chart de Helm. Antes lo
instalabas a mano (`helm install`); ahora lo gestiona ArgoCD desde git. Es el programa que da vida
a los recursos `ExternalSecret` (va al store, coge el valor, crea el Secret de Kubernetes).

### `sys-services/external-secrets/clustersecretstore-fake.yaml` — el store de mentira (nivel 3)
El `ClusterSecretStore` de tipo `fake`: un backend inventado que devuelve valores estáticos, para
no conectar a un AWS real. Es **solo para la práctica local**. Vive dentro de `sys-services/` para
que lo gestione la app `services` (antes estaba suelto en la raíz). En producción sería un store
real (AWS, Vault) sin valores dentro, solo datos de conexión.

---

## Cómo encaja todo (el flujo)

```
kubectl apply root.Application.yaml   (una sola vez, a mano)
        ↓
ArgoCD crea el app raíz → lee SOLO los ficheros de argocd/ (recurse: false)
        ↓
   ┌──────────────────────────┬──────────────────────────────┐
   ▼                          ▼
 apps                       services
 (lee argocd/apps/)         (lee argocd/sys-services/)
   ↓                          ↓
 atm ApplicationSet         external-secrets Application  +  store fake
   ↓                          ↓
 genera 3 Applications      instala el operador
 (staging, pro, dev)
   ↓
 cada una aplica sus
 manifiestos → pods corriendo
```

Todo cuelga del app raíz, en tres niveles limpios. Tú solo escribes config en git; ArgoCD hace el
resto.

---

## Dos ramas, dos cosas distintas (recordatorio)

- **`main`** (esta carpeta `argocd/` + tu config) → lo que escribes tú.
- **`deployment`** → los manifiestos renderizados que el `ApplicationSet` aplica.

El raíz, `apps` y `services` leen de `main` (la config de ArgoCD). El `ApplicationSet` lee de
`deployment` (los manifiestos). No los confundas: `argocd/` dice *cómo* desplegar; `deployment`
dice *qué*.

---

## En una frase

`argocd/` es el **panel de control**: un app raíz que aplica dos agrupadores —`apps` y
`services`— y cada uno pone bajo ArgoCD su carpeta, dejando un árbol limpio de tres niveles con
tus aplicaciones separadas de los servicios del sistema, todo desde git.
