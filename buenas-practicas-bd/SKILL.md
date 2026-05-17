---
name: buenas-practicas-bd
description: "Buenas prácticas obligatorias para cambios de base de datos, especialmente Supabase/Postgres: seguridad de funciones RPC, privilegios, RLS, migraciones y exposición pública. Usar antes de tocar esquema, funciones, políticas, triggers o migraciones de BD."
category: database
risk: safe
source: project
date_added: "2026-05-16"
---

# Buenas Prácticas BD

Usá esta skill antes de cualquier cambio relacionado con bases de datos: Supabase, Postgres, Prisma, migraciones, RLS, funciones SQL/RPC, triggers, policies, roles, permisos o seeds.

## Overview operativo en 11 pasos

1. Entender repo y stack real antes de programar.
2. Confirmar tablas, columnas, rutas, helpers y contratos existentes.
3. Clasificar datos: backend-only, frontend con RLS o público real.
4. Diseñar primero; no mezclar diseño, migración, backfill, índices, comportamiento y frontend.
5. Crear proposals/checks SELECT-only antes de tocar datos o schema.
6. Aplicar cambios DB con rollback, grants mínimos y verificación.
7. Backfillear por fases, con conteos, mismatches y demo/test controlados.
8. Diseñar índices desde queries reales antes de cambiar lecturas.
9. Migrar backend gradualmente con fallback legacy y diagnóstico seguro.
10. Recién después cambiar comportamiento/frontend si está aprobado y testeado.
11. Entregar evidencia: archivos, lecturas, tests, no-touch, riesgos, rollback y siguiente paso.

## Mini índice rápido

- **Prácticas 1–4**: funciones RPC, tablas/RLS, grants mínimos y migrations verificables.
- **Prácticas 5–8**: inventario RLS, tenant/auth, modelo base multiempresa y seed/backfill tenant.
- **Prácticas 9–13**: hardening backend-only, resolver tenant, diagnóstico y contrato `req.tenantContext`.
- **Prácticas 14–18**: validación BPI pasiva, `empresa_id nullable`, planning y backfill Fase A/B.
- **Prácticas 19–25**: backfill BPI agua por lotes, rollback y escalamiento seguro.
- **Prácticas 26–30**: revisión final, índices, diseño backend `empresa_id`, helper aislado y diagnóstico-only.
- **Prácticas 31–34**: protocolo IA, no inventar contratos, separar etapas y patrón para nuevas funciones.
- **Prácticas 35–37**: uploads/integraciones, plantilla de entrega y regla multiempresa.
- **Prácticas 38–39**: checklist Supabase y optimización Postgres basada en queries reales.
- **Guía rápida**: orden operativo para aplicar la skill sin leer todo el historial.
- **Caso completo**: ejemplo de camino legacy → multi-tenant operativo.

## Regla de uso obligatoria

Antes de modificar o crear objetos de base de datos:

1. Revisá si el cambio expone superficie pública por API/RPC.
2. Confirmá qué roles necesitan acceso real: `service_role`, `authenticated`, `anon`, usuarios internos o nadie vía API.
3. Aplicá privilegios explícitos en la migración.
4. Dejá evidencia de verificación: SQL aplicado/revisado, advisors si corresponde, y riesgos restantes.

## Práctica 1: no exponer funciones administrativas

No expongas funciones administrativas en `public` si no deben ser llamadas por usuarios.

En Supabase/Postgres, las funciones SQL ubicadas en esquemas expuestos pueden ser llamadas por API/RPC. Por eso, toda función debe clasificarse antes de publicarse:

- **Pública/intencional**: puede ser llamada por `anon` o `authenticated`.
- **De aplicación autenticada**: sólo `authenticated` con validaciones/RLS apropiadas.
- **Administrativa/interna**: no debe ser invocable por usuarios vía API/RPC.

### Reglas

- Preferí `SECURITY INVOKER` por defecto.
- Usá `SECURITY DEFINER` sólo cuando haya una razón clara y documentada.
- Si usás `SECURITY DEFINER`, fijá `search_path` de forma explícita para evitar resolución insegura de objetos.
- Si la función no debe ser pública, revocá `EXECUTE` a `PUBLIC`, `anon` y `authenticated`.
- Otorgá `EXECUTE` sólo al rol mínimo necesario.

### Plantilla segura para funciones internas

```sql
create or replace function public.internal_admin_task(...)
returns void
language plpgsql
security definer
set search_path = public, pg_temp
as $$
begin
  -- implementación
end;
$$;

revoke execute on function public.internal_admin_task(...) from public;
revoke execute on function public.internal_admin_task(...) from anon;
revoke execute on function public.internal_admin_task(...) from authenticated;
-- grant execute on function public.internal_admin_task(...) to service_role; -- sólo si corresponde
```

### Checklist de revisión

- [ ] ¿La función está en un esquema expuesto por Supabase API?
- [ ] ¿Debe existir en `public` o conviene moverla a un esquema interno/no expuesto?
- [ ] ¿Usa `SECURITY INVOKER` salvo necesidad justificada?
- [ ] Si usa `SECURITY DEFINER`, ¿tiene `set search_path = ...`?
- [ ] ¿Se revocó `EXECUTE` a `PUBLIC`, `anon` y `authenticated` si no es pública?
- [ ] ¿Los grants finales reflejan el menor privilegio posible?
- [ ] ¿La migración documenta por qué la función necesita esos permisos?

## Práctica 2: clasificar tablas antes de crear RLS

En Supabase, las tablas en schemas expuestos como `public` deben tener RLS activo. Las policies funcionan como un `WHERE` automático sobre cada consulta, pero no reemplazan el diseño de acceso ni el filtrado explícito desde la aplicación.

Supabase recomienda ser explícito con autenticación: `auth.uid()` devuelve `null` cuando no hay usuario autenticado. Una policy como `auth.uid() = user_id` falla silenciosamente para usuarios anónimos porque `null = user_id` nunca es verdadero. Si la tabla requiere usuario autenticado, dejalo expresado.

### Clasificación obligatoria antes de policies

No crees policies a ciegas. Antes de escribir o cambiar RLS, clasificá cada tabla:

1. **Tabla pública real**: datos intencionalmente visibles por `anon` o público.
2. **Tabla sólo backend/service_role**: no debe accederse desde clientes; preferir sin grants a roles API y acceso vía backend confiable.
3. **Tabla multiempresa**: requiere aislamiento por empresa/tenant y columna/index de tenant claro.
4. **Tabla por usuario**: requiere aislamiento por `auth.uid()`/usuario propietario.
5. **Tabla interna/auditoría**: no debe ser pública; lectura/escritura restringida a roles internos o service role.
6. **Tabla legacy/demo**: confirmar si se conserva, migra, bloquea o elimina en una etapa explícita.

### Reglas

- Activá RLS en tablas de schemas expuestos como `public` salvo decisión documentada y compensada.
- Especificá roles con `TO authenticated`, `TO anon`, etc.; no dejes policies implícitamente aplicadas a todos si no corresponde.
- Para acceso autenticado, usá checks explícitos como:

```sql
using ((select auth.uid()) is not null and (select auth.uid()) = user_id)
```

- Para tablas multiempresa, definí primero la fuente confiable del tenant (`empresa_id`, claim validado, tabla de membresía, etc.). No mezcles criterios legacy sin documentarlo.
- No dependas sólo del filtro implícito de RLS para rendimiento: agregá filtros explícitos en queries/backend/frontend cuando conozcas `user_id`, `empresa_id` u otra clave.
- Agregá índices en columnas usadas por RLS que no sean ya PK/unique, por ejemplo `user_id`, `empresa_id`, `tenant_id`, `owner_id` o FKs de membresía.
- En policies que llaman funciones estables por request, preferí envolverlas con `select` cuando no dependen de la fila, por ejemplo `(select auth.uid())`, para permitir mejor planificación.
- Evitá joins pesados dentro de policies; si son necesarios, medí con `EXPLAIN` y considerá helpers seguros fuera de schemas expuestos.

### Plantilla de clasificación

```md
| Tabla          | Clasificación | Roles API permitidos | Clave RLS  | Índices requeridos | Notas                       |
| -------------- | ------------- | -------------------- | ---------- | ------------------ | --------------------------- |
| public.ejemplo | multiempresa  | authenticated        | empresa_id | empresa_id         | pendiente validar membresía |
```

### Checklist de revisión

- [ ] ¿La tabla está en un schema expuesto por Supabase API?
- [ ] ¿RLS está activo si la tabla está en `public` u otro schema expuesto?
- [ ] ¿La tabla fue clasificada antes de crear policies?
- [ ] ¿La policy especifica roles con `TO ...`?
- [ ] ¿El acceso autenticado valida explícitamente que `auth.uid()` no sea `null`?
- [ ] ¿Las columnas usadas por RLS tienen índices adecuados?
- [ ] ¿Las queries agregan filtros explícitos además de confiar en RLS?
- [ ] ¿Se evitó crear policies genéricas tipo `using (true)` salvo tabla pública real documentada?
- [ ] ¿Se midió o dejó pendiente medir rendimiento si la policy usa joins o funciones?

## Práctica 3: decidir identidad antes de empresa_id y policies

Antes de escribir policies RLS multiempresa o crear `empresa_id`, decidí primero:

- quién es el usuario real;
- a qué empresa pertenece;
- qué tabla será la fuente oficial de membresía;
- qué funciones son públicas, internas o sólo backend/service role.

En este proyecto todavía conviven `public.users` legacy, `empresa` como texto y la posibilidad futura de usar `auth.users`. Por eso no conviene crear policies hasta decidir si la identidad primaria será `public.users`, `auth.users` o una tabla puente.

No agregues nuevas piezas BPI/RCM sin ordenar primero base, seguridad y multiempresa. BPI ya tiene avances reales con upload, tenant/empresa, backend y persistencia; antes de seguir agregando máquinas, rutas, costos o repuestos, hay que ordenar la base multiempresa para no mezclar datos ni crear un diseño difícil de escalar.

### Reglas

- No crees `empresa_id` ni policies hasta definir cómo se identifica al usuario y cómo se valida su empresa/tenant.
- No confíes en la `empresa` enviada desde el frontend. La empresa debe salir de una relación segura usuario ↔ empresa, por ejemplo una futura tabla `empresa_usuarios`.
- Tratá `empresa` text legacy como compatibilidad temporal, no como identidad definitiva del tenant.
- En schemas expuestos como `public`, mantené RLS activo; recordá que cada policy se comporta como un filtro automático tipo `WHERE`.
- Toda policy autenticada debe validar sesión explícitamente porque `auth.uid()` devuelve `null` si no hay sesión.
- No uses metadata editable por el usuario para autorización. En Supabase, `raw_user_meta_data` puede ser modificada por el usuario.
- Si en el futuro se usa Supabase Auth, la autorización debe venir desde tablas controladas por backend/admin o claims seguros, no desde datos que el usuario pueda modificar.
- Para autorización basada en claims, preferí `raw_app_meta_data` o tablas propias de membresía/roles; no uses `raw_user_meta_data`.
- Si existe `public.users` legacy, no asumas que `public.users.id` equivale a `auth.uid()` sin una relación documentada y verificable.
- Si se usa una tabla puente, debe definir unicidad, índices y reglas claras para usuario ↔ empresa ↔ rol.
- Separá credenciales/permisos internos de datos de perfil frontend-readable.
- No dejes funciones sensibles como RPC públicas por accidente. Antes de crear más RLS o helpers, clasificá cada función como pública intencional, interna, trigger/helper o backend/service_role.

### Decisiones obligatorias antes de implementar

Documentá estas decisiones antes de cualquier migration activa:

1. ¿La identidad primaria será `auth.users`, `public.users` legacy o puente?
2. ¿Cuál será la fuente oficial de membresía usuario ↔ empresa?
3. ¿Cómo se mapeará un usuario autenticado a una empresa sin confiar en el frontend?
4. ¿Qué rol/claim define autorización y quién puede modificarlo?
5. ¿`empresa` text legacy seguirá como compatibilidad o se migrará a `empresa_id`?
6. ¿Qué tablas necesitan acceso directo por Supabase API y cuáles serán backend-only?
7. ¿Qué funciones son RPC públicas intencionales, internas o sólo backend/service_role?
8. ¿Cómo se evitará romper BPI/RCM existentes durante la transición?

### Plantilla de decisión

```md
| Decisión            | Opción elegida | Motivo                                     | Riesgo                | Mitigación                              |
| ------------------- | -------------- | ------------------------------------------ | --------------------- | --------------------------------------- |
| Identidad primaria  | pendiente      | comparar auth.users vs public.users        | ruptura de login/RLS  | transición por tabla puente             |
| Fuente de membresía | pendiente      | no confiar en empresa enviada por frontend | mezcla de tenants     | empresa_usuarios controlada por backend |
| Tenant canónico     | pendiente      | migrar empresa text a empresa_id           | datos legacy mixtos   | mapeo incremental                       |
| Funciones RPC       | pendiente      | separar públicas vs internas               | exposición accidental | revocar/mover helpers internos          |
```

### Checklist de revisión

- [ ] ¿Está definida la identidad canónica antes de tocar `empresa_id`?
- [ ] ¿Está definida la fuente oficial de membresía usuario ↔ empresa?
- [ ] ¿Se evita confiar en `empresa` enviada desde frontend?
- [ ] ¿`empresa` text legacy queda tratada como compatibilidad temporal?
- [ ] ¿Está definido el vínculo usuario ↔ empresa ↔ rol?
- [ ] ¿Se evitó usar `raw_user_meta_data` para autorización?
- [ ] ¿Los claims usados para autorización vienen de `raw_app_meta_data` o de tablas controladas por backend/admin?
- [ ] ¿Las policies futuras validarán `auth.uid() is not null` explícitamente?
- [ ] ¿Se preserva BPI/RCM sin agregar nuevas piezas antes de ordenar base, seguridad y multiempresa?
- [ ] ¿Las funciones públicas fueron clasificadas antes de crear nuevos helpers/RLS?
- [ ] ¿Hay plan de transición/rollback antes de cualquier migration activa?

## Práctica 4: crear base tenant mínima antes de BPI/RCM

Ahora que el modelo híbrido temporal quedó aprobado como dirección, el siguiente paso no es tocar BPI ni RCM. Primero hay que crear una base tenant mínima, no destructiva y compatible con el login actual:

