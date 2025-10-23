# GitHub Actions para Documentación

Este directorio contiene los workflows de GitHub Actions para automatizar la generación de documentación.

## Workflows Disponibles

### `generate-pdfs.yml` - Generación Automática de PDFs

Este workflow genera automáticamente archivos PDF a partir de los archivos markdown de documentación.

#### ¿Cuándo se ejecuta?

- **Automáticamente**: Cuando se hace push a las ramas `main` o `master` y se modifican:
  - `ARQUITECTURA.md`
  - `DATABASE-DOMAINS-STRUCTURE.md`
  - `DIAGRAMAS-SIMPLIFICADOS.md`
  - `exports/schema.sql`
  - El propio workflow

- **Manualmente**: Desde la pestaña "Actions" en GitHub usando el botón "Run workflow"

#### ¿Qué hace?

1. Instala las dependencias necesarias (Node.js 20 y md-to-pdf)
2. Genera PDFs con formato A4 y márgenes de 20mm
3. Aplica estilos GitHub Markdown CSS para mejor presentación
4. Guarda los PDFs en la carpeta `exports/`
5. Hace commit automático de los PDFs generados (si hay cambios)

#### Archivos generados

Los siguientes PDFs se generan automáticamente en `exports/`:

- ✅ `ARQUITECTURA.pdf` - Documentación de la arquitectura del sistema
- ✅ `DATABASE-DOMAINS-STRUCTURE.pdf` - Estructura de base de datos por dominios
- ✅ `DIAGRAMAS-SIMPLIFICADOS.pdf` - Diagramas simplificados del sistema

#### Características

- **Formato consistente**: Todos los PDFs usan formato A4 con márgenes uniformes
- **Estilos GitHub**: Aplica estilos de GitHub Markdown para mejor legibilidad
- **Auto-commit**: Los PDFs se actualizan automáticamente en el repositorio
- **Skip CI**: El commit de PDFs incluye `[skip ci]` para evitar loops infinitos
- **Resumen**: Genera un resumen con el tamaño de cada PDF generado

#### Permisos necesarios

El workflow requiere permisos de escritura en el repositorio para hacer commit de los PDFs.

```yaml
permissions:
  contents: write
```

#### Ejecutar manualmente

Para ejecutar el workflow manualmente:

1. Ve a la pestaña "Actions" en GitHub
2. Selecciona "Generate Documentation PDFs"
3. Haz clic en "Run workflow"
4. Selecciona la rama y haz clic en "Run workflow"

#### Troubleshooting

Si el workflow falla, verifica:

1. Que los archivos markdown existen y tienen sintaxis válida
2. Que la carpeta `exports/` existe en el repositorio
3. Que el repositorio tiene permisos de escritura habilitados
4. Los logs del workflow en la pestaña "Actions"
