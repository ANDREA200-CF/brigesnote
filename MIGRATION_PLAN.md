# Plan de Migración — Brigesnote a PHP 8.3

**Proyecto:** Bridges Note — MINEDU ESCALE Bridge API
**Repositorio base:** https://github.com/hubelsolis/brigesnote
**Fork de trabajo:** https://github.com/ANDREA200-CF/brigesnote
**Rama de migración:** `migracion-php83`
**Responsable:** Élida Andrea Campos Flores

---

## 1. Objetivo

Migrar el proyecto Brigesnote para que funcione sobre **PHP 8.3**, documentando cada fase y dejando el sistema probado en un entorno local (Laragon) antes de cualquier despliegue a producción.

## 2. Situación inicial

| Elemento | Estado inicial |
|---|---|
| Framework | Laravel 7.x |
| PHP soportado | 7.4 / 8.0 |
| PHP objetivo | 8.3 |
| Entorno local | Laragon con PHP 8.3.36 |

**Problema detectado:** al ejecutar `composer install` sobre PHP 8.3, Laravel 7 falla porque sus clases no cumplen las firmas tipadas que PHP 8.1+ exige en interfaces nativas como `ArrayAccess`. Laravel 7 no tiene versiones compatibles con PHP 8.1 o superior, por lo que la migración de PHP obliga a actualizar el framework.

## 3. Decisiones técnicas

### 3.1 Versión de destino: Laravel 10
Se eligió **Laravel 10** por ser la versión compatible con PHP 8.3 que implica menos cambios estructurales respecto a Laravel 7. Laravel 11 y 12 cambian la estructura base del proyecto (eliminan `app/Http/Kernel.php`, unifican la configuración en `bootstrap/app.php`), lo que aumentaría el riesgo y el alcance de esta migración.

### 3.2 Avisos de seguridad de Composer
Laravel 10 ya no recibe parches de seguridad y todas sus versiones tienen avisos publicados en Packagist. Composer bloquea por defecto la instalación de paquetes con avisos, por lo que se desactivó temporalmente ese bloqueo en `composer.json`:

```json
"config": {
    "policy": {
        "advisories": {
            "block": false
        }
    }
}
```

Esta es una **decisión consciente y temporal**. Se recomienda una fase posterior de migración a **Laravel 12**, que tiene soporte de seguridad vigente (ver sección 7).

---

## 4. Fases de la migración

### Fase 0 — Preparación ✅
- Instalación de Laragon con PHP 8.3.36.
- Fork del repositorio original en la cuenta personal de GitHub para conservar la autoría y evitar problemas de permisos.
- Clonado en `C:\laragon\www\brigesnote` y configuración de credenciales de Git.
- Creación de la rama `migracion-php83` para aislar los cambios de `main`.

### Fase 1 — Auditoría de compatibilidad ✅
- Intento de `composer install` sobre PHP 8.3 → falla por incompatibilidad de Laravel 7 con PHP 8.1+.
- Revisión del código de la aplicación (`app/`, `config/`, `routes/`, `database/`, `tests/`) en busca de APIs eliminadas entre Laravel 7 y 10.
- Hallazgos que rompen la ejecución:
  - `app/Http/Middleware/TrustProxies.php` depende de `fideloper/proxy`, paquete eliminado.
  - `app/Http/Kernel.php` usa `\Fruitcake\Cors\HandleCors`, paquete eliminado.
  - `app/Http/Middleware/CheckForMaintenanceMode.php` hereda de una clase eliminada en Laravel 9.
  - `phpunit.xml` usa la sección `<filter><whitelist>`, no soportada por PHPUnit 10.

### Fase 2 — Actualización de dependencias ✅
Commit `e35f790`

Cambios en `composer.json`:

| Paquete | Antes | Después |
|---|---|---|
| `php` | `^7.2.5\|^8.0` | `^8.2` |
| `laravel/framework` | `^7.29` | `^10.0` (instalado 10.50.4) |
| `nunomaduro/collision` | `^4.3` | `^7.0` |
| `phpunit/phpunit` | `^8.5.8\|^9.3.3` | `^10.0` |

Paquetes eliminados por quedar integrados en el framework o reemplazados: `fideloper/proxy`, `fruitcake/laravel-cors`, `facade/ignition`, `swiftmailer/swiftmailer`, `opis/closure`.

### Fase 3 — Adaptación del código ✅
Commit `5ed9cd9`

| Archivo | Cambio |
|---|---|
| `app/Http/Middleware/TrustProxies.php` | Ahora extiende `Illuminate\Http\Middleware\TrustProxies` y usa las cabeceras `X-Forwarded-*` individuales (`HEADER_X_FORWARDED_ALL` ya no existe). |
| `app/Http/Kernel.php` | `\Fruitcake\Cors\HandleCors` → `\Illuminate\Http\Middleware\HandleCors`; `CheckForMaintenanceMode` → `PreventRequestsDuringMaintenance`; `$routeMiddleware` → `$middlewareAliases`. |
| `app/Http/Middleware/PreventRequestsDuringMaintenance.php` | Creado (reemplaza a `CheckForMaintenanceMode.php`, eliminado). |
| `phpunit.xml` | `<filter><whitelist>` → `<source><include>` (formato de PHPUnit 10). |

