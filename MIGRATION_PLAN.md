# Plan de Migración — Brigesnote a PHP 8.3

**Proyecto:** Bridges Note — MINEDU ESCALE Bridge API
**Repositorio base:** https://github.com/hubelsolis/brigesnote
**Fork de trabajo:** https://github.com/ANDREA200-CF/brigesnote
**Ramas de migración:** `migracion-php83` (etapa 1) y `migracion-laravel13` (etapa 2)
**Responsable:** Élida Andrea Campos Flores

> **Nota sobre el historial:** este plan se redactó al inicio de la tarea, pero por un error al ejecutar `git add` no quedó incluido en el commit del 12/11 ("iniciando migracion a 8.3"). Se versionó después, junto con los resultados de cada fase.

---

## 1. Objetivo

Migrar el proyecto Brigesnote para que funcione sobre **PHP 8.3**, documentando cada fase y dejando el sistema probado en un entorno local (Laragon) antes de cualquier despliegue a producción.

## 2. Situación inicial

| Elemento | Estado inicial | Estado final |
|---|---|---|
| Framework | Laravel 7.x | **Laravel 13.42.0** |
| PHP soportado | 7.4 / 8.0 | **8.3 – 8.5** |
| PHPUnit | 8.5 / 9.x | 12.5 |
| Avisos de seguridad (`composer audit`) | — | **Ninguno** |
| Entorno local | Laragon con PHP 8.3.36 | Laragon con PHP 8.3.36 |

**Problema detectado:** al ejecutar `composer install` sobre PHP 8.3, Laravel 7 falla porque sus clases no cumplen las firmas tipadas que PHP 8.1+ exige en interfaces nativas como `ArrayAccess`. Laravel 7 no tiene versiones compatibles con PHP 8.1 o superior, por lo que la migración de PHP obliga a actualizar el framework.

## 3. Decisiones técnicas

### 3.1 Versión de destino: Laravel 13, en dos etapas

| Versión | PHP | Soporte de seguridad | Comentario |
|---|---|---|---|
| Laravel 10 | 8.1 – 8.3 | Finalizado | Etapa intermedia: menor salto desde Laravel 7. |
| Laravel 12 | 8.2 – 8.4 | Vigente | No exige PHP 8.3. |
| **Laravel 13** (marzo 2026) | 8.3 – 8.5 | Hasta el primer trimestre de 2028 | **Destino final**: exige PHP 8.3, el mismo objetivo de esta migración, y tiene soporte vigente. |

La migración se hizo en **dos etapas** para reducir el riesgo:

1. **Etapa 1 — Laravel 7 → 10:** concentra los cambios de código que rompían la aplicación (paquetes eliminados, middleware renombrado, PHPUnit). Al terminarla, la aplicación ya funcionaba sobre PHP 8.3 y quedó guardada en `main` como punto de respaldo.
2. **Etapa 2 — Laravel 10 → 13:** recupera el soporte de seguridad. Laravel 10 ya no recibe parches y todas sus versiones tienen avisos publicados, por lo que en la etapa 1 fue necesario desactivar temporalmente el bloqueo de avisos de Composer (`policy.advisories.block: false`). En la etapa 2 ese bloqueo se reactivó.

### 3.2 Se conserva la estructura de la aplicación
A partir de Laravel 11 los proyectos **nuevos** usan una estructura simplificada (sin `app/Http/Kernel.php`, configuración en `bootstrap/app.php`). La guía oficial de actualización recomienda **no** migrar la estructura en proyectos existentes, porque Laravel 11+ sigue soportando la estructura de Laravel 10. Por eso se conservaron `Kernel.php`, `RouteServiceProvider` y los proveedores existentes.

---

## 4. Fases de la migración

### Fase 0 — Preparación ✅
- Instalación de Laragon con PHP 8.3.36.
- Fork del repositorio original en la cuenta personal de GitHub para conservar la autoría y no modificar el repositorio del docente.
- Clonado en `C:\laragon\www\brigesnote` y configuración de credenciales de Git.
- Trabajo en ramas separadas (`migracion-php83`, `migracion-laravel13`) para aislar los cambios de `main`.