- `empresas`;
- `empresa_usuarios`;
- `user_auth_links` o puente equivalente.

La idea es preparar multiempresa real sin romper `public.users`, sin cambiar login y sin mezclar datos legacy.

### Reglas

- No agregues nuevas piezas BPI/RCM antes de crear y validar la base tenant mínima.
- La migración debe ser pequeña, reversible y verificable.
- Usá migraciones versionadas en `supabase/migrations` para capturar cambios SQL antes de aplicarlos al proyecto remoto.
- No crees policies RLS todavía si la etapa sólo prepara modelo base.
- Las tablas nuevas deben nacer pensando en RLS aunque las policies vengan después: llaves estables, timestamps, índices mínimos, constraints y fuente oficial de membresía.
- En schemas expuestos como `public`, RLS debe estar activo; recordá que las policies se comportan como filtros automáticos sobre cada consulta.
- No expongas `service_role`, claves secretas ni secretos en frontend, documentación o logs.
- Las claves secretas/service role pueden saltarse RLS y deben usarse sólo desde backend confiable.
- `empresa_usuarios` debe ser la fuente oficial de membresía cuando el modelo empiece a usarse; no el valor `empresa` enviado desde frontend.
- `user_auth_links` debe permitir transición futura a `auth.users` sin forzar cambio inmediato de login.

### Migración mínima esperada

La primera migración activa de esta etapa debería limitarse a preparar estructura base, sin backfill masivo ni cambios funcionales:

```sql
-- Ejemplo conceptual, no copiar sin revisar firmas/constraints finales.
create table if not exists public.empresas (...);
create table if not exists public.empresa_usuarios (...);
create table if not exists public.user_auth_links (...);

alter table public.empresas enable row level security;
alter table public.empresa_usuarios enable row level security;
alter table public.user_auth_links enable row level security;
```

### Checklist de revisión

- [ ] ¿La migración vive en `supabase/migrations` con nombre versionado?
- [ ] ¿Es pequeña y no destructiva?
- [ ] ¿Tiene rollback manual claro?
- [ ] ¿Incluye checks SQL SELECT-only para verificar estructura y datos base?
- [ ] ¿No toca BPI, RCM, frontend, backend, Prisma schema ni login?
- [ ] ¿No crea policies todavía salvo aprobación explícita?
- [ ] ¿Las tablas nuevas tienen PK, timestamps, índices mínimos y constraints?
- [ ] ¿RLS queda activo en tablas nuevas de `public`?
- [ ] ¿No aparece `service_role` ni secretos en frontend/docs/logs?
- [ ] ¿La membresía oficial queda modelada en tabla controlada por backend/admin?

## Práctica 5: revisar/aplicar migraciones Supabase por etapas

Antes de aplicar una migración en Supabase, primero confirmá qué cambia, qué no cambia, cómo se verifica y cómo se revierte. Si la migración ya está preparada pero todavía no fue aplicada, el primer objetivo es revisar/aplicar sólo esa base y ejecutar checks, no avanzar de golpe a BPI, RCM, policies o backfill.

En migraciones tenant, la estructura debe prepararse de forma aditiva y no destructiva: crear tablas nuevas como `empresas`, `empresa_usuarios` y `user_auth_links`, pero sin borrar `empresa` text legacy, sin modificar `public.users`, sin cambiar login, sin agregar `empresa_id` a BPI/RCM y sin hacer backfill masivo todavía.

### Reglas

- Antes de aplicar en Supabase, documentá:
  - qué objetos crea/modifica;
  - qué objetos explícitamente no toca;
  - qué checks se correrán después;
  - cómo se revierte si algo sale mal.
- Una migración tenant base debe ser aditiva y no destructiva.
- Separá creación de estructura de seed/backfill: primero estructura, después checks, después decisión explícita sobre seed controlado.
- No borres ni reemplaces `empresa` text legacy durante la etapa base tenant.
- No modifiques `public.users` ni cambies login/auth en una migración cuyo objetivo es sólo crear estructura tenant.
- No agregues `empresa_id` a BPI/RCM hasta que la base tenant y las decisiones de autorización estén validadas.
- Si las tablas nuevas quedan en `public`, deben nacer con RLS enabled.
- No crees policies reales si todavía no está definida la autorización final ni la membresía oficial validada.
- No expongas `service_role`: ninguna migración, check, doc, log ni frontend debe contener claves secretas.
- `service_role` debe seguir siendo sólo backend/servidor confiable; nunca frontend ni documentación pública.
- Para seed/backfill, tratá valores especiales como decisión separada y explícita: `demo_public`, `user_basico`, `integra`, `idubox`, `default` y valores null/vacíos.

### Flujo recomendado

1. Preparar migración aditiva en `supabase/migrations`.
2. Confirmar que el timestamp/versionado sea posterior a migraciones ya aplicadas.
3. Revisar SQL: sin destrucción, sin backfill, sin tocar BPI/RCM/app/login.
4. Preparar checks SELECT-only.
5. Documentar apply, verificación, rollback y decisiones pendientes.
6. Aplicar en Supabase sólo con aprobación explícita.
7. Ejecutar checks.
8. Recién después decidir si corresponde seed/backfill controlado.

### Checklist de revisión

- [ ] ¿La migración tiene timestamp posterior a las ya aplicadas?
- [ ] ¿Está claro qué cambia y qué no cambia?
- [ ] ¿Existe verificación SELECT-only?
- [ ] ¿Existe rollback documentado?
- [ ] ¿La migración es aditiva y no destructiva?
- [ ] ¿No borra `empresa` text legacy?
- [ ] ¿No modifica `public.users`?
- [ ] ¿No cambia login/auth?
- [ ] ¿No agrega `empresa_id` a BPI/RCM?
- [ ] ¿No hace backfill masivo ni seed productivo en la misma etapa?
- [ ] ¿Las tablas nuevas en `public` tienen RLS enabled?
- [ ] ¿No se crean policies reales antes de definir autorización final?
- [ ] ¿No aparece `service_role` ni secretos en frontend, docs o logs?
- [ ] ¿Los tenants especiales quedan para decisión/seed separado?

## Práctica 6: clasificar tenants antes de seed/backfill

Cuando la base tenant ya existe, el siguiente paso no es tocar BPI ni RCM. Primero hay que clasificar tenants especiales y revisar candidatos de seed/backfill. Las tablas `empresas`, `empresa_usuarios` y `user_auth_links` pueden estar creadas y vacías; eso es correcto hasta decidir qué significan realmente valores legacy como `demo_public`, `user_basico`, `integra`, `idubox`, `default` y valores null/vacíos.

No hagas seed/backfill automático desde `public.users.empresa`. Aunque parezca simple convertir cada valor de `empresa` en una fila de `empresas`, eso puede mezclar clientes reales, usuarios demo, soporte interno o datos legacy. La etapa correcta es ejecutar diagnósticos SELECT-only, resumir candidatos y pedir aprobación explícita por cada grupo antes de insertar datos.

### Reglas

- No tocar BPI ni RCM mientras la tarea sea clasificación tenant/seed base.
- No hacer seed/backfill automático desde `public.users.empresa`.
- Separar siempre:
  1. clasificación de tenants;
  2. propuesta de seed/backfill;
  3. ejecución del seed/backfill.
- Ejecutar primero checks SELECT-only para candidatos, colisiones, tenants especiales, usuarios sin empresa y usuarios admin/soporte/globales.
- Pedir aprobación explícita para cada grupo antes de insertar filas.
- En Prompt 6 se puede preparar una propuesta o migración de seed/backfill, pero no aplicarla sin aprobación posterior.
- El seed/backfill de esta etapa sólo puede apuntar a tablas tenant base: `empresas`, `empresa_usuarios`, `user_auth_links`.
- No tocar frontend, backend, Prisma, login, policies ni `empresa_id` en tablas existentes durante clasificación/seed tenant base.
- No modificar `public.users`; sólo leerlo como fuente legacy para diagnosticar candidatos.
- Cada seed debe tener batch metadata para poder revertir sólo las filas creadas por esa etapa.
- El rollback del seed debe borrar primero membresías, luego vínculos auth y finalmente empresas sólo si no tienen dependencias.

### Tenants especiales a clasificar

| Valor legacy  | Riesgo                                     | Regla inicial                                    |
| ------------- | ------------------------------------------ | ------------------------------------------------ |
| `demo_public` | puede ser demo/fallback, no cliente real   | no mapear como cliente sin aprobación            |
| `user_basico` | puede ser marcador de usuario público      | no mapear como cliente normal                    |
| `integra`     | posible cliente real con variantes de slug | revisar variantes antes de seed                  |
| `idubox`      | posible interno/soporte o cliente          | decidir si es soporte global o tenant normal     |
| `default`     | placeholder legacy ambiguo                 | revisión manual; no auto-map                     |
| null/vacío    | usuarios sin tenant claro                  | no crear `sin_empresa` automático sin aprobación |

### Batch metadata recomendada

Toda migración de seed propuesta debe marcar filas con metadata suficiente para rollback dirigido, por ejemplo:

```sql
-- Ejemplo conceptual: no aplicar sin revisar estructura final.
jsonb_build_object(
  'seed_batch', 'tenant_seed_YYYYMMDDHHMM',
  'source', 'public.users.empresa',
  'stage', 'prompt_6'
)
```

La metadata debe permitir responder:

- qué etapa creó la fila;
- desde qué fuente legacy vino;
- qué decisión aprobó el mapeo;
- cómo borrar sólo esas filas si se revierte el seed.

### Checklist de revisión

- [ ] ¿Se ejecutaron diagnósticos SELECT-only antes de proponer seed?
- [ ] ¿Se revisaron colisiones de slug?
- [ ] ¿Se clasificaron `demo_public`, `user_basico`, `integra`, `idubox`, `default` y null/vacío?
- [ ] ¿Se identificaron usuarios admin/soporte/globales antes de crear membresías?
- [ ] ¿El seed propuesto toca sólo `empresas`, `empresa_usuarios` y `user_auth_links`?
- [ ] ¿No toca BPI, RCM, frontend, backend, Prisma, login, policies ni `empresa_id` en tablas existentes?
- [ ] ¿No modifica `public.users`?
- [ ] ¿Cada grupo de filas tiene aprobación explícita?
- [ ] ¿Cada fila propuesta incluye batch metadata?
- [ ] ¿Existe rollback específico por batch?
- [ ] ¿Prompt 6 queda como propuesta hasta aprobación explícita posterior?

## Práctica 7: mantener tenant base backend-only antes de policies

Después de crear y poblar tablas tenant base, no hay que abrirlas inmediatamente al frontend. Si el login actual sigue usando `public.users` legacy y todavía no existe una relación runtime estable con `auth.users`, no conviene crear policies complejas con `auth.uid()`. Primero dejá las tablas tenant base como backend-only.

RLS activo con cero policies bloquea acceso efectivo desde `anon` y `authenticated`, pero si existen grants heredados amplios conviene limpiarlos en una etapa pequeña y controlada. Esto reduce el riesgo futuro de que una policy mal creada exponga datos de empresas o membresías.

### Reglas

- No crear policies reales para tenant base hasta definir si habrá lectura directa desde Supabase Client.
- Mantener como backend-only estas tablas por defecto:
  - `public.empresas`;
  - `public.empresa_usuarios`;
  - `public.user_auth_links`;
  - `public.audit_empresa_membresias`.
- El backend sigue siendo dueño operativo de estas tablas usando credenciales server-side; el frontend debe seguir pasando por rutas backend.
- No mezclar limpieza de grants/RLS con BPI, RCM, frontend, login, Prisma ni cambios de dominio.
- Si se limpian grants heredados, hacerlo en una migración pequeña, revisable y con rollback explícito.
- No revocar permisos globales o de schemas completos en esta etapa; tocar sólo las tablas tenant base aprobadas.
- No exponer `service_role` ni secretos en docs, logs o frontend.
- No usar `raw_user_meta_data` para autorización.
- No asumir que activar una policy después será seguro si los grants siguen amplios; diseñar grants y policies juntos, por etapas.

### Migración esperada para limpieza controlada

Una etapa futura de hardening puede limitarse a revocar acceso directo de API roles sobre tablas tenant base, sin tocar datos ni policies:

```sql
-- Ejemplo conceptual: revisar roles reales antes de aplicar.
revoke all on table public.empresas from anon, authenticated;
revoke all on table public.empresa_usuarios from anon, authenticated;
revoke all on table public.user_auth_links from anon, authenticated;
revoke all on table public.audit_empresa_membresias from anon, authenticated;
```

No mezclar con:

- creación de policies;
- seed/backfill;
- BPI/RCM;
- frontend/backend routes;
- Prisma;
- login/auth.

### Checklist de revisión

- [ ] ¿Las tablas tenant base deben seguir backend-only por ahora?
- [ ] ¿RLS está activo en las tablas tenant base?
- [ ] ¿Hay cero policies reales si no hay modelo de lectura directa aprobado?
- [ ] ¿Se revisaron grants directos a `anon` y `authenticated`?
- [ ] ¿La migración de grants toca sólo las cuatro tablas tenant base?
- [ ] ¿No toca BPI, RCM, frontend, backend routes, Prisma ni login?
- [ ] ¿No modifica datos ni `public.users`?
- [ ] ¿No crea policies reales todavía?
- [ ] ¿Tiene rollback explícito para restaurar grants si fuera necesario?
- [ ] ¿No expone `service_role` ni secretos?

## Práctica 8: diseñar tenantResolver backend antes de integrar dominios

Cuando `empresas`, `empresa_usuarios` y `user_auth_links` ya existen, tienen seed controlado y fueron protegidas como backend-only, el siguiente paso no es crear policies ni tocar BPI/RCM. Primero hay que diseñar cómo el backend resolverá el tenant oficial del usuario usando `public.users` legacy y las nuevas tablas tenant.

El frontend todavía no debe leer directamente `empresas` ni `empresa_usuarios`. El acceso debe seguir pasando por backend porque el login actual sigue basado en `public.users`, `auth_user_id` aún está null, y todavía no hay policies reales para `authenticated`.

### Reglas

- No reemplazar de golpe `req.user.empresa`.
- Durante la transición, el backend debe poder resolver:
  1. usuario legacy;
  2. membresías en `empresa_usuarios`;
  3. empresa actual seleccionada;
  4. fallback temporal a `public.users.empresa`.
