# Vida Cotidiana - Contexto

## Purpose & Context

Kelsie tiene proyectos e intereses independientes que abarcan gaming, cocina, finanzas personales y RPGs de mesa. Los objetivos generalmente se centran en resolución práctica de problemas y desarrollo de habilidades. Algunas conversaciones se conducen en español.

---

## Estado Actual

### Android - Transferencia de save files
- Intentó copiar save files de un juego Ren'Py (`com.akaime.airevolution`) en Samsung Galaxy Tab S6 Lite a PC
- Métodos agotados: MTP, ADB, Shizuku + ZArchiver — todos fallaron por restricciones de scoped storage de Android
- **Opciones restantes**: feature de backup/export in-game o Solid Explorer

### D&D 5e - Personaje activo
- **Bárbaro nivel 8** (PHB 2014 base, con homebrew aprobado por DM y elementos de edición 2024)
- **2 flags sin resolver** de la última auditoría:
  1. Rage damage bonus mostrando +3 (correcto solo en nivel 9)
  2. Daño base 2d10 del Old Knight Sword pendiente de confirmación DM

### Crédito (Colombia) - Nu Colombia
- Tarjeta de crédito Nu Colombia recientemente aprobada
- Contexto: deuda pagada que dejó reporte negativo en Datacrédito hasta octubre 2028
- Trabajando en uso responsable para construir historial crediticio
- Meta: acceder a tarjetas, financiamiento vehicular y productos de crédito a largo plazo

---

## On the Horizon

- Decidir si intentar **Solid Explorer** como último recurso para transferencia de save files Android
- Resolver los 2 flags del D&D con el DM (Rage damage bonus, Old Knight Sword base damage)
- Monitorear **utilización de crédito Nu** y elegibilidad para aumento de cupo a los 6 meses; tarjeta respaldada por CDT como opción futura cuando haya capital
- Cambio de política de **GitHub Copilot** requiere decisión (flaggeado en auditoría de email)
- **Pull request abierto** en AZCKeeper flaggeado como inactivo
- **2 cursos de project management abandonados** — patrón recurrente de iniciar pero no completar capacitación formal

---

## Key Learnings & Principios

### Android / Tech
- Android 11+ scoped storage bloquea acceso a archivos incluso con Shizuku para APKs no-debuggable
- Shizuku otorga listado de directorio pero no acceso al contenido de archivos en este contexto

### Cocina
- Para seguridad con pescado para sushi: la certificación sashimi-grade y protocolos de congelación importan
- Sin ellos, la preparación cocida es el camino más seguro

### Finanzas (Colombia)
- Para recuperación crediticia: utilización baja consistente (10-30%) y pago total del saldo son las palancas principales
- La tarjeta Nu sola es suficiente para las metas actuales de construcción de crédito
- No perseguir CDT cuando el capital y entendimiento no están ahí

### D&D
- PHB 2014 es la baseline
- Aprobación del DM es requerida y suficiente para anular restricciones de edición o homebrew
- Debe ser explícitamente confirmada antes de aceptar mecánicas no-estándar
- Claude debe flaggear discrepancias proactivamente contra la baseline establecida

---

## Approach & Patterns

- Prefiere troubleshooting sistemático paso a paso, trabajando opciones secuencialmente
- Valora evaluación honesta de situaciones (ej. aceptar pescado cocido tras flag de seguridad; no perseguir CDT sin capital)
- En contextos D&D, espera que Claude flaggee discrepancias proactivamente contra el ruleset base
- Se engancha con outputs interactivos o visuales para información compleja (ej. radar dashboard de Gmail)
- Conduce algunas conversaciones en español; Claude debe matchear el idioma

---

## Tools & Recursos

| Categoría | Herramientas |
|-----------|-------------|
| Dispositivos | Samsung Galaxy Tab S6 Lite (Android), PC (username: koichi) |
| Android tools | ADB, Shizuku, ZArchiver, Solid Explorer (pendiente) |
| Gmail | Conector Gmail para análisis de inbox con filtrado progresivo |
| Finanzas Colombia | Nu Colombia, Nequi, Dale, Daviplata, Davivienda Credi Express, Icetex Centro |
| Crédito | Datacrédito (reportes crediticios) |
| D&D | Player's Handbook 2014 (referencia primaria), homebrew + elementos 2024 catalogados por personaje |