### Fase 4 — Pruebas en entorno local ✅
Entorno: Laragon (Apache + MySQL) con PHP 8.3.36, accediendo por `http://brigesnote.test`.

Configuración local necesaria (no se versiona, `.env` está en `.gitignore`):
```
php artisan key:generate
php artisan config:clear
php artisan migrate
```

| Prueba | Resultado |
|---|---|
| `php -v` | PHP 8.3.36 ✅ |
| `php artisan --version` | Laravel Framework 10.50.4 ✅ |
| `composer dump-autoload` + `package:discover` | Sin errores, 6082 clases ✅ |
| Página de inicio (`http://brigesnote.test`) | Carga correctamente ✅ |
| `php artisan migrate` (7 migraciones: users, password_resets, failed_jobs, regions, provinces, districts, schools) | Todas ejecutadas ✅ |
| `php artisan test` | 2 tests, 2 aprobados ✅ |
| `php artisan sync:locations` (sincronización con API MINEDU) | ⏳ Pendiente |
| Endpoints `/api/regiones`, `/api/provincias/{id}`, `/api/distritos/{id}`, `/api/colegios/{ubigeo}` | ⏳ Pendiente |

> **Nota:** `php artisan serve` no funciona en este proyecto porque no tiene carpeta `public/` (ver sección 6). Las pruebas se hicieron con el virtual host de Laragon.

### Fase 5 — Entrega y despliegue ⏳
- [ ] Completar las pruebas pendientes de la Fase 4.
- [ ] Actualizar la sección de requisitos del `README.md` (todavía indica PHP 7.4 y Laravel 7).
- [ ] Abrir Pull Request de `migracion-php83` hacia `main` en el fork.
- [ ] Respaldo de la base de datos de producción antes de desplegar.
- [ ] Despliegue en un servidor con PHP 8.3 y verificación de los endpoints.

---

## 5. Procedimiento de reversión

Si la migración presenta fallos en producción:
1. Volver a la rama `main`, que conserva el código en Laravel 7.
2. Restaurar el respaldo de la base de datos.
3. Ejecutar el servidor con PHP 7.4 hasta corregir el problema en la rama `migracion-php83`.

---

## 6. Hallazgos, riesgos y deuda técnica

| # | Hallazgo | Riesgo | Recomendación |
|---|---|---|---|
| 1 | El proyecto no tiene carpeta `public/`: `index.php`, `.htaccess` y `favicon.ico` están en la raíz, así que la raíz entera es accesible desde la web. | **Alto.** Archivos como `.env` (credenciales de base de datos) podrían descargarse desde el navegador. | Volver a la estructura estándar de Laravel con `public/` como raíz del servidor web, o al menos bloquear archivos ocultos en `.htaccess`. |
| 2 | Laravel 10 está fuera de soporte de seguridad y se desactivó el bloqueo de avisos de Composer. | Medio. | Migrar a Laravel 12 en una fase posterior y reactivar el bloqueo. |
| 3 | `server.php` apunta a `public/index.php`, que no existe. Laravel 10 ya no usa este archivo. | Bajo. | Eliminarlo. |
| 4 | `app/Http/Kernel.php` declara `$commands` con `SyncLocations`, pero ese arreglo no tiene efecto en el Kernel HTTP. El comando funciona porque `app/Console/Kernel.php` carga la carpeta `Commands`. | Bajo (código muerto). | Eliminar la propiedad del Kernel HTTP. |
| 5 | `database/factories/UserFactory.php` usa el formato de factories de Laravel 7 (`$factory->define`), que ya no existe. | Bajo (ningún seeder lo usa). | Reescribirlo como clase `Database\Factories\UserFactory`. |
| 6 | Existen dos modelos `Region`: `app/Region.php` y `app/Models/Region.php`. | Bajo (confusión). | Identificar cuál se usa y eliminar el otro. |
| 7 | La carpeta `database/seeds` usa el nombre de Laravel 7 (en Laravel 8+ es `database/seeders` con namespace). | Bajo (funciona vía `classmap`). | Renombrar y agregar namespace al modernizar. |

---

## 7. Trabajo futuro

1. Migración de Laravel 10 a Laravel 12 para recuperar soporte de seguridad.
2. Reestructurar el proyecto con carpeta `public/` (hallazgo #1).
3. Ampliar las pruebas automatizadas: los tests actuales son los de ejemplo y no cubren `LocationController` ni `SyncLocations`.