- Diseñar `tenantResolver` o servicio equivalente antes de modificar rutas.
- No tocar BPI ni RCM en la etapa de diseño del resolver.
- No agregar `empresa_id` a tablas existentes mientras sólo se diseña backend tenant.
- No crear RLS policies todavía.
- No cambiar login/auth todavía.
- No exponer tablas tenant al frontend.
- Mantener `public.users.empresa` como compatibilidad temporal hasta que todos los dominios migren.
- `idubox` como soporte interno/global no debe convertirse en miembro de todas las empresas cliente.
- El acceso global de soporte debe diseñarse como regla controlada en backend, no como membresías duplicadas en cada cliente.

### Salida esperada de diseño

Una etapa de diseño debe producir, sin implementar rutas todavía:

- contrato de entrada de `tenantResolver`;
- contrato de salida;
- cómo elegir empresa actual si el usuario tiene varias membresías;
- cómo usar fallback `public.users.empresa`;
- cómo representar soporte/global;
- errores esperados;
- pruebas unitarias/integración futuras;
- plan de integración gradual con rutas existentes;
- riesgos y rollback.

### Contrato conceptual

```ts
// Ejemplo conceptual, no implementar sin aprobación.
type TenantContext = {
  legacyUserId: number;
  legacyEmpresa: string | null;
  currentEmpresaId: string | null;
  currentEmpresaSlug: string | null;
  memberships: Array<{
    empresaId: string;
    slug: string;
    rol:
      | "owner"
      | "admin_empresa"
      | "editor"
      | "viewer"
      | "soporte"
      | "operador";
    estado: string;
  }>;
  isSupport: boolean;
  source: "empresa_usuarios" | "legacy_empresa_fallback";
};
```

### Checklist de revisión

- [ ] ¿La etapa es diseño, no modificación de rutas?
- [ ] ¿No toca BPI ni RCM?
- [ ] ¿No cambia login/auth?
- [ ] ¿No expone `empresas`/`empresa_usuarios` al frontend?
- [ ] ¿No crea policies RLS?
- [ ] ¿Mantiene `public.users.empresa` como fallback temporal?
- [ ] ¿Define cómo resolver usuario → membresías → empresa actual?
- [ ] ¿Define soporte/global sin duplicar membresías en todos los clientes?
- [ ] ¿Define contratos de entrada/salida del resolver?
- [ ] ¿Define pruebas antes de implementar?
- [ ] ¿Explica cómo integrar después sin romper BPI, RCM ni login?

## Práctica 9: implementar tenantResolver mínimo sin cambiar comportamiento

Cuando la base tenant ya existe, tiene seed y está protegida como backend-only, el siguiente paso operativo no es cambiar el comportamiento de la app. Primero hay que implementar un resolver mínimo, aislado, probado y reversible.

El `tenantResolver` debe existir como módulo backend independiente antes de conectarlo masivamente a rutas productivas. Su primera versión debe poder ejecutarse con tests y/o diagnóstico interno, manteniendo intacto el comportamiento legacy.

### Reglas

- Implementar `tenantResolver` como módulo aislado, por ejemplo `utils/tenantResolver.js`, sólo con aprobación explícita.
- Derivar tenant desde datos controlados por servidor:
  1. `req.user` legacy;
  2. `public.users.id`;
  3. `empresa_usuarios`;
  4. `empresas`;
  5. `tenantContext`.
- No aceptar como autoridad una `empresa` enviada por frontend.
- Si en el futuro el frontend pide seleccionar empresa, backend debe validar que el usuario tenga membresía real en esa empresa.
- Mantener compatibilidad con `req.user.empresa`.
- Crear `req.tenantContext` como concepto/módulo paralelo, no reemplazar de golpe `req.user.empresa`.
- Recordar que BPI y RCM siguen dependiendo de `empresa` texto.
- No conectar el resolver masivamente a rutas productivas en la primera implementación.
- No mezclar implementación del resolver con migraciones.
- No crear policies RLS en esta etapa.
- No agregar `empresa_id` a tablas existentes.
- No tocar BPI.
- No tocar RCM.
- No cambiar login/auth.
- No cambiar frontend.
- No exponer tablas tenant al frontend directo.
- No usar `raw_user_meta_data` para autorización.

### Implementación mínima esperada

Una etapa de implementación mínima puede crear:

- módulo `tenantResolver` aislado;
- tests unitarios con fixtures;
- helper/repositorio de lectura backend;
- diagnóstico interno opcional sólo si se aprueba;
- documentación de rollback.

No debe crear todavía:

- cambios en `routes/`;
- cambios globales en middleware productivo;
- migrations;
- RLS policies;
- `empresa_id`;
- cambios BPI/RCM;
- cambios frontend/login.

### Checklist de revisión

- [ ] ¿El resolver se implementó como módulo aislado?
- [ ] ¿Deriva tenant desde `req.user`/`public.users.id` y membresías, no desde frontend?
- [ ] ¿Valida cualquier selector futuro contra membresías reales?
- [ ] ¿Mantiene `req.user.empresa` intacto?
- [ ] ¿No conecta masivamente `req.tenantContext` a rutas productivas todavía?
- [ ] ¿No toca BPI ni RCM?
- [ ] ¿No cambia login/auth ni frontend?
- [ ] ¿No crea migrations, policies ni `empresa_id`?
- [ ] ¿No modifica datos?
- [ ] ¿Tiene tests para integra, idubox, user_basico, fallback y multi-membership?
- [ ] ¿Tiene rollback simple: dejar de importar/usar el módulo y volver al flujo legacy?

### Relación con Prompt 10

`docs/security/prompt-10-proposed.md` define la implementación mínima backend-only del resolver. Ese prompt no debe migrar BPI/RCM, no debe crear RLS policies, no debe agregar `empresa_id` y no debe cambiar login.

## Práctica 10: integrar tenantResolver como middleware opcional post-auth

Cuando `tenantResolver` ya existe y está probado, no conviene conectarlo de golpe a todo el sistema. Primero debe integrarse como middleware opcional post-auth, es decir, después de que `middleware/auth.js` ya haya llenado `req.user`.

La integración inicial debe agregar contexto, no cambiar comportamiento legacy.

### Reglas

- Integrar el resolver después de auth, nunca antes de que exista `req.user`.
- No borrar ni reemplazar `req.user.empresa` todavía.
- Agregar un campo paralelo:

```js
req.tenantContext = tenantContext;
```

- Mantener intacto:

```js
req.user.empresa;
```

- Si no hay usuario autenticado, continuar sin fallar:

```js
if (!req.user) return next();
```

- Si hay usuario autenticado, resolver tenant context:

```js
req.tenantContext = await resolveTenantContext(...);
```

- No romper rutas públicas ni endpoints que no requieren usuario.
- No exponer `empresas`, `empresa_usuarios` ni `user_auth_links` al frontend.
- Usar acceso backend/service-side mediante dependencias inyectadas.
- No filtrar secretos, tokens, cookies ni `service_role`.
- No tocar BPI ni RCM en esta etapa.
- No cambiar frontend.
- No cambiar login/auth.
- No crear policies RLS.
- No crear migrations.
- No agregar `empresa_id` a tablas existentes.

### Alcance esperado

Una etapa de middleware opcional puede crear:

- wrapper/middleware post-auth;
- tests del middleware;
- feature flag o modo opt-in si hace falta;
- logs seguros para warnings/fallbacks;
- documentación de rollback.

No debe crear todavía:

- cambios funcionales en BPI;
- cambios funcionales en RCM;
- cambios de frontend;
- cambios de login;
- migrations;
- policies;
- cambios de grants;
- exposición directa de tablas tenant.

### Checklist de revisión

- [ ] ¿El middleware corre después de `middleware/auth.js`?
- [ ] ¿`sin req.user → next()` sin error?
- [ ] ¿`con req.user → req.tenantContext`?
- [ ] ¿`req.user.empresa` queda intacto?
- [ ] ¿No se conectó masivamente a comportamiento de dominio?
- [ ] ¿No cambió BPI?
- [ ] ¿No cambió RCM?
- [ ] ¿No cambió frontend/login?
- [ ] ¿No creó migrations, policies, grants ni `empresa_id`?
- [ ] ¿No expone tablas tenant al frontend?
- [ ] ¿No filtra secrets/service_role/tokens?
- [ ] ¿Tiene tests de request sin usuario, usuario con tenant, fallback y error controlado?
- [ ] ¿Tiene rollback simple: quitar middleware/feature flag y seguir con `req.user.empresa`?

### Relación con Prompt 11

`docs/security/prompt-11-proposed.md` define este alcance: integrar `tenantResolver` como middleware opcional post-auth, adjuntar `req.tenantContext`, mantener `req.user.empresa`, no cambiar BPI/RCM/frontend/login, no crear policies, no crear migrations y no agregar `empresa_id` a tablas existentes.

## Práctica 11: usar req.tenantContext sólo para diagnóstico antes de autorizar

Cuando `req.tenantContext` ya existe, no se debe usar todavía para cambiar permisos, filtros, escrituras ni lecturas de BPI/RCM. Primero hay que observar si el contexto tenant resuelve bien en producción controlada: empresa actual, fallback, warnings, errores y diferencias con `req.user.empresa`.

Diagnóstico no es autorización. Aunque `tenantContext` diga que un usuario pertenece a `integra`, todavía no debe reemplazar reglas actuales ni cambiar comportamiento de dominio.

### Reglas

- No usar `req.tenantContext` todavía para autorizar BPI/RCM.
- No usar `req.tenantContext` todavía para cambiar filtros de queries.
- No usar `req.tenantContext` todavía para cambiar escrituras o lecturas.
- No reemplazar `req.user.empresa`.
- Observar pasivamente:
  - `fallbackUsed`;
  - `legacy_empresa_mismatch`;
  - `TENANT_REQUIRED`;
  - `TENANT_FORBIDDEN`;
  - `MEMBERSHIP_INACTIVE`;
  - `TENANT_INACTIVE`.
- No exponer tablas tenant al frontend.
- Si se crea una vista/endpoint diagnóstico, debe ser interno/admin, sanitizado y sin secretos.
- No mostrar tokens, cookies, `service_role`, passwords, metadata sensible ni datos de otras empresas que no correspondan.
- BPI debe seguir usando `empresa` legacy por ahora.
- En BPI sólo se permite comparación pasiva:

```text
req.tenantContext.currentEmpresaSlug
vs
req.user.empresa
```

- No cambiar filtros BPI.
- No cambiar uploads BPI.
- No cambiar consultas BPI.
- No cambiar normalización BPI.
- No cambiar escrituras BPI.
- No tocar RCM.
- No cambiar frontend/login.
- No crear policies, migrations, grants ni `empresa_id`.
- No modificar datos.

### Señales diagnósticas esperadas

Una etapa diagnóstica debe medir/reportar de forma segura:

- cantidad de requests con `fallbackUsed = true`;
- cantidad de `legacy_empresa_mismatch`;
- usuarios sin tenant resoluble (`TENANT_REQUIRED`);
- selector no permitido (`TENANT_FORBIDDEN`), si existe selector futuro;
- membresías suspendidas (`MEMBERSHIP_INACTIVE`);
- tenants inactivos (`TENANT_INACTIVE`);
- comparación pasiva entre `currentEmpresaSlug` y `req.user.empresa`.

### Checklist de revisión

- [ ] ¿La etapa es diagnóstico, no autorización?
- [ ] ¿BPI sigue usando `empresa` legacy?
- [ ] ¿No cambiaron filtros/uploads/consultas/escrituras BPI?
- [ ] ¿No se tocó RCM?
- [ ] ¿No se reemplazó `req.user.empresa`?
- [ ] ¿No se usa `tenantContext` para permisos todavía?
- [ ] ¿Los logs/endpoint diagnóstico están sanitizados?
- [ ] ¿No se exponen tablas tenant al frontend?
- [ ] ¿No se filtran secrets/tokens/cookies/passwords/service_role?
- [ ] ¿No se crean policies, migrations, grants ni `empresa_id`?
- [ ] ¿No se modifican datos?

### Relación con Prompt 12

`docs/security/prompt-12-proposed.md` define esta etapa como diagnóstico solamente: observar `req.tenantContext`, warnings/fallbacks y preparar validación BPI sin cambiar BPI, RCM, frontend, login, policies, migrations ni datos.

## Práctica 12: validar BPI pasivamente con req.tenantContext

Cuando `req.tenantContext` ya existe, todavía no debe reemplazar a `req.user.empresa` en BPI. Esta etapa debe ser sólo observación pasiva: comparar, registrar warnings seguros y confirmar que no hay diferencias entre el tenant nuevo y el tenant legacy.

Diagnóstico no es migración. Aunque BPI ya pueda ver `req.tenantContext`, no debe cambiar todavía:

- queries;
- uploads;
- normalización;
- filtros;
- persistencia;
- empresa usada para escribir;
- empresa usada para leer.

### Reglas

- BPI sigue usando `req.user.empresa` y el flujo legacy actual.
- `req.tenantContext` sólo se puede usar para comparar y registrar diagnóstico.
- No cambiar queries BPI.
- No cambiar uploads BPI.
- No cambiar normalización BPI.
- No cambiar filtros BPI.
- No cambiar persistencia BPI.
- No cambiar la `empresa` usada para escribir.
- No cambiar la `empresa` usada para leer.
- No agregar `empresa_id`.
- No tocar RCM.
- No cambiar frontend/login.
- No crear policies, migrations ni grants.
- No modificar datos.

### Logs seguros

Los logs de diagnóstico no deben incluir:

- payloads completos;
- archivos;
- tokens;
- cookies;
- `service_role`;
- headers;
- passwords;
- datos crudos BPI;
- objetos completos de request.

Sólo pueden incluir datos mínimos:

- `legacyUserId`;
- `legacyEmpresa`;
- `currentEmpresaSlug`;
- `fallbackUsed`;
- `warnings`;
- `route`;
- `tipoDato`.

### Tests obligatorios

Si se agrega diagnóstico dentro de `routes/bpi.js` o `routes/bpiUpload.js`, los tests deben demostrar que BPI sigue igual:

- la empresa usada para filtrar sigue saliendo del flujo legacy actual;
- la empresa usada para escribir sigue saliendo del flujo legacy actual;
- `tenantContext` no cambia datos;
- uploads no cambian destino ni metadata;
- lecturas no cambian filtros;
- normalización no cambia.

