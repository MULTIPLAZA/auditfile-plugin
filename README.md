# AuditFile — plugin para Claude Code

Audita la salud de los datos de un sistema tipo ERP/POS (PRIMATE, MIXIT, mi-pos, GESIMP, o cualquier otro) en 4 ejes: **Aprovechamiento, Integridad, Latencia y Veracidad**. Entrega un informe con hallazgos concretos y priorizados — no arregla nada solo, diagnostica.

## Instalación

Dentro de una sesión de Claude Code:

```
/plugin marketplace add <usuario>/auditfile-plugin
/plugin install auditfile@auditfile-marketplace
```

(Reemplazá `<usuario>` por el usuario/organización de GitHub donde quede publicado este repo.)

Una vez instalado, el skill queda disponible como `/AuditFile`, y también se activa solo cuando le pedís cosas como "auditá los datos de X" o "revisá la salud del dato".

## Qué necesita para funcionar

AuditFile no trae ninguna dependencia de código — es un solo archivo de instrucciones (`SKILL.md`). Lo único que necesita en cada proyecto donde se use es:

- Acceso de solo lectura a la base de datos del sistema a auditar (SQL Server, D1, Supabase, lo que sea).
- Que el usuario indique el alcance de la auditoría (un proyecto, un módulo, o una pregunta puntual) — el skill lo pregunta si no se lo dan.

## Estructura de este repo

```
auditfile-plugin/
├── .claude-plugin/
│   └── marketplace.json      -- catálogo del marketplace (lista este plugin)
└── plugins/
    └── auditfile/
        ├── .claude-plugin/
        │   └── plugin.json   -- identidad del plugin
        └── skills/
            └── auditfile/
                └── SKILL.md  -- el skill en sí
```

## Actualizar el skill

Si se le suman mejoras a `SKILL.md`, subir el cambio a este repo alcanza — quien ya lo instaló actualiza corriendo de nuevo `/plugin install auditfile@auditfile-marketplace`.