### Fase 1 — Auditoría de compatibilidad ✅
- Intento de `composer install` sobre PHP 8.3 → falla por incompatibilidad de Laravel 7 con PHP 8.1+.
- Revisión del código de la aplicación (`app/`, `config/`, `routes/`, `database/`, `tests/`) en busca de APIs eliminadas entre Laravel 7 y 13.
- Hallazgos que rompen la ejecución:
  - `app/Http/Middleware/TrustProxies.php` depende de `fideloper/proxy`, paquete eliminado.
  - `app/Http/Kernel.php` usa `\Fruitcake\Cors\HandleCors`, paquete eliminado.
  - `app/Http/Middleware/CheckForMaintenanceMode.php` hereda de una clase eliminada en Laravel 9.
  - `phpunit.xml` usa la sección `<filter><whitelist>`, no soportada desde PHPUnit 10.
  - `app/Http/Middleware/VerifyCsrfToken.php` hereda de `VerifyCsrfToken`, renombrado a `PreventRequestForgery` en Laravel 13.
- Cambios de Laravel 11–13 revisados que **no** afectan al proyecto: modificación de columnas con `change()`, Carbon 3, caché de objetos (`serializable_classes`), restricciones en `boot()` de modelos y anotaciones de PHPUnit (`@test`).

### Fase 2 — Etapa 1: dependencias Laravel 7 → 10 ✅
Commit `e35f790`

| Paquete | Antes | Después |
|---|---|---|
| `php` | `^7.2.5\|^8.0` | `^8.2` |
| `laravel/framework` | `^7.29` | `^10.0` (instalado 10.50.4) |
| `nunomaduro/collision` | `^4.3` | `^7.0` |
| `phpunit/phpunit` | `^8.5.8\|^9.3.3` | `^10.0` |

Paquetes eliminados por quedar integrados en el framework o reemplazados: `fideloper/proxy`, `fruitcake/laravel-cors`, `facade/ignition`, `swiftmailer/swiftmailer`, `opis/closure`.

### Fase 3 — Etapa 1: adaptación del código ✅
Commit `5ed9cd9`

| Archivo | Cambio |
|---|---|
| `app/Http/Middleware/TrustProxies.php` | Ahora extiende `Illuminate\Http\Middleware\TrustProxies` y usa las cabeceras `X-Forwarded-*` individuales (`HEADER_X_FORWARDED_ALL` ya no existe). |
| `app/Http/Kernel.php` | `\Fruitcake\Cors\HandleCors` → `\Illuminate\Http\Middleware\HandleCors`; `CheckForMaintenanceMode` → `PreventRequestsDuringMaintenance`; `$routeMiddleware` → `$middlewareAliases`. |
| `app/Http/Middleware/PreventRequestsDuringMaintenance.php` | Creado (reemplaza a `CheckForMaintenanceMode.php`, eliminado). |
| `phpunit.xml` | `<filter><whitelist>` → `<source><include>` (formato de PHPUnit 10+). |

### Fase 4 — Etapa 2: Laravel 10 → 13 ✅
Rama `migracion-laravel13`

| Paquete | Antes | Después |
|---|---|---|
| `php` | `^8.2` | `^8.3` |
| `laravel/framework` | `^10.0` | `^13.0` (instalado 13.42.0) |
| `laravel/tinker` | `^2.8` | `^3.0` |
| `guzzlehttp/guzzle` | `^7.2` | `^7.8` |
| `nunomaduro/collision` | `^7.0` | `^8.6` (instalado 8.9.6) |
| `phpunit/phpunit` | `^10.0` | `^12.0` (instalado 12.5.44) |
| `fakerphp/faker` / `mockery/mockery` | `^1.9.1` / `^1.4.4` | `^1.23` / `^1.6` |

Otros cambios:
- Se eliminó `policy.advisories.block: false` de `composer.json`: Composer vuelve a bloquear paquetes con avisos de seguridad.
- `app/Http/Middleware/VerifyCsrfToken.php` ahora extiende `Illuminate\Foundation\Http\Middleware\PreventRequestForgery` (nuevo nombre del middleware CSRF en Laravel 13).

Comandos:
```
composer update -W
composer dump-autoload
```

### Fase 5 — Pruebas en entorno local ✅
Entorno: Laragon (Apache + MySQL) con PHP 8.3.36, accediendo por `http://brigesnote.test`.

Configuración local necesaria (no se versiona, `.env` está en `.gitignore`):
```
php artisan key:generate
php artisan config:clear
php artisan migrate
php artisan sync:locations
```