### Checklist de revisión

- [ ] ¿La etapa es observación pasiva, no migración?
- [ ] ¿BPI sigue usando `req.user.empresa`?
- [ ] ¿`req.tenantContext` no cambia filtros/lecturas/escrituras/uploads?
- [ ] ¿Los logs son mínimos y sanitizados?
- [ ] ¿No se loguean payloads, archivos, headers, tokens, cookies, service_role, passwords ni crudos BPI?
- [ ] ¿Los tests prueban que la empresa legacy sigue siendo la usada?
- [ ] ¿No se tocó RCM?
- [ ] ¿No se tocó frontend/login?
- [ ] ¿No se crearon policies, migrations, grants ni `empresa_id`?
- [ ] ¿No se modificaron datos?

### Relación con Prompt 13

`docs/security/prompt-13-proposed.md` define este alcance: validación BPI pasiva con `req.tenantContext`, sin cambiar lecturas, escrituras, uploads, filtros ni normalización. Esta etapa sí puede tocar BPI, pero sólo para diagnóstico pasivo.

## Práctica 13: revisar diagnóstico BPI antes de planear empresa_id

Antes de agregar `empresa_id` a BPI, primero hay que revisar si el diagnóstico pasivo de la Etapa 13 fue limpio. No alcanza con que `req.tenantContext` exista: hay que confirmar que no aparezcan señales de mismatch, fallback o error de resolución.

Esta etapa debe ser planificación, no migración. Todavía no se debe crear `empresa_id`, no se debe hacer backfill y no se debe cambiar comportamiento BPI. El objetivo es diseñar qué tablas tocar después, qué índices serán necesarios, cómo validar los datos y cómo hacer rollback.

BPI debe mantenerse estable. Aunque el diagnóstico pasivo ya exista en `routes/bpi.js` y `routes/bpiUpload.js`, BPI sigue usando `req.user.empresa` / `getAuthenticatedTenant(req)` como fuente efectiva. La Etapa 14 no debe cambiar queries, uploads, filtros, normalización, persistencia ni respuestas.

RCM no debe tocarse todavía. BPI se debe migrar primero porque ya tiene tablas claras y columna `empresa`; RCM requiere un diseño separado por activos, mantenimiento, suministros, solicitudes y relaciones.

### Señales que bloquean avanzar a empresa_id

No avances a diseño de migración activa si aparece cualquiera de estas señales sin explicación y mitigación documentada:

- `bpi_tenant_mismatch`;
- `legacy_empresa_fallback_used`;
- `TENANT_REQUIRED`;
- `TENANT_CONTEXT_UNAVAILABLE`.

### Reglas

- Revisar evidencia real del diagnóstico pasivo antes de proponer `empresa_id`.
- Documentar conteos y ejemplos seguros de mismatch/fallback/error, sin payloads ni datos crudos BPI.
- Si hay señales bloqueantes, preparar corrección/diagnóstico adicional antes de migrar.
- Mantener la etapa como planificación: no crear migrations, no aplicar SQL, no modificar datos.
- No crear `empresa_id` todavía.
- No hacer backfill todavía.
- No cambiar BPI reads/writes/uploads/filtros/normalización/persistencia/respuestas.
- No tocar RCM.
- Diseñar tablas BPI futuras, índices, validaciones, checks y rollback antes de cualquier cambio activo.
- Separar explícitamente BPI de RCM en el plan.

### Plan mínimo esperado

La salida de esta etapa debe ser un plan, no código de migración aplicado:

1. Resumen del diagnóstico Etapa 13:
   - cantidad de eventos `bpi_tenant_mismatch`;
   - cantidad de `legacy_empresa_fallback_used`;
   - cantidad de `TENANT_REQUIRED`;
   - cantidad de `TENANT_CONTEXT_UNAVAILABLE`;
   - decisión go/no-go.
2. Inventario de tablas BPI candidatas para `empresa_id`.
3. Índices futuros necesarios para filtros por tenant.
4. Estrategia de validación previa y posterior.
5. Estrategia de rollback.
6. Lista explícita de no-cambios: BPI behavior, RCM, frontend, login, policies, grants, datos.

### Checklist de revisión

- [ ] ¿Se revisó evidencia de diagnóstico pasivo Etapa 13?
- [ ] ¿No hay `bpi_tenant_mismatch` sin explicar?
- [ ] ¿No hay `legacy_empresa_fallback_used` sin explicar?
- [ ] ¿No hay `TENANT_REQUIRED` o `TENANT_CONTEXT_UNAVAILABLE` sin explicar?
- [ ] ¿La etapa sigue siendo planificación y no migración?
- [ ] ¿No se creó `empresa_id`?
- [ ] ¿No se hizo backfill?
- [ ] ¿No se cambiaron queries/uploads/filtros/normalización/persistencia/respuestas BPI?
- [ ] ¿BPI sigue usando `req.user.empresa` / `getAuthenticatedTenant(req)` como fuente efectiva?
- [ ] ¿No se tocó RCM?
- [ ] ¿El plan separa BPI de RCM?
- [ ] ¿El plan incluye tablas, índices, validaciones y rollback futuro?

### Relación con Prompt 14

`docs/security/prompt-14-proposed.md` debe tratarse como planificación posterior al diagnóstico pasivo. No autoriza migraciones, backfill ni cambios de comportamiento. Cualquier cambio activo de `empresa_id` requiere una etapa separada y aprobación explícita.

## Práctica 14: resolver legacy BPI no mapeable antes de crear empresa_id

Antes de crear `empresa_id` en BPI, hay que resolver primero los valores legacy que no mapean a `public.empresas.slug`. Si se agrega `empresa_id` sin resolverlos, el backfill quedará parcial y después será más difícil distinguir datos reales, demo y test.

`empresa_id` debe entrar primero como nullable, nunca obligatorio en la primera etapa. BPI todavía usa `empresa` texto como fuente efectiva y la transición debe ser gradual:

1. `empresa` text sigue funcionando;
2. `empresa_id` se agrega como columna auxiliar nullable;
3. el backfill se valida;
4. recién después se evalúa usar `empresa_id` en queries/writes/policies.

No mezcles limpieza de datos con migración estructural. Primero se deben clasificar los valores legacy no mapeables y decidir si serán tenants demo/test, aliases de una empresa existente o datos a dejar sin mapear.

Valores legacy detectados en Etapa 14 que requieren decisión antes de backfill completo:

- `empresa_demo`;
- `test-empresa` / `test_empresa`;
- `empresa_maquinas_test`.

Tablas grandes como `bpi_agua_readings` requieren cuidado. Aunque `235.847` filas no es enorme para Postgres, igual conviene evitar un backfill masivo sin controles, sobre todo si después crecerá. La Etapa 14 ya dejó documentado que `bpi_agua_readings` es de alto volumen y requiere estrategia por lotes o revisión especial.

### Reglas

- No crear `empresa_id` hasta clasificar valores legacy no mapeables o documentar por qué quedarán `null`.
- No hacer backfill completo si existen empresas legacy no mapeables sin decisión.
- No convertir `empresa_id` en `not null` al inicio.
- No borrar ni dejar de escribir `empresa` text durante la transición.
- No cambiar queries/writes BPI para usar `empresa_id` en la etapa de columna auxiliar.
- Separar limpieza/clasificación de datos de migración estructural.
- Tratar demo/test explícitamente: tenant demo/test, alias aprobado o datos sin mapear documentados.
- En tablas grandes, planear backfill por lotes o con revisión especial antes de ejecutar.
- Validar conteos antes/después de cualquier backfill futuro.
- Mantener rollback posible mientras BPI siga usando `empresa` text.

### Checklist de revisión

- [ ] ¿Se listaron valores legacy no mapeables?
- [ ] ¿Cada valor no mapeable tiene decisión: tenant demo/test, alias o dejar sin mapear?
- [ ] ¿La migración estructural está separada de limpieza/backfill de datos?
- [ ] ¿`empresa_id` se mantiene nullable?
- [ ] ¿`empresa` text sigue funcionando y no se borra?
- [ ] ¿No se cambiaron queries/writes/uploads/filtros BPI?
- [ ] ¿`bpi_agua_readings` tiene estrategia por lotes o revisión especial?
- [ ] ¿Hay checks de conteos antes/después?
- [ ] ¿El rollback sigue siendo posible sin perder `empresa` text?

## Práctica 15: crear empresa_id nullable sin backfill ni cambio BPI

Una migración estructural no debe mezclarse con backfill. En esta etapa sólo se debe agregar la columna `empresa_id` como nullable, sin poblarla todavía. Eso reduce riesgo, permite rollback y mantiene BPI funcionando igual con `empresa` text legacy.

No cambies comportamiento BPI al mismo tiempo que cambiás estructura. Aunque se agregue `empresa_id`, BPI debe seguir usando:

```text
req.user.empresa / getAuthenticatedTenant(req)
```

No se deben cambiar queries, writes, uploads, filtros, normalización, persistencia ni respuestas.

No agregues índices pesados todavía si no son estrictamente necesarios. `bpi_agua_readings` tiene muchas filas, y crear índices grandes o hacer backfill puede generar locks o impacto operativo. En esta etapa conviene limitarse a columna nullable + FK, y dejar índices/backfill para etapas separadas.

Los valores demo/test deben quedar sin mapear por ahora. Ya se clasificaron:

- `empresa_demo`;
- `test-empresa` / `test_empresa`;
- `empresa_maquinas_test`.

La decisión vigente es no mapearlos a `integra`, `idubox` ni `user_basico`. Por eso `empresa_id` debe aceptar `null`. El Prompt 16 propuesto en el repo confirma este alcance: crear migración real no destructiva para `empresa_id nullable`, sin backfill, sin cambiar BPI, sin tocar RCM y sin policies.

### Reglas

- Separar migración estructural de backfill.
- Agregar `empresa_id` sólo como `uuid null` al inicio.
- No poblar `empresa_id` en la misma etapa.
- No convertir `empresa_id` en obligatorio.
- No borrar ni reemplazar `empresa` text.
- No cambiar `req.user.empresa` / `getAuthenticatedTenant(req)` como fuente efectiva BPI.
- No cambiar queries, writes, uploads, filtros, normalización, persistencia ni respuestas BPI.
- No crear índices pesados en tablas grandes salvo aprobación explícita.
- No hacer backfill ni update masivo en `bpi_agua_readings` en esta etapa.
- Dejar demo/test con `empresa_id null` hasta decisión posterior.
- No tocar RCM.
- No crear policies RLS todavía.

### Checklist de revisión

- [ ] ¿La migración sólo agrega columna nullable + FK?
- [ ] ¿No hay `UPDATE`/backfill en la migración?
- [ ] ¿No hay cambios en rutas/queries/writes/uploads BPI?
- [ ] ¿BPI sigue usando `req.user.empresa` / `getAuthenticatedTenant(req)`?
- [ ] ¿`empresa` text sigue intacta?
- [ ] ¿`empresa_id` acepta `null`?
- [ ] ¿Demo/test quedan sin mapear por ahora?
- [ ] ¿No se mapearon demo/test a `integra`, `idubox` ni `user_basico`?
- [ ] ¿No se agregaron índices pesados sin aprobación explícita?
- [ ] ¿No se tocó RCM/frontend/login/policies/grants/datos?

### Relación con Prompt 16

`docs/security/prompt-16-proposed.md` autoriza sólo una futura migración estructural no destructiva para `empresa_id nullable`, sin backfill, sin cambios BPI, sin RCM y sin policies. No autoriza poblar datos ni usar `empresa_id` en comportamiento BPI.

## Práctica 16: aplicar migración empresa_id nullable sólo con checks limpios

Una migración preparada no debe aplicarse hasta confirmar tres cosas:

1. el SQL no tiene `INSERT`, `UPDATE` ni `DELETE`;
2. el SQL no crea policies, grants ni revokes;
3. los checks posteriores pueden demostrar que sólo cambió la estructura.

Esta etapa debe aplicar sólo la columna `empresa_id` nullable, sin backfill. Después de aplicar, lo esperado es:

- `empresa_id` existe;
- `empresa_id` es `uuid`;
- `empresa_id` es nullable;
- la FK apunta a `public.empresas(id)`;
- `count(empresa_id) = 0`;
- `empresa` text sigue existiendo.

No cambies BPI al mismo tiempo que se aplica la estructura. BPI debe seguir usando:

```text
req.user.empresa / getAuthenticatedTenant(req)
```

No se deben cambiar lecturas, escrituras, uploads, filtros, normalización, persistencia ni respuestas.

Los datos demo/test deben seguir sin mapear. Como todavía no se decidió crear aliases o tenants demo/test, valores como `empresa_demo`, `test-empresa` y `empresa_maquinas_test` deben quedar con `empresa_id null`.

El Prompt 17 propuesto en el repo confirma este alcance: aplicar la migración `20260516200000_add_bpi_empresa_id_nullable.sql`, ejecutar checks, confirmar `count(empresa_id)=0`, no hacer backfill y no cambiar BPI/RCM/frontend/login/policies/grants/datos.

### Reglas

- Antes de aplicar, verificar que la migración no contiene DML (`INSERT`/`UPDATE`/`DELETE`).
- Antes de aplicar, verificar que la migración no crea policies, grants ni revokes.
- Aplicar sólo la migración estructural aprobada.
- No aplicar backfill en la misma etapa.
- Después de aplicar, ejecutar checks SELECT-only.
- Confirmar que `empresa_id` existe, es `uuid`, nullable y con FK correcta.
- Confirmar que `count(empresa_id) = 0` en todas las tablas BPI afectadas.
- Confirmar que `empresa` text sigue existiendo.
- Confirmar que demo/test siguen con `empresa_id null`.
- No cambiar BPI behavior.
- No tocar RCM/frontend/login/policies/grants/datos.

### Checklist de revisión

