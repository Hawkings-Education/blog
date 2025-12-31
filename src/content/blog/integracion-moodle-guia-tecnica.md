---
title: 'Integración con Moodle: Guía Técnica Completa'
description: 'Cómo Hawkings se conecta nativamente con tu LMS usando Moodle Web Services REST API para automatizar la evaluación.'
pubDate: 'Dec 30 2024'
heroImage: '../../assets/blog-placeholder-2.jpg'
---

**Cómo Hawkings se conecta nativamente con tu LMS para automatizar la evaluación**

---

## TL;DR

- **No usamos LTI**: Hawkings utiliza la **Moodle Web Services REST API** para una integración más profunda y bidireccional
- **Autenticación por token**: Un único `wstoken` permite acceso seguro a todas las operaciones necesarias
- **Sincronización automática**: Importamos cursos, tareas, entregas y usuarios automáticamente
- **Calificación en tiempo real**: Las notas y retroalimentación generadas por IA se suben directamente al libro de calificaciones de Moodle
- **Detección de duplicados**: Identificamos automáticamente trabajos duplicados entre estudiantes

---

## El problema que resuelve

Las universidades que usan Moodle enfrentan un desafío técnico significativo cuando quieren automatizar la evaluación:

1. **Fragmentación de datos**: Las entregas viven en Moodle, pero la evaluación ocurre en otro sistema
2. **Sincronización manual**: Los profesores deben exportar/importar calificaciones manualmente
3. **Pérdida de contexto**: Al mover datos entre sistemas, se pierde información valiosa como rúbricas y criterios
4. **Retraso en feedback**: El tiempo entre entrega y retroalimentación puede ser de días o semanas

Hawkings resuelve esto conectándose **directamente** a la API de Moodle, eliminando intermediarios y automatizando todo el flujo.

---

## Cómo lo hacemos (explicación accesible)

### 1. Conexión inicial

Cuando configuras Hawkings, solo necesitas dos cosas de tu Moodle:
- **URL base** de tu instalación (ej: `https://campus.universidad.edu`)
- **Token de Web Services** con los permisos adecuados

No hay instalación de plugins, no hay configuración SAML compleja, no hay certificados SSL adicionales. Un token y listo.

### 2. Importación de cursos y tareas

Hawkings puede listar todos los cursos disponibles y sus tareas de entrega (`assignments`). Los administradores eligen qué cursos quieren sincronizar, y el sistema mantiene un mapeo entre las entidades de Moodle y las de Hawkings.

### 3. Sincronización de entregas

Una vez configurado, Hawkings:
1. **Consulta periódicamente** las entregas nuevas
2. **Descarga los archivos** enviados por los estudiantes
3. **Procesa el documento** (PDF, DOCX, etc.) para extracción de texto
4. **Ejecuta la evaluación con IA** usando las rúbricas configuradas
5. **Genera retroalimentación personalizada** para cada estudiante

### 4. Publicación de calificaciones

Cuando la evaluación está lista:
1. **Valida que la entrega no haya sido calificada manualmente** (para no sobrescribir)
2. **Sube la nota numérica** al libro de calificaciones
3. **Incluye retroalimentación detallada** como comentario
4. **Marca la entrega como "graded"** en Moodle

Todo esto ocurre sin intervención humana (en modo automático) o tras validación del profesor (en modo semi-automático).

---

## Bajo el capó (detalles técnicos)

### Arquitectura de integración

La integración se basa en el patrón **Platform Abstraction**:

```
LearningPlatform (Model)
    └── PlatformFactory
        └── Moodle\Manager (implementación específica)
            ├── Course
            ├── Assignment
            ├── Submission
            ├── Grade
            ├── User
            └── File
```

Esta arquitectura permite que Hawkings soporte múltiples LMS (Moodle, Canvas, Google Classroom) con la misma lógica de negocio.

### Autenticación: Moodle Web Services REST API

**No usamos LTI 1.1 ni LTI 1.3**. En su lugar, utilizamos la API REST nativa de Moodle con autenticación por token:

```php
// Request.php - Todas las llamadas pasan por aquí
protected function requestBody(array $params = []): array
{
    return [
        'wsfunction' => $this->wsfunction,
        'moodlewsrestformat' => 'json',
        'wstoken' => $this->row->credentials,  // Token de autenticación
    ] + $params + $this->params;
}
```

El endpoint es siempre `/webservice/rest/server.php` y cada operación tiene su función específica.

### Funciones de Moodle utilizadas

| Función | Propósito |
|---------|-----------|
| `core_webservice_get_site_info` | Verificar conexión y permisos |
| `core_course_get_courses` | Listar cursos disponibles |
| `core_enrol_get_users_courses` | Obtener cursos de un usuario |
| `mod_assign_get_assignments` | Listar tareas de un curso |
| `mod_assign_get_submissions` | Obtener entregas de una tarea |
| `mod_assign_save_grade` | **Subir calificación y feedback** |
| `core_user_get_users_by_field` | Buscar usuarios por email o ID |

### Sincronización de entregas

El proceso de sincronización itera sobre cursos y tareas habilitados. Para cada entrega, el sistema:

1. **Verifica validez**: Solo procesa entregas con status `submitted` y no calificadas
2. **Detecta actualizaciones**: Compara `timemodified` para identificar re-entregas
3. **Gestiona duplicados**: Si el estudiante re-envía, archiva la versión anterior

### Detección de trabajos duplicados

Hawkings calcula un hash del contenido del archivo para detectar plagios entre estudiantes. Si detecta un duplicado, reutiliza la evaluación anterior y marca el trabajo con un comentario indicando la coincidencia.

### Control de publicación

Antes de subir una calificación, el sistema verifica:

- Configuración global habilitada
- Estado correcto de la evaluación IA
- Validación humana si es requerida
- Nota dentro del rango válido
- Feedback suficientemente detallado (mín. 500 caracteres)
- Delay de publicación cumplido
- Que no se haya calificado manualmente en Moodle

---

## Por qué importa

### Para el equipo de IT

- **Sin plugins adicionales**: No requiere instalar nada en Moodle
- **Seguridad controlada**: El token tiene permisos específicos y limitados
- **Trazabilidad completa**: Logs detallados de cada operación
- **Modo debug**: Activable por plataforma para troubleshooting

### Para los profesores

- **Cero trabajo manual**: Las notas aparecen en Moodle automáticamente
- **Feedback inmediato**: Los estudiantes reciben retroalimentación en horas, no días
- **Control cuando lo necesitan**: Pueden revisar antes de publicar si lo prefieren

### Para los estudiantes

- **Misma experiencia**: Todo ocurre dentro de Moodle, su entorno habitual
- **Feedback útil**: Comentarios detallados, no solo una nota

---

## Conclusión

La integración de Hawkings con Moodle no es un parche ni un workaround. Es una conexión nativa que aprovecha la API oficial de Moodle para crear un flujo completamente automatizado de evaluación.

Al usar Web Services REST en lugar de LTI, obtenemos:
- **Mayor profundidad**: Acceso a más datos y operaciones
- **Bidireccionalidad real**: Leemos y escribimos, no solo lanzamos
- **Menor fricción**: Un token vs configuración LTI compleja

El resultado: universidades que evalúan miles de entregas semanales sin que ningún profesor tenga que exportar un CSV.

---

*¿Quieres ver cómo funciona con tu Moodle? Agenda una demo técnica con nuestro equipo.*
