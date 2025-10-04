# Corrección de Errores 403 FORBIDDEN y 404 Not Found

## Problema Identificado

El sistema presentaba dos errores principales:

1. **Error 403 FORBIDDEN en el directorio raíz (`/majorbot/`)**: Cuando se accedía al directorio raíz del proyecto, Apache mostraba un error 403 FORBIDDEN porque no encontraba un archivo index válido y el listado de directorios estaba deshabilitado.

2. **Error 404 Not Found en `public/`**: Cuando se accedía directamente a la carpeta `public/`, se mostraba un error 404 porque el enrutador no podía manejar correctamente el acceso directo al directorio.

## Causa Raíz

### Problema 1: Root .htaccess
El archivo `.htaccess` en la raíz del proyecto tenía la siguiente condición:

```apache
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ public/$1 [L]
```

La condición `!-d` significa "NO aplicar esta regla si el REQUEST_FILENAME es un directorio". Esto causaba que cuando se accedía al directorio raíz (`/majorbot/`), la redirección a `public/` NO se aplicaba, resultando en un error 403 FORBIDDEN.

### Problema 2: Public .htaccess
El archivo `.htaccess` en `public/` no tenía la directiva `DirectoryIndex` configurada, por lo que Apache no sabía qué archivo servir cuando se accedía al directorio `public/` directamente.

## Solución Implementada

### 1. Actualización de `.htaccess` raíz

**ANTES:**
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    
    # Allow direct access to test_connection.php and other root files
    RewriteCond %{REQUEST_FILENAME} -f
    RewriteRule ^ - [L]
    
    # Redirect to public folder
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

**DESPUÉS:**
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    
    # Allow direct access to test_connection.php and other root files
    RewriteCond %{REQUEST_FILENAME} -f
    RewriteRule ^ - [L]
    
    # Redirect everything else to public folder (including root directory access)
    RewriteRule ^(.*)$ public/$1 [L]
</IfModule>
```

**Cambios:**
- Se eliminó la condición `RewriteCond %{REQUEST_FILENAME} !-d`
- Ahora TODOS los accesos (incluyendo accesos a directorios) se redirigen a `public/`, excepto archivos que existen en el root (como `test_connection.php`)

### 2. Actualización de `public/.htaccess`

**ANTES:**
```apache
<IfModule mod_rewrite.c>
    RewriteEngine On
    
    # Don't rewrite files or directories
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    
    # Rewrite everything else to index.php
    RewriteRule ^(.*)$ index.php?url=$1 [QSA,L]
</IfModule>
```

**DESPUÉS:**
```apache
# Set DirectoryIndex to prevent 403 errors
DirectoryIndex index.php

<IfModule mod_rewrite.c>
    RewriteEngine On
    
    # Don't rewrite files or directories
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    
    # Rewrite everything else to index.php
    RewriteRule ^(.*)$ index.php?url=$1 [QSA,L]
</IfModule>
```

**Cambios:**
- Se agregó `DirectoryIndex index.php` al inicio del archivo
- Esto le indica a Apache que cuando se accede a un directorio (como `public/`), debe servir automáticamente `index.php`

## Cómo Funciona Ahora

### Escenario 1: Acceso al directorio raíz (`http://localhost/majorbot/`)

1. Apache procesa `.htaccess` raíz
2. Verifica si `/majorbot/` es un archivo existente → NO
3. Aplica la regla: `RewriteRule ^(.*)$ public/$1`
4. Redirige internamente a `public/`
5. Apache procesa `public/.htaccess`
6. Ve `DirectoryIndex index.php` y sirve `public/index.php`
7. El Router carga el HomeController y muestra la página de inicio

### Escenario 2: Acceso directo a public/ (`http://localhost/majorbot/public/`)

1. Apache procesa `public/.htaccess`
2. Ve `DirectoryIndex index.php` y sirve `public/index.php`
3. El Router carga el HomeController y muestra la página de inicio

### Escenario 3: Acceso a una ruta específica (`http://localhost/majorbot/auth/login`)

1. Apache procesa `.htaccess` raíz
2. Verifica si `/majorbot/auth/login` es un archivo existente → NO
3. Aplica la regla: redirige internamente a `public/auth/login`
4. Apache procesa `public/.htaccess`
5. Verifica si `public/auth/login` es un archivo o directorio → NO
6. Aplica la regla: `RewriteRule ^(.*)$ index.php?url=$1`
7. Redirige a `index.php?url=auth/login`
8. El Router parsea la URL y carga `AuthController->login()`

### Escenario 4: Acceso a test_connection.php (`http://localhost/majorbot/test_connection.php`)

1. Apache procesa `.htaccess` raíz
2. Verifica si `/majorbot/test_connection.php` es un archivo existente → SÍ
3. Aplica la regla: `RewriteRule ^ - [L]` (no reescribe, sirve el archivo directamente)
4. Apache sirve `test_connection.php`

## Verificación

Para verificar que los cambios funcionan correctamente:

1. **Verificar que mod_rewrite está habilitado:**
   ```bash
   apache2ctl -M | grep rewrite
   ```
   Debe mostrar: `rewrite_module (shared)`

2. **Verificar permisos de los archivos .htaccess:**
   ```bash
   ls -la .htaccess public/.htaccess
   ```
   Deben ser legibles por Apache (normalmente 644)

3. **Verificar configuración de Apache:**
   Asegurarse de que `AllowOverride All` esté configurado para el directorio del proyecto

4. **Probar los accesos:**
   - `http://localhost/majorbot/` → Debe mostrar la página de inicio
   - `http://localhost/majorbot/public/` → Debe mostrar la página de inicio
   - `http://localhost/majorbot/auth/login` → Debe mostrar el formulario de login
   - `http://localhost/majorbot/test_connection.php` → Debe mostrar el test de conexión

## Beneficios de la Solución

1. **Consistencia**: Todos los accesos al directorio raíz ahora funcionan correctamente
2. **Seguridad**: Los directorios internos (app, config, core) siguen protegidos
3. **Flexibilidad**: Los archivos raíz específicos (como test_connection.php) siguen accesibles
4. **Estándares**: Usa prácticas estándar de Apache (DirectoryIndex)
5. **Mantenibilidad**: Código más simple y fácil de entender

## Compatibilidad

Esta solución funciona en:
- Apache 2.4+
- PHP 7.0+
- Todas las configuraciones de directorio (subdirectorios, virtual hosts, etc.)
- Servidores Linux, Windows y macOS

## Notas Adicionales

- Si estás actualizando desde una versión anterior, asegúrate de reemplazar AMBOS archivos `.htaccess`
- Los cambios son retrocompatibles con el resto del código del sistema
- No se requieren cambios en la configuración de Apache más allá de tener `AllowOverride All` habilitado