- [ ] ¿Se revisó que la migración no tenga `INSERT`/`UPDATE`/`DELETE`?
- [ ] ¿Se revisó que la migración no tenga policies/grants/revokes?
- [ ] ¿Se aplicó sólo la migración estructural aprobada?
- [ ] ¿Los checks post-apply son SELECT-only?
- [ ] ¿`empresa_id` existe en todas las tablas esperadas?
- [ ] ¿`empresa_id` es `uuid` y nullable?
- [ ] ¿Todas las FKs apuntan a `public.empresas(id)`?
- [ ] ¿`count(empresa_id) = 0` después de aplicar?
- [ ] ¿`empresa` text sigue existiendo?
- [ ] ¿Demo/test siguen sin mapear?
- [ ] ¿No cambió BPI behavior?
- [ ] ¿No se tocó RCM/frontend/login/policies/grants/datos?

### Relación con Prompt 17

`docs/security/prompt-17-proposed.md` autoriza sólo aplicar y verificar la migración estructural nullable. No autoriza backfill, aliases, tenants demo/test, cambios BPI, RCM, frontend, login, policies, grants ni datos.

## Práctica 17: diseñar backfill parcial antes de cualquier UPDATE

Aunque `empresa_id` ya existe en BPI, todavía está vacío. El siguiente paso no debe ser hacer `UPDATE` directo. Primero hay que diseñar el backfill parcial, separando filas mapeables de filas demo/test no mapeables.

El backfill debe ir por etapas. No conviene partir por `bpi_agua_readings` porque es la tabla más grande. Primero se deben planear tablas chicas y controladas:

- `bpi_energia_readings`;
- `bpi_produccion_records`;
- `bpi_ingestion_jobs`;
- `bpi_rutas_productivas`.

Luego tablas con datos demo/test:

- `bpi_upload_batches`;
- `bpi_maquinas`.

Y recién después:

- `bpi_agua_readings`.

El backfill no debe cambiar el comportamiento de BPI. Aunque se llene `empresa_id`, BPI debe seguir usando `empresa` text hasta que una etapa posterior cambie lecturas/escrituras de forma controlada.

Los valores demo/test deben quedar fuera del backfill parcial por ahora:

- `empresa_demo`;
- `test-empresa`;
- `empresa_maquinas_test`.

Eso evita mezclar datos de prueba con empresas reales. La Etapa 17 confirmó que `empresa_id` existe, está nullable, tiene FK, pero `count(empresa_id)=0` en todas las tablas, y no hubo backfill.

### Reglas

- No ejecutar `UPDATE` directo sin plan de backfill parcial aprobado.
- Separar filas mapeables de filas demo/test no mapeables.
- Excluir demo/test del backfill parcial inicial.
- No mapear demo/test a `integra`, `idubox` ni `user_basico`.
- Planear orden por riesgo: tablas chicas primero, tablas con demo/test después, `bpi_agua_readings` al final.
- Mantener `empresa` text como fuente efectiva BPI durante y después del backfill.
- No cambiar queries/writes/uploads/filtros/normalización/persistencia/respuestas BPI durante backfill.
- Validar conteos antes/después por tabla.
- Diseñar rollback por tabla antes de ejecutar.
- No tocar RCM/frontend/login/policies/grants.

### Checklist de revisión

- [ ] ¿Hay plan de backfill antes de cualquier `UPDATE`?
- [ ] ¿El plan separa mapeables vs demo/test no mapeables?
- [ ] ¿Demo/test quedan fuera del backfill parcial inicial?
- [ ] ¿El orden empieza por tablas chicas/controladas?
- [ ] ¿`bpi_agua_readings` queda para el final con estrategia especial?
- [ ] ¿BPI sigue usando `empresa` text como fuente efectiva?
- [ ] ¿No se cambiaron queries/writes/uploads/filtros/respuestas BPI?
- [ ] ¿Hay conteos pre/post por tabla?
- [ ] ¿Hay rollback por tabla?
- [ ] ¿No se tocó RCM/frontend/login/policies/grants?

### Relación con Prompt 18

`docs/security/prompt-18-proposed.md` debe tratarse como diseño de backfill parcial, no ejecución. No autoriza `UPDATE`, backfill real, índices, aliases, tenants demo/test, cambios BPI, RCM, frontend, login, policies ni grants.

## Práctica 18: ejecutar primero backfill Fase A chico y reversible

El primer backfill debe ser pequeño, reversible y medible. No conviene partir por `bpi_agua_readings` porque tiene alto volumen. La Fase A debe usar sólo tablas chicas y 100% mapeables:

- `bpi_energia_readings`;
- `bpi_produccion_records`;
- `bpi_ingestion_jobs`;
- `bpi_rutas_productivas`.

Backfill no debe cambiar comportamiento. Aunque se llene `empresa_id`, BPI debe seguir usando:

```text
req.user.empresa / getAuthenticatedTenant(req)
```

No se deben cambiar lecturas, escrituras, uploads, filtros, normalización, persistencia ni respuestas.

Demo/test queda fuera. No tocar todavía:

- `empresa_demo`;
- `test-empresa`;
- `test_empresa`;
- `empresa_maquinas_test`.

Antes y después del backfill hay que ejecutar checks SELECT-only. La condición de éxito es:

- `total_rows` no cambia;
- `empresa` text no cambia;
- `empresa_id` se llena sólo en Fase A;
- no hay mismatches `empresa_id` vs `empresa` normalizada;
- BPI behavior no cambia.

La Etapa 18 ya dejó definido que Prompt 19 debe aplicar sólo Fase A, sin tocar `bpi_agua_readings`, demo/test, BPI behavior, RCM, frontend, policies ni grants.

### Reglas

- Ejecutar primero sólo Fase A.
- No tocar `bpi_agua_readings` en el primer backfill.
- No tocar tablas con demo/test en el primer backfill.
- No tocar `bpi_upload_batches`, `bpi_maquinas` ni `bpi_upload_warnings` salvo aprobación explícita posterior.
- No tocar valores demo/test.
- Ejecutar checks SELECT-only antes y después.
- Registrar filas actualizadas por tabla.
- Mantener `empresa` text intacta.
- Mantener BPI usando `req.user.empresa` / `getAuthenticatedTenant(req)`.
- No cambiar queries/writes/uploads/filtros/normalización/persistencia/respuestas BPI.
- No crear índices, policies, aliases ni tenants demo/test.
- No tocar RCM/frontend/login/grants.

### Checklist de revisión

- [ ] ¿El backfill se limita a Fase A?
- [ ] ¿Se excluyó `bpi_agua_readings`?
- [ ] ¿Se excluyeron demo/test y tablas con demo/test?
- [ ] ¿Se ejecutaron checks SELECT-only antes?
- [ ] ¿Se ejecutaron checks SELECT-only después?
- [ ] ¿`total_rows` no cambió?
- [ ] ¿`empresa` text no cambió?
- [ ] ¿`empresa_id` se llenó sólo en Fase A?
- [ ] ¿No hay mismatches `empresa_id` vs `empresa` normalizada?
- [ ] ¿BPI behavior sigue igual?
- [ ] ¿No se tocó RCM/frontend/login/policies/grants?

### Relación con Prompt 19

`docs/security/prompt-19-proposed.md` autoriza sólo una futura ejecución de Fase A si el usuario lo aprueba explícitamente. No autoriza `bpi_agua_readings`, demo/test, índices, policies, aliases, tenants demo/test ni cambios de comportamiento BPI.

## Práctica 19: revisar Fase A y diseñar Fase B antes de ejecutar

Después de un backfill exitoso, no conviene ejecutar la siguiente fase de inmediato. Primero hay que revisar resultados, confirmar que no hubo mismatches y diseñar la siguiente fase con límites claros.

Fase B debe tratarse con más cuidado que Fase A porque ahora aparecen datos demo/test mezclados con datos mapeables. Las tablas candidatas son:

- `bpi_upload_batches`;
- `bpi_maquinas`.

Pero se deben excluir explícitamente:

- `empresa_demo`;
- `test-empresa`;
- `test_empresa`;
- `empresa_maquinas_test`.

Fase B todavía debe ser diseño, no ejecución. No debe hacer `UPDATE`, no debe crear migración real, no debe tocar `bpi_agua_readings`, no debe crear índices y no debe cambiar comportamiento BPI.

La Fase A quedó aplicada correctamente y sirve como prueba controlada: `bpi_energia_readings`, `bpi_produccion_records`, `bpi_ingestion_jobs` y `bpi_rutas_productivas` ya tienen `empresa_id`, sin mismatches y sin cambiar BPI. El Prompt 20 propuesto en el repo confirma que esta etapa debe revisar Fase A y diseñar Fase B, sin ejecutarla.

### Reglas

- Revisar resultados Fase A antes de planear Fase B.
- Confirmar que Fase A no tuvo mismatches.
- Confirmar que BPI behavior sigue usando `empresa` text.
- Fase B debe ser diseño-only hasta aprobación explícita.
- No ejecutar `UPDATE` en Fase B durante la etapa de diseño.
- No tocar `bpi_agua_readings`.
- No tocar `bpi_upload_warnings` salvo checks SELECT-only.
- Excluir demo/test de cualquier propuesta Fase B.
- No crear índices, policies, aliases, tenants demo/test ni migrations reales.
- No cambiar reads/writes/uploads/filtros/normalización/persistencia/respuestas BPI.
- No tocar RCM/frontend/login/grants.

### Checklist de revisión

- [ ] ¿Se revisaron resultados Fase A?
- [ ] ¿Fase A tiene 0 mismatches?
- [ ] ¿Fase A no cambió BPI behavior?
- [ ] ¿Fase B está limitada a `bpi_upload_batches` y `bpi_maquinas`?
- [ ] ¿Demo/test están excluidos explícitamente?
- [ ] ¿No se ejecutó `UPDATE`?
- [ ] ¿No se tocó `bpi_agua_readings`?
- [ ] ¿No se crearon índices/policies/aliases/tenants/migrations?
- [ ] ¿No se cambió BPI behavior?
- [ ] ¿No se tocó RCM/frontend/login/grants?

### Relación con Prompt 20

`docs/security/prompt-20-proposed.md` debe tratarse como revisión de Fase A y diseño de Fase B. No autoriza ejecución Fase B, `bpi_agua_readings`, demo/test, índices, policies, aliases, tenants demo/test ni cambios BPI.

## Práctica 20: ejecutar Fase B sólo con conteos confirmados y rollback claro

Fase B ya no es tan simple como Fase A porque sus tablas mezclan filas mapeables con filas demo/test. Por eso el backfill debe actualizar sólo filas mapeables y dejar fuera explícitamente:

- `empresa_demo`;
- `test-empresa`;
- `test_empresa`;
- `empresa_maquinas_test`.

El backfill debe seguir siendo reversible. Esta etapa sólo debe tocar `empresa_id` en:

- `bpi_upload_batches`;
- `bpi_maquinas`.

No debe modificar `empresa` text, no debe borrar datos, no debe cambiar uploads ni comportamiento BPI.

No tocar todavía `bpi_agua_readings`. Esa tabla tiene alto volumen y requiere estrategia por lotes en otra etapa. Tampoco tocar `bpi_upload_warnings`, salvo checks, porque actualmente no tiene filas.

Antes de ejecutar Fase B hay que confirmar que los conteos siguen iguales. El Prompt 21 propuesto exige validar que `bpi_upload_batches` tenga 3 filas mapeables y 39 demo/test, y que `bpi_maquinas` tenga 123 filas mapeables y 6 demo/test.

### Reglas

- Ejecutar checks SELECT-only antes de cualquier `UPDATE` Fase B.
- Confirmar `bpi_upload_batches`: 3 mapeables y 39 demo/test.
- Confirmar `bpi_maquinas`: 123 mapeables y 6 demo/test.
- Actualizar sólo filas con `empresa_id is null` y mapeo válido a `public.empresas.slug`.
- Excluir demo/test explícitamente en cada `UPDATE`.
- Tocar sólo `empresa_id`; no tocar `empresa` text.
- No borrar datos.
- No cambiar uploads ni comportamiento BPI.
- No tocar `bpi_agua_readings`.
- No tocar `bpi_upload_warnings` salvo checks SELECT-only.
- Mantener rollback por tabla y documentado.

### SQL conceptual

```sql
update public.bpi_upload_batches b
set empresa_id = e.id
from public.empresas e
where b.empresa_id is null
  and b.empresa not in ('empresa_demo', 'test-empresa', 'test_empresa', 'empresa_maquinas_test')
  and e.slug = lower(regexp_replace(trim(coalesce(b.empresa, '')), '[^a-zA-Z0-9]+', '_', 'g'));

update public.bpi_maquinas b
set empresa_id = e.id
from public.empresas e
where b.empresa_id is null
  and b.empresa not in ('empresa_demo', 'test-empresa', 'test_empresa', 'empresa_maquinas_test')
  and e.slug = lower(regexp_replace(trim(coalesce(b.empresa, '')), '[^a-zA-Z0-9]+', '_', 'g'));
```

Este SQL es conceptual. Sólo se ejecuta con aprobación explícita y después de checks limpios.

### Checklist de revisión

- [ ] ¿Los conteos Fase B siguen iguales?
- [ ] ¿`bpi_upload_batches` tiene 3 mapeables y 39 demo/test?
- [ ] ¿`bpi_maquinas` tiene 123 mapeables y 6 demo/test?
- [ ] ¿Se excluyen `empresa_demo`, `test-empresa`, `test_empresa`, `empresa_maquinas_test`?
- [ ] ¿Sólo se toca `empresa_id`?
- [ ] ¿`empresa` text sigue intacto?
- [ ] ¿No se tocó `bpi_agua_readings`?
- [ ] ¿No se tocó `bpi_upload_warnings` salvo checks?
- [ ] ¿No cambió BPI uploads/comportamiento?
- [ ] ¿Existe rollback documentado por tabla?

### Relación con Prompt 21

`docs/security/prompt-21-proposed.md` puede usarse sólo si el usuario aprueba ejecutar Fase B. Antes de ejecutar, debe confirmar conteos, exclusiones demo/test, no-touch de `bpi_agua_readings`, no cambios BPI y rollback claro.

## Práctica 21: diseñar Fase C por lotes antes de tocar bpi_agua_readings

Antes de tocar `bpi_agua_readings`, hay que detenerse a diseñar una estrategia por lotes. Esta tabla tiene alto volumen comparado con las otras, por eso no conviene hacer un `UPDATE` masivo sin plan.

Fase A y Fase B ya funcionaron como pruebas controladas. Eso significa que el mapeo `empresa` text → `public.empresas.slug` → `empresa_id` funciona, pero `bpi_agua_readings` requiere cuidado extra por cantidad de filas.