| Prueba | Laravel 10 (etapa 1) | Laravel 13 (etapa 2) |
|---|---|---|
| `php -v` | PHP 8.3.36 ✅ | PHP 8.3.36 ✅ |
| `php artisan --version` | 10.50.4 ✅ | 13.42.0 ✅ |
| `composer dump-autoload` + `package:discover` | 6082 clases ✅ | 6845 clases ✅ |
| `composer audit` | 8 avisos (bloqueo desactivado) ⚠️ | Sin vulnerabilidades ✅ |
| `php artisan migrate` / `migrate:status` (7 migraciones) | Ejecutadas ✅ | Reconocidas ✅ |
| `php artisan test` | 2/2 aprobados ✅ | 2/2 aprobados ✅ |
| `php artisan sync:locations` (API MINEDU) | Funciona; ejecución parcial (2 regiones: 01 Amazonas, 02 Áncash) ✅ | — |
| Página de inicio (`http://brigesnote.test`) | ✅ | ✅ |
| `GET /api/regiones` | Devuelve regiones (01 Amazonas, 02 Áncash, …) ✅ | ✅ |
| `/api/provincias/{id}`, `/api/distritos/{id}`, `/api/colegios/{ubigeo}` | ⏳ Pendiente | ⏳ Pendiente |

> **Notas:**
> - `php artisan serve` no funciona en este proyecto porque no tiene carpeta `public/` (ver sección 6). Las pruebas se hicieron con el virtual host de Laragon.
> - En Windows, `composer update` puede tardar varios minutos en "Generating optimized autoload files". Si se interrumpe, el autoload queda desactualizado y aparecen errores de clases no encontradas; se corrige con `composer dump-autoload`.

### Fase 6 — Entrega y despliegue
- [x] Actualizar `README.md` con los requisitos y el proceso de migración.
- [x] Unir las ramas de migración a `main` en el fork.
- [ ] Respaldo de la base de datos de producción antes de desplegar.
- [ ] Despliegue en un servidor con PHP 8.3 y verificación de los endpoints.

---

## 5. Procedimiento de reversión

Si la migración presenta fallos en producción:
1. **Volver a Laravel 10:** restaurar el commit `cba2116` (fin de la etapa 1), que funciona sobre PHP 8.3, y ejecutar `composer install`.
2. **Volver a Laravel 7:** restaurar el commit `e947779` (estado previo a la migración) y ejecutar el servidor con PHP 7.4.
3. En ambos casos, restaurar el respaldo de la base de datos si hubo cambios.

---

## 6. Hallazgos, riesgos y deuda técnica

| # | Hallazgo | Riesgo | Recomendación |
|---|---|---|---|
| 1 | El proyecto no tiene carpeta `public/`: `index.php`, `.htaccess` y `favicon.ico` están en la raíz, así que la raíz entera es accesible desde la web. | **Alto.** Archivos como `.env` (credenciales de base de datos) podrían descargarse desde el navegador. | Volver a la estructura estándar de Laravel con `public/` como raíz del servidor web, o al menos bloquear archivos ocultos en `.htaccess`. |
| 2 | `server.php` apunta a `public/index.php`, que no existe. Laravel 10+ ya no usa este archivo. | Bajo. | Eliminarlo. |
| 3 | `app/Http/Kernel.php` declara `$commands` con `SyncLocations`, pero ese arreglo no tiene efecto en el Kernel HTTP. El comando funciona porque `app/Console/Kernel.php` carga la carpeta `Commands`. | Bajo (código muerto). | Eliminar la propiedad del Kernel HTTP. |
| 4 | `database/factories/UserFactory.php` usa el formato de factories de Laravel 7 (`$factory->define`), que ya no existe. | Bajo (ningún seeder lo usa). | Reescribirlo como clase `Database\Factories\UserFactory`. |
| 5 | Existen dos modelos `Region`: `app/Region.php` y `app/Models/Region.php`. | Bajo (confusión). | Identificar cuál se usa y eliminar el otro. |
| 6 | La carpeta `database/seeds` usa el nombre de Laravel 7 (en Laravel 8+ es `database/seeders` con namespace). | Bajo (funciona vía `classmap`). | Renombrar y agregar namespace al modernizar. |

---

## 7. Trabajo futuro

1. Reestructurar el proyecto con carpeta `public/` (hallazgo #1).
2. Ampliar las pruebas automatizadas: los tests actuales son los de ejemplo y no cubren `LocationController` ni `SyncLocations`.
3. Resolver la deuda técnica menor (hallazgos #2 a #6).
4. Planificar la actualización a Laravel 14 antes de que termine el soporte de seguridad de Laravel 13 (primer trimestre de 2028).
