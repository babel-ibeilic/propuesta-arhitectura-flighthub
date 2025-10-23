# FlightHub - Documentación

Documentación técnica completa del sistema FlightHub basado en arquitectura DDD (Domain-Driven Design).

## 📚 Documentos Principales

### 1. [ARQUITECTURA.md](ARQUITECTURA.md)
Documentación completa de la arquitectura del sistema FlightHub, incluyendo:
- Visión general del sistema
- Componentes principales
- Patrones de diseño utilizados
- Flujos de datos
- Integraciones externas

**PDF**: [exports/ARQUITECTURA.pdf](exports/ARQUITECTURA.pdf)

### 2. [DATABASE-DOMAINS-STRUCTURE.md](DATABASE-DOMAINS-STRUCTURE.md)
Estructura de base de datos organizada por dominios DDD:
- 13 dominios independientes
- 26 tablas organizadas
- Esquemas SQL completos
- Índices y relaciones
- Estrategia de migración

**PDF**: [exports/DATABASE-DOMAINS-STRUCTURE.pdf](exports/DATABASE-DOMAINS-STRUCTURE.pdf)

### 3. [DIAGRAMAS-SIMPLIFICADOS.md](DIAGRAMAS-SIMPLIFICADOS.md)
Diagramas visuales simplificados del sistema:
- Arquitectura general
- Flujos de mensajes
- Dominios de negocio
- Diagramas de secuencia

**PDF**: [exports/DIAGRAMAS-SIMPLIFICADOS.pdf](exports/DIAGRAMAS-SIMPLIFICADOS.pdf)

### 4. [exports/schema.sql](exports/schema.sql)
Schema SQL completo listo para importar:
- 26 tablas con prefijo `fh_*`
- Índices optimizados
- Comentarios detallados
- Campos de auditoría

## 📦 Carpeta Exports

La carpeta `exports/` contiene:
- ✅ PDFs generados automáticamente de los documentos principales
- ✅ Schema SQL exportado
- ✅ Archivos listos para distribución

## 🤖 Generación Automática de PDFs

Los PDFs se generan **automáticamente** mediante GitHub Actions cuando:
- Se hace push a las ramas `main` o `master`
- Se modifican los archivos de documentación markdown
- Se ejecuta manualmente el workflow

### Ejecutar manualmente en GitHub

1. Ve a la pestaña **Actions** en GitHub
2. Selecciona **"Generate Documentation PDFs"**
3. Haz clic en **"Run workflow"**
4. Los PDFs se actualizarán automáticamente

## 📁 Estructura del Proyecto

```
docs/
├── .github/
│   ├── workflows/
│   │   └── generate-pdfs.yml       # GitHub Action para PDFs
│   └── README.md                   # Documentación de workflows
├── exports/
│   ├── ARQUITECTURA.pdf            # ✅ Auto-generado
│   ├── DATABASE-DOMAINS-STRUCTURE.pdf  # ✅ Auto-generado
│   ├── DIAGRAMAS-SIMPLIFICADOS.pdf # ✅ Auto-generado
│   └── schema.sql                  # Schema SQL exportado
├── diagrams/                       # Diagramas fuente
├── images/                         # Imágenes de documentación
├── ARQUITECTURA.md                 # 📄 Fuente
├── DATABASE-DOMAINS-STRUCTURE.md   # 📄 Fuente
├── DIAGRAMAS-SIMPLIFICADOS.md      # 📄 Fuente
└── README.md                       # Este archivo
```

## 🔧 Dominios del Sistema

El sistema está organizado en **13 dominios independientes**:

1. **🎯 Flight Orchestrator** - Identificación única y ciclo de vida
2. **📍 Resources** - Recursos aeroportuarios
3. **⏰ Timeline** - Tiempos operacionales
4. **⚠️ Delays** - Gestión de retrasos
5. **👨‍✈️ Crew** - Tripulación
6. **🚨 Alerts** - Alarmas y desvíos
7. **👥 Passengers** - Pasajeros y check-in
8. **🎒 Baggage** - Equipaje y carga
9. **⛽ Fuel** - Combustible
10. **✈️ Aircraft** - Aeronaves
11. **📅 Schedules** - Programación
12. **🔄 Onward Flights** - Vuelos de continuación
13. **🤝 Codeshare** - Vuelos compartidos

## 🚀 Principios de Diseño

### Domain-Driven Design (DDD)
- ✅ Dominios independientes
- ✅ Bounded contexts claros
- ✅ Sin JOINs entre dominios
- ✅ Escalabilidad por dominio

### Identificación Única
- **FUID** (ULID): Identificador interno inmutable
- **6 campos**: Identificación externa estándar
  - operation_date
  - flight_designator
  - operational_suffix
  - airline_designator
  - departure_airport
  - departure_number

### Auditoría
Todas las tablas incluyen:
- `created_at` / `created_by`
- `updated_at` / `updated_by`

## 📊 Estadísticas

- **26 tablas** organizadas en 13 dominios
- **~350+ campos** distribuidos por responsabilidad
- **0 JOINs** necesarios entre dominios
- **6 campos** de identificación replicados

## 🔄 Workflow de Actualización

1. **Editar** archivos markdown (`*.md`)
2. **Commit & Push** a GitHub
3. **GitHub Actions** genera PDFs automáticamente
4. **PDFs actualizados** disponibles en `exports/`

## 📝 Contribuir

Para contribuir a la documentación:

1. Edita los archivos markdown fuente
2. Crea un pull request
3. Los PDFs se generarán automáticamente al merge

## 🛠️ Herramientas

- **md-to-pdf**: Conversión de Markdown a PDF
- **GitHub Actions**: Automatización CI/CD
- **GitHub Markdown CSS**: Estilos consistentes

## 📞 Soporte

Para más información sobre el sistema FlightHub, consulta los documentos principales o contacta al equipo de desarrollo.

---

**Última actualización**: Automático mediante GitHub Actions
**Versión**: 2.0
**Mantenido por**: Equipo FlightHub