`bpi_agua_readings` todavía debe seguir usando `empresa` text como fuente efectiva. Aunque tenga `empresa_id`, no se debe cambiar comportamiento BPI, filtros, queries, reportes ni frontend.

Demo/test siguen fuera del backfill:

- `empresa_demo`;
- `test-empresa`;
- `test_empresa`;
- `empresa_maquinas_test`.

En `bpi_agua_readings` hay 235.845 filas mapeables y 2 demo/test; por eso el diseño debe llenar sólo las mapeables y dejar las demo/test en `empresa_id null`.

Esta etapa debe ser diseño, no ejecución. No debe actualizar `bpi_agua_readings` todavía. El Prompt 21 anterior dejó claro que Fase B no tocaba `bpi_agua_readings`; ahora corresponde diseñar Fase C, no ejecutarla.

### Reglas

- No ejecutar `UPDATE` sobre `bpi_agua_readings` durante diseño Fase C.
- Diseñar estrategia por lotes antes de cualquier ejecución.
- Mantener `empresa` text como fuente efectiva BPI.
- No cambiar filtros, queries, reportes, uploads ni frontend.
- Excluir demo/test explícitamente.
- Llenar sólo filas mapeables en una futura ejecución aprobada.
- Dejar demo/test en `empresa_id null`.
- Definir tamaño de lote, orden, checkpoints y post-checks.
- Definir rollback o plan de pausa/reanudación.
- No crear indexes/policies/grants/migrations reales durante diseño.
- No tocar RCM/frontend/login/auth.

### Checklist de revisión

- [ ] ¿La etapa es diseño-only?
- [ ] ¿No se ejecutó `UPDATE` sobre `bpi_agua_readings`?
- [ ] ¿Se reconoció el volumen alto de `bpi_agua_readings`?
- [ ] ¿Se diseñó estrategia por lotes?
- [ ] ¿Se excluyen demo/test?
- [ ] ¿Las 235.845 filas mapeables son el objetivo futuro?
- [ ] ¿Las 2 demo/test quedan `empresa_id null`?
- [ ] ¿BPI sigue usando `empresa` text?
- [ ] ¿No cambiaron filtros/queries/reportes/frontend?
- [ ] ¿No se crearon indexes/policies/grants/migrations?

### Relación con Prompt 22

`docs/security/prompt-22-proposed.md` debe tratarse como revisión de Fase B y diseño de Fase C por lotes para `bpi_agua_readings`. No autoriza ejecución de `bpi_agua_readings`, índices, policies, cambios BPI, RCM ni frontend.

## Práctica 22: ejecutar bpi_agua_readings sólo con piloto medido

Antes de tocar `bpi_agua_readings`, hay que diseñar una estrategia por lotes. Es la tabla más grande de BPI, y aunque Postgres puede manejar muchas filas, no conviene ejecutar un `UPDATE` masivo sin medir duración, locks y validaciones intermedias.

Fase A y Fase B ya demostraron que el mapeo funciona en tablas chicas y medianas. Ahora `bpi_agua_readings` debe tratarse como una fase aparte, con piloto pequeño, validación antes/después y rollback por lote.

No tocar demo/test. En `bpi_agua_readings` hay filas mapeables y también valores test/demo que deben seguir `empresa_id = null`:

- `empresa_demo`;
- `test-empresa`;
- `test_empresa`;
- `empresa_maquinas_test`.

Aunque se haga backfill, BPI sigue usando `empresa` text. No debe cambiar lectura, escritura, filtros, uploads, normalización, reportes ni frontend. El backfill sólo llena una columna auxiliar para preparar una migración futura.

### Reglas

- Ejecutar primero un lote piloto pequeño, no toda la tabla.
- Medir duración del lote.
- Observar posibles locks o degradación operativa.
- Validar antes y después del lote.
- Mantener rollback por lote.
- Excluir demo/test explícitamente.
- Mantener demo/test con `empresa_id null`.
- No modificar `empresa` text.
- No cambiar lecturas/escrituras/filtros/uploads/normalización/reportes/frontend.
- No crear indexes/policies/grants/migrations en el mismo cambio.
- No tocar RCM/login/auth.

### Checklist de revisión

- [ ] ¿El lote piloto es pequeño y explícito?
- [ ] ¿Se midió duración?
- [ ] ¿Se revisaron locks o impacto operativo?
- [ ] ¿Se ejecutaron pre-checks y post-checks?
- [ ] ¿El rollback por lote está documentado?
- [ ] ¿Demo/test siguen `empresa_id null`?
- [ ] ¿`empresa` text sigue intacto?
- [ ] ¿BPI sigue usando `empresa` text?
- [ ] ¿No cambiaron lecturas/escrituras/filtros/uploads/reportes/frontend?
- [ ] ¿No se mezclaron índices/policies/migrations con el backfill?

### Relación con Prompt 23

`docs/security/prompt-23-proposed.md` puede usarse sólo si el usuario aprueba ejecutar un primer lote piloto. No autoriza backfill completo de `bpi_agua_readings`, cambios BPI, demo/test, índices, policies, RCM ni frontend.

## Práctica 23: primer lote piloto de bpi_agua_readings por rango id

El primer backfill de `bpi_agua_readings` debe ser un piloto pequeño, no una migración completa. Esta tabla tiene más de 235 mil filas, por eso el primer lote debe servir para medir tiempo, validar conteos y confirmar que no aparecen mismatches.

Usar el rango por `id`, porque Etapa 22 confirmó que `id` es `bigint`, PK y confiable. El lote sugerido es:

```sql
id between 4 and 5003
```

Eso evita las filas demo/test detectadas en `id = 1` y `id = 3`, y permite probar aproximadamente 5.000 filas mapeables.

Si el primer lote falla o actualiza una cantidad inesperada, se debe detener. No avanzar a más lotes hasta revisar el resultado.

El backfill sigue sin cambiar BPI. Aunque se llene `empresa_id`, BPI debe seguir usando:

```text
req.user.empresa / getAuthenticatedTenant(req)
```

No se deben cambiar filtros, queries, uploads, reportes, frontend ni comportamiento.

El Prompt 23 propuesto en el repo autoriza sólo un primer lote piloto de `bpi_agua_readings`, no backfill completo, no índices, no policies, no RCM, no frontend y no demo/test.

### Reglas

- Ejecutar sólo el primer lote piloto aprobado.
- Usar rango `id between 4 and 5003` si los checks previos lo confirman.
- No ejecutar backfill completo.
- Medir tiempo de ejecución del lote.
- Validar conteos antes y después.
- Confirmar mismatches = 0.
- Confirmar demo/test siguen `empresa_id null`.
- Detener si el número actualizado no coincide con lo esperado.
- No avanzar a lotes siguientes sin revisión explícita.
- Mantener BPI usando `req.user.empresa / getAuthenticatedTenant(req)`.
- No cambiar filtros, queries, uploads, reportes, frontend ni comportamiento.
- No crear índices, policies, grants, migrations ni tocar RCM.

### SQL conceptual

```sql
update public.bpi_agua_readings b
set empresa_id = e.id
from public.empresas e
where b.empresa_id is null
  and b.id between 4 and 5003
  and b.empresa not in ('empresa_demo', 'test-empresa', 'test_empresa', 'empresa_maquinas_test')
  and e.slug = lower(regexp_replace(trim(coalesce(b.empresa, '')), '[^a-zA-Z0-9]+', '_', 'g'));
```

Este SQL es conceptual. Sólo se ejecuta con aprobación explícita y checks limpios.

### Checklist de revisión

- [ ] ¿El rango piloto es `id between 4 and 5003`?
- [ ] ¿Los checks previos confirman que el rango evita demo/test?
- [ ] ¿Se actualiza sólo `empresa_id`?
- [ ] ¿Se midió duración?
- [ ] ¿Filas actualizadas coinciden con lo esperado?
- [ ] ¿Mismatches = 0?
- [ ] ¿Demo/test siguen `empresa_id null`?
- [ ] ¿No se avanzó a más lotes?
- [ ] ¿BPI sigue usando `req.user.empresa / getAuthenticatedTenant(req)`?
- [ ] ¿No cambiaron filtros/queries/uploads/reportes/frontend/comportamiento?
- [ ] ¿No se crearon índices/policies/migrations?

## Práctica 24: escalar lotes gradualmente después del piloto

Después de un lote piloto exitoso, no se debe saltar directo al backfill completo. Primero se revisa el resultado, se define el siguiente tamaño de lote y se deja una estrategia de escalamiento gradual.

El piloto de `bpi_agua_readings` fue correcto:

- `id 4 → 5003`;
- 5.000 filas actualizadas;
- 0 mismatches;
- demo/test intactos;
- BPI behavior sin cambios.

Ahora conviene subir a 10.000 filas, pero sólo para el siguiente lote. Si pasa, recién ahí diseñamos o ejecutamos el siguiente lote de 20.000. El Prompt 24 propuesto permite revisar el piloto, decidir el siguiente tamaño y proponer rangos explícitos, sin ejecutar backfill completo automático.

### Reglas

- No saltar de piloto exitoso a backfill completo.
- Revisar resultado del piloto antes de cualquier lote nuevo.
- Usar un siguiente lote explícito de 10.000 filas si el piloto fue PASS.
- No ejecutar lote de 20.000 hasta que el lote de 10.000 tenga PASS.
- Mantener rangos `id` explícitos.
- Ejecutar checks antes y después de cada lote.
- Medir duración e impacto operativo por lote.
- Detener si aparecen mismatches, conteos inesperados o demo/test.
- Mantener BPI behavior sin cambios.
- No crear indexes/policies/grants/migrations en el mismo cambio.

### Checklist de revisión

- [ ] ¿Se revisó el piloto `4..5003`?
- [ ] ¿El piloto actualizó 5.000 filas?
- [ ] ¿El piloto tuvo 0 mismatches?
- [ ] ¿Demo/test siguen intactos?
- [ ] ¿BPI behavior sigue igual?
- [ ] ¿El próximo lote está limitado a 10.000 filas?
- [ ] ¿El rango `id` del próximo lote es explícito?
- [ ] ¿No se autorizó backfill completo automático?
- [ ] ¿No se propuso 20.000 antes de validar 10.000?
- [ ] ¿Prompt 24 sigue siendo revisión/decisión, no ejecución masiva?

### Relación con Prompt 24

`docs/security/prompt-24-proposed.md` debe revisar el piloto, elegir el siguiente tamaño y proponer rangos explícitos. No autoriza backfill completo automático ni cambios BPI, índices, policies, RCM o frontend.

## Práctica 25: ejecutar sólo el lote 10.000 aprobado

Después de que el piloto de 5.000 filas fue PASS y la Etapa 24 dejó preparado el rango, el siguiente paso es ejecutar sólo un lote de 10.000 filas, no más. Aunque se quiera escalar a 20.000, 40.000 o 100.000, cada salto debe tener su propia verificación.

El rango aprobado para esta etapa es:

```text
id 5004 → 15003
```

Condición de éxito:

- 10.000 filas actualizadas;
- 0 mismatches;
- demo/test intactos;
- Fase A/B intactas;
- lote piloto intacto;
- BPI behavior sin cambios.

### Reglas

- Ejecutar sólo `id 5004 → 15003`.
- No ejecutar lote de 20.000 en la misma etapa.
- No ejecutar 40.000, 100.000 ni remanente final.
- No ejecutar backfill completo automático.
- Ejecutar checks SELECT-only antes y después.
- Medir duración con timestamps externos.
- Actualizar sólo `empresa_id` desde `null` a UUID válido.
- Excluir demo/test explícitamente.
- Detener si el conteo actualizado no es 10.000.
- Detener si aparece mismatch.
- Mantener Fase A/B y piloto intactos.
- Mantener BPI usando `req.user.empresa / getAuthenticatedTenant(req)`.
- No cambiar filtros, queries, uploads, reportes, frontend ni comportamiento.

### Checklist de revisión

- [ ] ¿El rango ejecutado es sólo `5004..15003`?
- [ ] ¿Se actualizaron exactamente 10.000 filas?
- [ ] ¿Mismatches = 0?
- [ ] ¿Demo/test siguen intactos?
- [ ] ¿Fase A/B siguen intactas?
- [ ] ¿El piloto `4..5003` sigue intacto?
- [ ] ¿BPI behavior sigue sin cambios?
- [ ] ¿No se ejecutó lote de 20.000 o mayor?
- [ ] ¿No se ejecutó backfill completo?
- [ ] ¿Se documentó duración y rollback del lote?

### Relación con Prompt 25

`docs/security/prompt-25-proposed.md` autoriza sólo el lote `id 5004 → 15003` si el usuario aprueba ejecutarlo y los checks previos coinciden. No autoriza lotes mayores, backfill completo, índices, policies, RCM, frontend ni cambios BPI.

## Práctica 26: revisión final antes de cambiar código tras backfill por etapas

Después de terminar un backfill por etapas, no se debe pasar inmediatamente a cambiar el código. Primero hay que hacer una revisión final de consistencia: conteos por tabla, mismatches, demo/test, rollback disponible y confirmación de que BPI aún usa `empresa` text.

El backfill de `bpi_agua_readings` quedó completo para filas mapeables:

| Métrica                 |   Valor |
| ----------------------- | ------: |
| total_rows              | 235.847 |
| rows_with_empresa_id    | 235.845 |
| rows_without_empresa_id |       2 |
| demo/test null          |       2 |
| mismatches              |       0 |

El lote final quedó documentado en el commit `16c5fb5`, con `empresa_demo` y `test-empresa` manteniéndose en `empresa_id null`, y sin cambios de comportamiento BPI.

### Reglas

- Hacer revisión final antes de cambiar código BPI.
- Confirmar conteos por tabla.
- Confirmar `rows_with_mismatched_empresa_id = 0`.
- Confirmar demo/test con `empresa_id null`.
- Confirmar rollback disponible por lote.
- Confirmar que `empresa` text sigue existiendo.
- Confirmar que BPI sigue usando `req.user.empresa / getAuthenticatedTenant(req)`.
- No cambiar filtros, queries, uploads, reportes, frontend ni comportamiento en la misma etapa.
- No crear indexes/policies/grants/migrations junto con la revisión final.
- Diseñar cualquier migración de comportamiento como etapa separada.

### Checklist de revisión

