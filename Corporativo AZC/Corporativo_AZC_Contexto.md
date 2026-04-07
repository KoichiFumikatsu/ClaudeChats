# Corporativo AZC - Contexto del Proyecto

## Propósito

Kelsie es desarrolladora de software y líder del equipo de TI en **AZC Legal**, una firma legal colombiana, reportando directamente al gerente general. Su trabajo abarca desarrollo hands-on, modernización de sistemas y liderazgo de equipo — incluyendo contratación, onboarding e impulso de la adopción de flujos de trabajo integrados con IA.

Un valor clave de Kelsie es conectar la ejecución técnica con la comunicación organizacional, especialmente al presentar a un gerente que prefiere outputs visuales y diseñados sobre texto estructurado.

## Estado Actual

### AZCKeeper (Keeper AZC)
- Aplicación de monitoreo de empleados para Windows
- **Stack:** C# .NET 8 (cliente), PHP/MySQL (backend), panel web admin (`keep.azclegal.com`)
- **Roadmap v2 corregido** con gap analysis real:
  - Screenshots (no existe módulo, endpoint ni tabla DB)
  - Notificaciones de alerta salientes (EventIngest recibe pero no despacha)
  - Reglas de alerta configurables (no existe tabla ni UI)
- Roadmap entregado en 3 formatos (PNG, PDF, HTML) con paleta corporativa: navy `#003A5D`, rojo `#BE1622`

### Equipo
- Nueva contratación técnica seleccionada, en proceso de onboarding vía HR
- Email de onboarding formalizado: IA como norma del equipo
- Responsabilidades del nuevo hire: soporte técnico, QA de proyectos IA, investigación de herramientas, desarrollo supervisado

### Modernización
- Iniciativa en curso hacia plataformas web y flujos de trabajo integrados con IA

## Próximos Pasos

- Ejecutar entregables del roadmap AZCKeeper (screenshots, alertas, reglas configurables)
- Integrar al nuevo hire y establecer adopción de IA como práctica estándar
- Continuar modernización de plataforma web

## Principios Clave

- Planificación basada en arquitectura real (validar contra DB schema, módulos C#, endpoints PHP)
- Outputs visuales (PDF, PNG, HTML) para comunicación con gerencia
- IA como herramienta de QA y validación, no reemplazo (reduce resistencia a la adopción)
- Contratar por fortaleza técnica cuando el líder cubre gestión y análisis

## Patrones de Trabajo

- Iterativo: contexto → borrador → validación contra ground truth → corrección → entrega final
- Entregables con fechas específicas, no timelines abstractos
- Comunicación bilingüe (español para interno/equipo, inglés según contexto)
- Claude agent en VS Code como parte del flujo de desarrollo
- Múltiples formatos de output según audiencia (técnica vs. gerencial)

## Herramientas

| Área | Stack |
|------|-------|
| Desarrollo | C# .NET 8, PHP, MySQL, VS Code + Claude agent |
| Infraestructura | Windows desktop (cliente), web admin `keep.azclegal.com` |
| Documentación | reportlab (PDF), HTML, PNG exports |
| Comunicación | HR para contrataciones |
