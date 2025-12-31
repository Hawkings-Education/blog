---
title: 'Integración con Canvas: Por Qué No Es Solo "Copiar y Pegar" desde Moodle'
description: 'Las diferencias arquitectónicas entre Canvas y Moodle requieren implementaciones completamente distintas. Aquí explicamos cómo lo resolvemos.'
pubDate: 'Dec 29 2024'
heroImage: '../../assets/blog-placeholder-3.jpg'
---

## TL;DR

Hawkings soporta tanto Moodle como Canvas LMS, pero las diferencias arquitectónicas entre ambas plataformas requieren implementaciones completamente distintas. Canvas usa una API REST moderna con autenticación Bearer token y paginación via headers Link, mientras que Moodle utiliza funciones WebService específicas con tokens en el body. El sistema de entregas y calificaciones también difiere significativamente en estructura y flujo de datos.

---

## El Problema que Resuelve

Las instituciones educativas no son homogéneas. Algunas usan Moodle (especialmente en Europa y Latinoamérica), otras prefieren Canvas (dominante en universidades estadounidenses). Cuando desarrollamos Hawkings Grader, el sistema de corrección automática con IA, necesitábamos que funcionara sin fricción en ambas plataformas.

El reto: Canvas y Moodle tienen filosofías de API completamente diferentes. Lo que funciona en uno, no funciona en el otro.

---

## Cómo Lo Hacemos

### Arquitectura de Abstracción

Utilizamos el patrón Factory para instanciar el manager correcto según el tipo de plataforma:

```php
// PlatformFactory.php
public static function get(Model $row): PlatformAbstract
{
    $class = static::class($row->type);  // 'moodle', 'canvas', etc.
    return new $class($row);
}
```

Ambos managers (`Canvas\Manager` y `Moodle\Manager`) implementan la misma interfaz abstracta `PlatformAbstract`, garantizando que el resto del sistema pueda trabajar con cualquier LMS sin conocer los detalles de implementación.

### Operaciones Soportadas (Interfaz Común)

```php
abstract public function gradeUpload(GradeModel $grade): void;
abstract public function submission(GradeModel $grade): ?SubmissionResource;
abstract public function assignmentsByCourseId(string $course_id): array;
abstract public function coursesAll(): array;
abstract public function fileDownloadByUrl(string $url): string;
```

---

## Bajo el Capó

### 1. Autenticación y Estructura de Requests

**Canvas** utiliza API REST con Bearer token en headers:

```php
// Canvas/Request.php
public function getCurl(?string $url = null): Curl
{
    return new Curl()
        ->setUrl($url ?: $this->getUrl())
        ->setAuthorization($this->row->credentials)  // Bearer token
        ->setJson(true)
        ->setTimeOut(60);
}

protected function getUrl(): string
{
    return $this->row->url.'/api/v1/'.trim($this->path, '/');
}
```

**Moodle** usa funciones WebService con token en el body de un POST:

```php
// Moodle/Request.php
protected function requestBody(array $params = []): array
{
    return [
        'wsfunction' => $this->wsfunction,
        'moodlewsrestformat' => 'json',
        'wstoken' => $this->row->credentials,
    ] + $params;
}
```

### 2. Paginación: Headers vs Sin Paginación

**Canvas** implementa paginación en headers HTTP siguiendo la especificación Link (RFC 5988). El sistema parsea los headers para encontrar la URL de la página siguiente y continúa iterando hasta que no hay más páginas.

**Moodle** devuelve todos los resultados en una sola respuesta (sin paginación nativa para la mayoría de endpoints).

### 3. Rate Limiting Inteligente

Solo Canvas tiene rate limiting explícito. Hawkings respeta el header `x-rate-limit-remaining` y hace una pausa preventiva cuando quedan pocas peticiones disponibles.

### 4. Subida de Calificaciones: Dos Mundos Diferentes

**Canvas** usa un PUT RESTful al endpoint de submissions con una estructura JSON clara.

**Moodle** requiere llamar a la función `mod_assign_save_grade` con parámetros específicos incluyendo `plugindata` para el feedback.

### Tabla Comparativa

| Aspecto | Canvas | Moodle |
|---------|--------|--------|
| **API** | REST `/api/v1/*` | WebService RPC |
| **Auth** | Bearer token en header | Token en body POST |
| **Paginación** | Headers Link RFC 5988 | Sin paginación |
| **Rate limit** | `x-rate-limit-remaining` | Implícito |
| **Calificación** | PUT a `/submissions/{id}` | `mod_assign_save_grade` |
| **Archivos** | Array plano `attachments[]` | Anidado en `plugins[].fileareas[]` |
| **ID Assignment** | Solo `assignment_id` | `assignment_id` + `cmid` |

---

## Por Qué Importa

### Para el equipo de IT

- **Una sola integración, múltiples LMS**: El mismo Hawkings funciona con Canvas, Moodle y (próximamente) más
- **Sin código duplicado**: La abstracción permite mantener una sola base de código
- **Escalabilidad real**: Añadir un nuevo LMS es implementar una interfaz, no reescribir el sistema

### Para los directivos

- **Sin vendor lock-in en el LMS**: Puedes cambiar de Canvas a Moodle (o viceversa) sin perder la automatización
- **Mismo resultado, diferente plataforma**: Los profesores y alumnos tienen la misma experiencia independientemente del LMS

### Para el día a día

- **Las diferencias son invisibles**: El profesor no sabe (ni necesita saber) que Canvas y Moodle funcionan diferente
- **Los bugs se aíslan**: Un problema en la integración Canvas no afecta a Moodle

---

## Conclusión

Soportar múltiples LMS no es trivial. Canvas y Moodle son productos diferentes con filosofías diferentes, y pretender que "es solo cambiar unas URLs" es simplificar demasiado.

Hawkings invierte en abstracciones robustas porque sabemos que las universidades no eligen su LMS pensando en nosotros. Nosotros nos adaptamos a ellas, no al revés.

El resultado: una experiencia consistente de evaluación automática, ya uses Canvas, Moodle, o lo que venga después.

---

*¿Tu universidad usa un LMS diferente? Cuéntanos cuál y exploramos la integración.*