- [ ] ¿Todas las tablas BPI tienen conteos finales revisados?
- [ ] ¿`bpi_agua_readings` tiene 235.845 filas con `empresa_id`?
- [ ] ¿Sólo quedan 2 filas null demo/test?
- [ ] ¿Mismatches = 0?
- [ ] ¿`empresa_demo` y `test-empresa` siguen null?
- [ ] ¿Hay rollback documentado por lote?
- [ ] ¿BPI sigue usando `empresa` text?
- [ ] ¿No se cambió código BPI?
- [ ] ¿No se cambiaron frontend/reportes/filtros/uploads?
- [ ] ¿No se mezclaron indexes/policies/migrations con esta revisión?

### Relación con prompt final

`docs/security/prompt-final-bpi-backfill-review.md` debe usarse para revisión final de consistencia. No autoriza migración de comportamiento, indexes, policies, RCM, frontend ni mapeo de demo/test.

## Práctica 27: diseñar índices empresa_id antes de cambiar comportamiento BPI

Después de completar un backfill, el siguiente paso no es cambiar el comportamiento del sistema inmediatamente. Primero hay que diseñar índices para que cuando BPI empiece a consultar por `empresa_id`, las lecturas sigan siendo rápidas y no dependan sólo de `empresa` text.

Diseñar índices no significa crearlos de inmediato. Primero hay que revisar cómo consulta BPI hoy y proponer equivalentes con `empresa_id`.

Patrones legacy a revisar:

- `empresa + fecha`;
- `empresa + producto`;
- `empresa + área`;
- `empresa + tipo`;
- `empresa + dispositivo`;
- `empresa + created_at`.

No hay que crear índices innecesarios. Cada índice acelera consultas específicas, pero también agrega costo en escrituras, uploads y backfills futuros. Por eso esta etapa debe ser diseño/proposal-only.

`bpi_agua_readings` es la tabla más grande, así que cualquier índice ahí debe elegirse con cuidado. Conviene revisar primero los índices legacy existentes y crear equivalentes sólo para rutas realmente usadas.

El repo ya dejó propuesto que la siguiente etapa sea diseñar índices `empresa_id` para BPI, sin crearlos todavía, sin cambiar comportamiento, sin tocar frontend, sin policies/RLS y sin tocar RCM.

### Reglas

- Hacer etapa de diseño/proposal-only antes de crear índices.
- Revisar rutas BPI reales antes de proponer índices.
- Revisar índices legacy existentes antes de proponer equivalentes.
- Priorizar índices que correspondan a queries reales.
- Proponer equivalentes `empresa_id + ...` para patrones usados.
- Tratar `bpi_agua_readings` como tabla de alto riesgo por tamaño.
- Evaluar costo de escritura/upload por cada índice propuesto.
- No crear índices durante la etapa de diseño.
- No cambiar comportamiento BPI durante el diseño de índices.
- No tocar frontend, RCM, policies/RLS, grants ni migrations ejecutables en esta etapa.

### Checklist de revisión

- [ ] ¿Se leyeron las rutas BPI que consultan por `empresa`?
- [ ] ¿Se identificaron patrones `empresa + fecha/producto/área/tipo/dispositivo/created_at`?
- [ ] ¿Se revisaron índices legacy existentes?
- [ ] ¿Cada índice propuesto corresponde a una query real?
- [ ] ¿Se evitó proponer índices redundantes?
- [ ] ¿Se evaluó el costo de escritura/upload?
- [ ] ¿`bpi_agua_readings` fue tratado con criterio especial por tamaño?
- [ ] ¿La salida es proposal-only?
- [ ] ¿No se creó ningún índice?
- [ ] ¿No se cambió BPI behavior/frontend/RCM/RLS?

### Relación con prompt siguiente

`docs/security/prompt-next-after-bpi-backfill.md` recomienda como siguiente etapa diseñar índices `empresa_id` para BPI, sin crearlos todavía. Esa etapa no autoriza cambios de comportamiento, frontend, policies/RLS, RCM ni mapeo de demo/test.

## Práctica 28: diseñar uso backend de empresa_id en paralelo antes de reemplazar empresa text

Aunque `empresa_id` ya está backfilleado, no conviene reemplazar de golpe `empresa` text en BPI. Primero hay que diseñar cómo el backend usará `empresa_id` de forma paralela y validada, manteniendo `empresa` text como fallback.

El frontend no debe cambiar todavía. La transición correcta es:

1. Backend resuelve tenant actual.
2. Backend valida `empresa` text vs `empresa_id`.
3. Backend puede preparar helpers internos.
4. Luego se migra lectura/escritura por endpoint.
5. Recién después se cambia frontend si hace falta.

No tocar RLS/policies todavía. Las tablas BPI siguen siendo backend-only y el backend controla acceso con `service_role`. Cambiar RLS, frontend y queries al mismo tiempo genera demasiado riesgo.

Demo/test siguen fuera del flujo normal. No se deben forzar a `empresa_id`, no se deben mapear a empresas reales y no se deben usar para autorizar.

El estado final documentado dejó explícito que `empresa_id` ya está completo para filas mapeables, pero BPI todavía usa `empresa` text como comportamiento efectivo. La siguiente opción recomendada después de índices era diseñar uso backend de `empresa_id` sin cambiar frontend.

### Reglas

- Diseñar uso backend de `empresa_id` antes de reemplazar `empresa` text.
- Mantener `empresa` text como fallback durante la transición.
- Validar consistencia entre `empresa` text y `empresa_id` antes de usar `empresa_id` como filtro efectivo.
- Preparar helpers internos antes de migrar endpoints.
- Migrar lectura/escritura endpoint por endpoint, no globalmente.
- No cambiar frontend en la etapa de diseño backend.
- No tocar RLS/policies/grants en la misma etapa.
- No usar demo/test para autorización.
- No mapear demo/test a empresas reales.
- No mezclar cambio de queries, frontend, RLS y uploads en una sola etapa.

### Checklist de revisión

- [ ] ¿El diseño mantiene `empresa` text como fallback?
- [ ] ¿Existe validación `empresa` text vs `empresa_id`?
- [ ] ¿Los helpers internos están definidos antes de migrar endpoints?
- [ ] ¿La migración está separada por endpoint?
- [ ] ¿No se cambió frontend?
- [ ] ¿No se tocaron RLS/policies/grants?
- [ ] ¿No se usaron demo/test para autorización?
- [ ] ¿No se mapearon demo/test a empresas reales?
- [ ] ¿No se cambió BPI behavior durante el diseño?
- [ ] ¿La etapa sigue siendo backend-only/proposal antes de aplicar?

### Relación con etapas futuras

Después de diseñar índices y helpers backend, una etapa futura puede migrar endpoints BPI de a uno. Esa etapa debe conservar fallback por `empresa` text, medir resultados, validar mismatches y no tocar frontend/RLS salvo aprobación separada.

## Práctica 29: implementar sólo helper aislado antes de conectar rutas BPI

Ahora que el diseño del uso backend de `empresa_id` está aprobado, el siguiente paso debe ser implementar sólo el helper aislado, no conectarlo todavía a rutas productivas.

Este helper debe permitir que el backend resuelva:

- `req.user.empresa`;
- `req.tenantContext.currentEmpresaSlug`;
- `public.empresas.slug`;
- `public.empresas.id`.

Pero todavía no debe cambiar cómo BPI lee o escribe datos.

El frontend nunca debe ser autoridad para `empresa_id`. Si en el futuro llega un `empresa_id` por body/query/frontend, el backend debe ignorarlo para autorización y derivar el contexto desde sesión/backend.

Mismatch no debe romper producción todavía. Si `req.tenantContext.currentEmpresaSlug` no coincide con `req.user.empresa`, el helper debe devolver warnings y mantener fallback a `empresa` text.

Demo/test no se mapea automáticamente:

- `empresa_demo`;
- `test-empresa`;
- `test_empresa`;
- `empresa_maquinas_test`.

### Reglas

- Implementar sólo helper aislado y tests unitarios.
- No conectar el helper a rutas productivas todavía.
- No cambiar queries BPI reales.
- No cambiar writes/uploads/persistence/response shape.
- Resolver `empresa_id` sólo desde backend/sesión/repositorio, nunca desde frontend como autoridad.
- En mismatch, devolver warning y fallback a `empresa` text.
- En demo/test, no mapear automáticamente y no autorizar con `empresa_id`.
- Mantener `canUseEmpresaIdForRead = false` y `canUseEmpresaIdForWrite = false` hasta etapa autorizada.
- No tocar frontend, RCM, RLS/policies, indexes ni migrations.

### Checklist de revisión

- [ ] ¿Sólo se creó helper aislado?
- [ ] ¿Hay tests unitarios para tenantContext correcto?
- [ ] ¿Hay tests para fallback sin tenantContext?
- [ ] ¿Hay tests para mismatch con warning?
- [ ] ¿Hay tests para empresa no existente?
- [ ] ¿Hay tests para demo/test no mapeable?
- [ ] ¿El frontend no es autoridad para `empresa_id`?
- [ ] ¿No se conectó a `routes/bpi.js` ni `routes/bpiUpload.js`?
- [ ] ¿No cambió BPI behavior?
- [ ] ¿No se tocaron frontend/RCM/RLS/indexes/migrations?

### Relación con prompt siguiente

`docs/security/prompt-next-bpi-empresa-id-backend-resolver.md` propone implementar `utils/bpiEmpresaResolver.js` y tests unitarios, sin conectarlo a rutas productivas ni cambiar comportamiento BPI.

## Práctica 30: conectar bpiEmpresaResolver sólo en diagnóstico

Ahora que `bpiEmpresaResolver` existe y está probado, el siguiente paso debe ser conectarlo sólo en diagnóstico, no como filtro efectivo. BPI debe seguir usando:

```text
req.user.empresa / getAuthenticatedTenant(req)
```

El helper debe observar y registrar diferencias, no cambiar respuestas. Si detecta `empresa_id`, mismatch, fallback o demo/test, sólo debe generar warnings seguros. No debe cambiar queries, uploads, normalización ni persistencia.

No conectar todo BPI de golpe. Primero conviene seleccionar puntos seguros de lectura y/o contexto general, donde el resolver pueda ejecutarse sin alterar el resultado.

El prompt anterior dejó claro que el helper existe, está testeado y no debe cambiar rutas productivas hasta una etapa explícita.

### Reglas

- Conectar `bpiEmpresaResolver` sólo en modo diagnóstico.
- Mantener `req.user.empresa / getAuthenticatedTenant(req)` como fuente efectiva.
- No usar `empresa_id` como filtro real todavía.
- No cambiar response shape.
- No cambiar uploads, normalización ni persistencia.
- Registrar sólo warnings seguros, sin datos sensibles.
- Seleccionar pocos puntos seguros; no conectar todo BPI de golpe.
- No tocar frontend, RCM, RLS/policies, indexes ni migrations.
- No mapear demo/test ni usarlos para autorizar.

### Checklist de revisión

- [ ] ¿El resolver se usa sólo para diagnóstico?
- [ ] ¿Las queries siguen usando `empresa` text?
- [ ] ¿`getAuthenticatedTenant(req)` sigue siendo fuente efectiva?
- [ ] ¿No cambió response shape?
- [ ] ¿No cambiaron uploads/normalización/persistencia?
- [ ] ¿Los warnings son seguros y sin datos sensibles?
- [ ] ¿Se conectaron pocos puntos seguros, no todo BPI?
- [ ] ¿Demo/test siguen fuera del flujo normal?
- [ ] ¿No se tocaron frontend/RCM/RLS/indexes/migrations?

### Relación con prompt siguiente

`docs/security/prompt-next-bpi-empresa-resolver-diagnostics.md` propone conectar el helper en modo diagnóstico a rutas BPI seleccionadas, manteniendo `empresa` text como filtro efectivo y sin cambiar respuestas.

## Práctica 31: protocolo general antes de programar con IA

Antes de modificar código, la IA debe leer la estructura real del repo, identificar el stack y ubicar backend, frontend, DB, rutas, helpers y tests. No debe asumir framework, ORM, tabla, columna, variable, ruta ni contrato.

Buscar archivos relacionados antes de proponer cambios es parte del trabajo, no un extra. La IA debe documentar qué archivos leyó y qué encontró.

### Reglas

- No programar por intuición.
- No asumir framework, ORM, tabla, columna, ruta, helper ni variable.
- Leer estructura del repo antes de tocar código.
- Identificar backend, frontend, DB, rutas, helpers y tests reales.
- Buscar archivos relacionados antes de proponer cambios.
- No crear archivos nuevos si ya existe un patrón equivalente.
- No duplicar lógica sin revisar helpers existentes.
- No tocar varias capas a la vez salvo aprobación explícita.
- Si la tarea toca BD, usar esta skill completa antes.
- Documentar qué archivos se leyeron antes de cambiar.

### Checklist de revisión

- [ ] ¿Se revisó el repo?
- [ ] ¿Se identificó stack real?
- [ ] ¿Se leyeron rutas/helpers/tests relacionados?
- [ ] ¿Se evitó inventar nombres?
- [ ] ¿Se identificó dónde vive la lógica actual?
- [ ] ¿Se sabe qué no se debe tocar?

## Práctica 32: no inventar tablas, columnas, rutas ni contratos

Una IA no debe crear nombres de tablas, columnas, rutas, enums, roles, estados ni contratos porque “parecen correctos”. Debe confirmar existencia con schema, migrations, código o documentación del repo.

Si necesita una nueva columna o tabla, debe pasar por etapa de diseño y migración. No asumir que `id`, `user_id`, `empresa_id`, `tenant_id` o `auth.uid()` existen donde no se verificó.

### Reglas

- Antes de usar una tabla, confirmar que existe.
- Antes de usar una columna, confirmar que existe.
- Antes de usar una ruta, confirmar que existe.
- Antes de cambiar un contrato API, revisar el frontend consumidor.
- Antes de crear una nueva tabla, revisar si ya existe una equivalente.
- No inventar enums, roles ni estados.
- No asumir `id`, `user_id`, `empresa_id`, `tenant_id` ni `auth.uid()` sin evidencia.
- Si hace falta una nueva pieza de schema, diseñarla y migrarla en etapa separada.

### Checklist de revisión

