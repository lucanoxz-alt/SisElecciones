# Stack y Descripción del Proyecto — Sistema de Elecciones UNP

## Base de datos creada en phpMyAdmin

El script crea la base:

```text
elecciones_unp
```

Contiene **27 tablas** en total:

```text
acta_electoral
candidato
cargo_admin
cargo_categoria_permitida
cargo_electoral
cargo_excluido_sorteo
constancia_voto
departamento
docente
facultad
firma_acta
impugnacion_electoral
lista_electoral
log_auditoria
mesa_electoral
mesa_sufragio          ← generada por JPA/Hibernate
miembro_mesa
observacion_electoral
padron_electoral
padron_mesa            ← generada por JPA/Hibernate
parametro_global
personero
proceso_electoral
sesion_activa
tacha
usuario
voto
```

> **Nota:** `mesa_sufragio` y `padron_mesa` son tablas generadas automáticamente por
> JPA/Hibernate del backend Spring Boot. Presentan columnas duplicadas
> (`proceso_id` / `id_proceso` y `docente_id` / `id_docente`) que son un
> artefacto del mapeo ORM y no representan redundancia de diseño.

La importación fue probada directamente contra el MariaDB de XAMPP y se verificó que las tablas fueron creadas correctamente.

Para importar el schema manualmente:

1. Abre phpMyAdmin en `http://localhost:8012/phpmyadmin`.
2. Selecciona **Importar**.
3. Elige `Base de Datos/01_schema.sql`.
4. Presiona **Continuar**.

El script no modifica bases existentes como `gestion_familiar`, `hotel` o `sistemabiblioteca`.

---

## Backend

| Tecnología      | Versión |
|-----------------|---------|
| Java            | 21      |
| Spring Boot     | 3.5     |
| Spring Web      | —       |
| Spring Data JPA | —       |
| Spring Security | —       |
| MariaDB Driver  | —       |
| Maven           | —       |

El endpoint de salud inicial es:

```text
GET http://localhost:8080/api/health
```

Respuesta esperada:

```json
{
  "status": "UP",
  "service": "elecciones-backend"
}
```

Para iniciarlo:

```powershell
cd "C:\Users\lucan\Documents\Lucano\UNP Ciclo VI\Analisis y Diseño de Sistemas II\SisElecciones\Sistema Elecciones UNP\backend"
mvn spring-boot:run
```

---

## Frontend

| Tecnología | Versión |
|------------|---------|
| React      | —       |
| Vite       | —       |
| JavaScript | —       |

Para ejecutarlo:

```powershell
cd "C:\Users\lucan\Documents\Lucano\UNP Ciclo VI\Analisis y Diseño de Sistemas II\SisElecciones\Sistema Elecciones UNP\frontend"
npm install
npm run dev
```

La aplicación estará disponible en:

```text
http://localhost:5173
```

El frontend consulta `/api/health` para verificar que el backend esté activo.

---

## Arquitectura del sistema

```text
React (frontend)
  |
  | HTTP / JSON
  v
Spring Boot REST API (backend)
  |
  | JPA / SQL parametrizado
  v
MariaDB / XAMPP (puerto 8012)
```

### Responsabilidades por capa

| Capa        | Responsabilidad                                              |
|-------------|--------------------------------------------------------------|
| React       | Interfaz y experiencia de usuario                            |
| Spring Boot | Reglas electorales, seguridad, transacciones, auditoría      |
| MariaDB     | Persistencia, integridad referencial, índices, restricciones |

---

## Lógica electoral en Spring Boot (no en procedimientos almacenados)

Se recomienda mantener la lógica en Spring Boot para facilitar pruebas, mantenimiento y trazabilidad. Los casos de uso que debe controlar Spring Boot son:

- Validaciones de elegibilidad
- Registro de candidaturas y tachas
- Sorteos de mesas
- Habilitación del padrón
- Votación anónima (transacción atómica)
- Generación de constancias QR
- Impugnaciones y observaciones
- Generación de actas
- Reglas de segunda vuelta
- Cálculo de resultados

### Ejemplo de transacción atómica para emisión de voto

```java
@Transactional
public ConstanciaVoto registrarVoto(...) {
    // 1. Bloquear y verificar padrón (ya_voto = false)
    // 2. Marcar ya_voto = true
    // 3. Insertar voto anónimo
    // 4. Generar constancia QR
}
```

Consulta SQL con bloqueo optimista para evitar doble voto:

```sql
UPDATE padron_electoral
SET    ya_voto         = TRUE,
       fecha_votacion  = CURRENT_TIMESTAMP
WHERE  id_proceso      = ?
  AND  id_cargo        = ?
  AND  id_docente      = ?
  AND  habilitado_para_votar = TRUE
  AND  ya_voto         = FALSE;
```

Spring Boot verifica que se haya actualizado exactamente **una fila**; si no, revierte la transacción.

---

## Orden de implementación recomendado

1. Configurar conexión y entidades JPA
2. Implementar autenticación y roles (Spring Security + JWT)
3. Implementar módulo de docentes y padrón
4. Implementar procesos y cargos electorales
5. Implementar listas y candidatos
6. Implementar personeros
7. Implementar sorteo de mesas
8. Implementar votación anónima
9. Implementar generación de QR / constancias
10. Implementar actas, impugnaciones y resultados