- [ ] ¿La tabla existe?
- [ ] ¿La columna existe?
- [ ] ¿La ruta existe?
- [ ] ¿El frontend consume esa respuesta?
- [ ] ¿Hay tests existentes?
- [ ] ¿Hay migrations relacionadas?
- [ ] ¿La nueva pieza realmente es necesaria?

## Práctica 33: separar diseño, implementación y comportamiento

Diseño no es implementación. Implementación interna no es cambio de comportamiento. Cambio de estructura no es backfill. Backfill no es cambio de queries. Cambio de queries no es cambio de frontend.

Separar etapas reduce riesgo y permite revisar, probar y revertir cada paso.

### Reglas

- Una etapa debe tener un solo objetivo.
- No mezclar migración + backfill + frontend.
- No mezclar helper + conexión a rutas + cambio de respuesta.
- No mezclar índices + cambio de comportamiento.
- No mezclar RLS + frontend + backend en un solo PR.
- Todo cambio debe tener rollback o forma de desactivarse.
- Definir explícitamente qué cambia y qué no cambia.
- Definir qué etapa viene después sin ejecutarla automáticamente.

### Checklist de revisión

- [ ] ¿La etapa tiene un solo objetivo?
- [ ] ¿Qué cambia?
- [ ] ¿Qué no cambia?
- [ ] ¿Cómo se prueba?
- [ ] ¿Cómo se revierte?
- [ ] ¿Qué etapa viene después?

## Práctica 34: patrón obligatorio para nuevas funciones

Antes de agregar una función nueva como máquinas, rutas, costos, repuestos, dashboard o RCM, la IA debe entender datos, rutas, UI, contrato y tenant. No agregar pantallas, endpoints ni tablas antes de saber qué existe.

### Secuencia obligatoria

1. Inventario de datos existentes.
2. Inventario de rutas existentes.
3. Inventario de frontend consumidor.
4. Diseño de contrato.
5. Plan de tablas/columnas si hacen falta.
6. Tests mínimos.
7. Implementación pequeña.
8. Validación no-touch.

### Reglas

- No agregar pantallas antes de entender datos.
- No agregar endpoints antes de definir contrato.
- No agregar tablas antes de revisar schema actual.
- No escribir directo a DB sin validar tenant/empresa.
- No crear lógica duplicada si hay helpers.
- No tocar RCM si la tarea es BPI, ni BPI si la tarea es RCM, salvo aprobación.
- Mantener cada nueva función acotada y testeable.

### Checklist de revisión

- [ ] ¿Qué usuario usará la función?
- [ ] ¿Qué dato necesita?
- [ ] ¿Dónde vive ese dato hoy?
- [ ] ¿Existe tabla?
- [ ] ¿Existe endpoint?
- [ ] ¿Existe UI?
- [ ] ¿Qué contrato API tendrá?
- [ ] ¿Qué validaciones de tenant/empresa requiere?
- [ ] ¿Qué tests mínimos tendrá?

## Práctica 35: seguridad para archivos, uploads e integraciones

Cualquier upload o integración externa es superficie de ataque. Hay que validar tipo, tamaño, estructura y columnas esperadas. No confiar en nombres de archivo ni procesar payloads enormes sin límites.

No guardar archivos corruptos como válidos. Usar cuarentena si corresponde. Loguear de forma segura sin exponer datos sensibles.

### Reglas

- Validar MIME/extensión/contenido.
- Validar columnas esperadas.
- Limitar tamaño.
- Sanitizar nombres.
- Rechazar o poner en cuarentena datos inválidos.
- No exponer stack traces al frontend.
- No loguear archivos completos.
- No loguear tokens, cookies, `service_role` ni payloads sensibles.
- No asumir que una integración externa respeta contrato.

### Checklist de revisión

- [ ] ¿Hay límite de tamaño?
- [ ] ¿Se validan columnas?
- [ ] ¿Se valida tipo de archivo?
- [ ] ¿Hay manejo de error seguro?
- [ ] ¿Hay rollback o cuarentena?
- [ ] ¿Se evita log sensible?

## Práctica 36: plantilla de entrega obligatoria para IA programadora

Cada respuesta final de la IA que programe debe reportar qué creó, qué modificó, qué leyó, qué cambió, qué no cambió, qué tests ejecutó y cómo se revierte.

### Reglas

- Reportar archivos creados.
- Reportar archivos modificados.
- Reportar lecturas previas.
- Reportar cambio realizado.
- Reportar no-touch confirmado.
- Reportar tests ejecutados y resultado.
- Reportar validaciones.
- Reportar rollback.
- Reportar riesgos pendientes.
- Reportar commit sugerido.
- Reportar siguiente paso recomendado.

### Plantilla

```text
Entrega obligatoria:

Archivos creados:
- ...

Archivos modificados:
- ...

Lecturas previas:
- ...

Cambio realizado:
- ...

No-touch confirmado:
- ...

Tests:
- comando
- resultado

Validaciones:
- ...

Rollback:
- ...

Riesgos pendientes:
- ...

Siguiente paso:
- ...
```

### Checklist de revisión

- [ ] ¿La entrega dice qué leyó antes de cambiar?
- [ ] ¿La entrega separa creados de modificados?
- [ ] ¿La entrega incluye tests y resultados?
- [ ] ¿La entrega incluye no-touch?
- [ ] ¿La entrega incluye rollback?
- [ ] ¿La entrega incluye riesgos y siguiente paso?

## Práctica 37: regla de oro para sistemas multiempresa

Nunca confiar en `empresa` o `empresa_id` enviada por frontend. El tenant debe derivarse del backend. Si todavía hay compatibilidad legacy, mantener `empresa` text mientras se valida `empresa_id` contra membresía o contexto backend.

`service_role` sólo debe usarse en backend. El frontend nunca debe tener secretos. Toda nueva función debe responder cómo evita mezclar datos entre empresas.

### Reglas

- Nunca confiar en empresa enviada por frontend.
- Derivar tenant desde sesión/backend.
- Validar membresía cuando aplique.
- Mantener `empresa` text legacy si todavía hay compatibilidad.
- Validar `empresa_id` contra membresía o tenant backend antes de usarlo como autoridad.
- Usar `service_role` sólo en backend.
- Nunca exponer secretos al frontend.
- Definir fallback para migraciones graduales.
- Tratar demo/test como caso explícito, no como tenant real por defecto.
- Toda nueva función debe explicar cómo evita mezcla de datos entre empresas.

### Checklist de revisión

- [ ] ¿De dónde sale tenant?
- [ ] ¿Se valida membresía?
- [ ] ¿Se usa `empresa_id` o `empresa` text?
- [ ] ¿Hay fallback?
- [ ] ¿Qué pasa con demo/test?
- [ ] ¿El frontend puede manipular empresa?
- [ ] ¿Se usa `service_role` sólo backend?
- [ ] ¿La función evita mezclar datos entre empresas?

## Práctica 38: checklist Supabase obligatorio antes de cambios DB

Antes de tocar Supabase/Postgres, la IA debe revisar seguridad, exposición, RLS, grants, funciones, views, secrets y migraciones. En Supabase, una tabla en schema expuesto como `public` puede quedar accesible por Data API; por eso no alcanza con “la usa sólo el backend” si grants/RLS no fueron revisados.

### Reglas

- Asumir que tablas en `public` son sensibles hasta demostrar lo contrario.
- Decidir explícitamente si una tabla es `backend-only`, pública real o accesible por frontend con RLS.
- RLS debe estar diseñado antes de exponer tablas a `anon`/`authenticated`.
- No usar `user_metadata` / `raw_user_meta_data` para autorización; es editable por el usuario.
- Usar `app_metadata`, tablas de membresía o backend validado para autorización.
- Views pueden saltar RLS según configuración; revisar `security_invoker` o grants antes de exponerlas.
- No poner funciones `security definer` en schemas expuestos si se puede evitar.
- Toda función `security definer` debe fijar `search_path`, minimizar grants y documentar por qué existe.
- Recordar que `UPDATE` con RLS también necesita policy de `SELECT` sobre la fila.
- `service_role` sólo backend; nunca frontend, logs, bundles ni variables públicas.
- Después de DDL real, correr advisors de seguridad/performance si están disponibles.
- DDL real requiere plan, rollback, checks pre/post y aprobación explícita.

### Checklist de revisión

- [ ] ¿La tabla está en schema expuesto?
- [ ] ¿Es backend-only, pública real o frontend con RLS?
- [ ] ¿RLS está habilitado/diseñado si corresponde?
- [ ] ¿No se usa `user_metadata` para autorización?
- [ ] ¿Views y funciones tienen grants seguros?
- [ ] ¿Funciones `security definer` tienen `search_path` seguro?
- [ ] ¿UPDATE con RLS tiene SELECT policy necesaria?
- [ ] ¿`service_role` queda sólo en backend?
- [ ] ¿Se corrieron o planificaron advisors tras DDL?
- [ ] ¿Hay rollback y checks pre/post?

## Práctica 39: optimización Supabase/Postgres basada en queries reales

Optimizar no es agregar índices “por si acaso”. En Supabase/Postgres, cada índice acelera algunas lecturas pero encarece writes, uploads, backfills y mantenimiento. Todo índice debe responder a una query real y tener plan de medición.

### Reglas

- No crear índices sin mapear queries reales.
- Revisar rutas/backend y filtros antes de proponer índices.
- Confirmar columnas reales antes de escribir SQL.
- Medir volumen de tabla y tamaño de índices existentes.
- Evitar índices redundantes o solapados.
- En tablas grandes/vivas, considerar `CREATE INDEX CONCURRENTLY` en etapa aprobada.
- No mezclar creación de índices con cambio de comportamiento.
- Documentar rollback con `DROP INDEX` explícito.
- Recordar que uploads/backfills se vuelven más caros con más índices.
- Usar `EXPLAIN` sólo en entorno seguro o con aprobación si puede ser costoso.
- Diferir índices para tablas vacías o rutas no usadas.

### Checklist de revisión

- [ ] ¿Qué query real acelera este índice?
- [ ] ¿Qué ruta o job usa esa query?
- [ ] ¿La columna existe y tiene datos?
- [ ] ¿Ya hay un índice equivalente?
- [ ] ¿La tabla es grande o recibe uploads frecuentes?
- [ ] ¿El índice afecta writes/backfills?
- [ ] ¿Se necesita `CONCURRENTLY`?
- [ ] ¿Hay rollback con nombre de índice exacto?
- [ ] ¿La etapa es sólo índices, sin cambio de comportamiento?

## Guía rápida para IA: uso operativo de esta skill

Esta skill es larga porque conserva historial y decisiones. Para usarla rápido, aplicar este orden:

1. **Entender antes de tocar**: leer repo, stack, rutas, helpers, schema/checks y tests relacionados.
2. **Verificar existencia**: no inventar tablas, columnas, rutas, IDs, helpers, roles, enums ni contratos.
3. **Definir etapa única**: diseño, migración, backfill, índices, helper, diagnóstico, comportamiento o frontend; sólo una cosa por etapa.
4. **Proteger tenant**: backend deriva tenant; frontend no es autoridad; `service_role` sólo backend.
5. **Supabase seguro**: revisar RLS, grants, views, RPC/security definer, policies, advisors y secrets.
6. **Performance real**: índices sólo para queries reales, con costo de writes/uploads y rollback.
7. **No-touch explícito**: listar lo que no se toca y validarlo con diff/grep/checks.
8. **Tests primero cuando hay código**: escribir/ejecutar tests enfocados y relacionados.
9. **Entrega completa**: reportar archivos, lecturas, cambios, no-touch, tests, rollback, riesgos y siguiente paso.

### Atajos de decisión

- Si no existe evidencia de una tabla/columna/ruta: **parar y verificar**.
- Si una etapa mezcla DB + backend + frontend: **separar fases**.
- Si toca datos: **checks SELECT-only antes/después + rollback**.
- Si toca Supabase expuesto: **RLS/grants/policies/advisors**.
- Si toca multiempresa: **tenant backend, no frontend**.
- Si toca uploads: **validar archivo, tamaño, columnas y logs seguros**.
- Si cambia comportamiento: **diseño + tests + fallback**.

## Caso completo: legacy empresa text → multi-tenant operativo

Ejemplo de camino seguro para migrar un sistema legacy que usa `empresa` text hacia multi-tenant con `empresa_id`, sin romper producción:

1. **Inventario inicial**: leer repo, rutas, helpers, tests, schema y migrations. Usar Prácticas 31–32.
2. **Clasificación Supabase**: decidir backend-only vs frontend/RLS, revisar grants, funciones y exposure. Usar Prácticas 1–4 y 38.
3. **Diseño tenant/auth**: definir `empresas`, membresías, fallback legacy y qué no se toca. Usar Prácticas 5–8 y 37.
4. **Base estructural**: agregar tablas/columnas con migrations aprobadas, rollback y checks. Usar Prácticas 3, 4, 15 y 16.
5. **Backfill gradual**: poblar `empresa_id` por fases/lotes con checks SELECT-only, mismatches y demo/test controlados. Usar Prácticas 17–25.
6. **Revisión final**: confirmar conteos, rollback, demo/test y que el sistema sigue usando `empresa` text. Usar Práctica 26.
7. **Índices por diseño**: revisar queries reales y proponer índices `empresa_id` sin crearlos todavía. Usar Prácticas 27, 38 y 39.
8. **Helper backend aislado**: implementar resolver testeable que derive `empresa_id` desde backend, no frontend. Usar Prácticas 28–29.
9. **Diagnóstico-only**: conectar helper para observar warnings sin cambiar queries ni responses. Usar Práctica 30.
10. **Lectura dual futura**: comparar conteos por `empresa` text vs `empresa_id` sin cambiar frontend. Usar Prácticas 33, 36 y 39.
11. **Cambio efectivo gradual**: migrar endpoints pequeños con fallback, tests, rollback y no-touch frontend/RLS salvo aprobación separada. Usar Prácticas 33, 34, 37 y 38.

Regla del caso completo: cada paso debe terminar con evidencia, tests/checks y un prompt/plan para el siguiente paso; nunca ejecutar el siguiente paso por impulso.

## Cómo actualizar esta skill

Cuando el usuario agregue una nueva buena práctica de BD, incorporala como una nueva sección numerada (`Práctica 5`, `Práctica 6`, etc.) manteniendo:

- explicación corta;
- reglas accionables;
- ejemplo SQL si aplica;
- checklist de revisión.
