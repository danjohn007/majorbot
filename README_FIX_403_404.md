# 🔧 Fix Rápido - Errores 403 y 404

## ⚡ Solución en 2 Pasos

Si tienes errores **403 FORBIDDEN** en la raíz o **404 Not Found** en public/, sigue estos pasos:

### Paso 1: Actualizar `.htaccess` raíz

Abre el archivo `.htaccess` en la raíz del proyecto y **elimina** esta línea:
```apache
RewriteCond %{REQUEST_FILENAME} !-d
```

El archivo debe quedar así:
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

### Paso 2: Actualizar `public/.htaccess`

Abre el archivo `public/.htaccess` y **agrega estas líneas al inicio**:
```apache
# Set DirectoryIndex to prevent 403 errors
DirectoryIndex index.php
```

El archivo debe quedar así:
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

### Paso 3: Verificar

1. Recarga Apache:
   ```bash
   sudo service apache2 reload
   ```

2. Accede a tu sitio:
   - `http://localhost/majorbot/` → ✅ Debe mostrar la página de inicio
   - `http://localhost/majorbot/public/` → ✅ Debe mostrar la página de inicio

## 📚 Documentación Completa

Para entender qué causa estos errores y cómo funciona la solución:

- **Explicación detallada**: [FIX_403_404_ERRORS.md](FIX_403_404_ERRORS.md)
- **Resumen visual**: [SOLUTION_SUMMARY.md](SOLUTION_SUMMARY.md)
- **Registro de cambios**: [CHANGELOG_FIX_403_404.md](CHANGELOG_FIX_403_404.md)

## ❓ ¿Por Qué Funciona?

### Problema 1: Error 403 en raíz
**Causa**: La condición `!-d` impedía redirigir directorios a `public/`  
**Solución**: Eliminar esa condición permite redirigir todo a `public/`

### Problema 2: Error 404 en public/
**Causa**: Apache no sabía qué archivo servir al acceder al directorio  
**Solución**: `DirectoryIndex index.php` le indica que sirva `index.php`

## ✅ Checklist de Verificación

- [ ] mod_rewrite está habilitado: `apache2ctl -M | grep rewrite`
- [ ] AllowOverride está en `All` en la configuración de Apache
- [ ] Archivo `.htaccess` raíz actualizado (sin línea `!-d`)
- [ ] Archivo `public/.htaccess` actualizado (con `DirectoryIndex`)
- [ ] Permisos correctos: `chmod 644 .htaccess public/.htaccess`
- [ ] Apache reiniciado: `sudo service apache2 reload`

## 🆘 ¿Aún Tienes Problemas?

1. Verifica los logs de Apache:
   ```bash
   tail -f /var/log/apache2/error.log
   ```

2. Revisa la configuración de Apache:
   ```apache
   <Directory /var/www/html/majorbot>
       AllowOverride All
       Require all granted
   </Directory>
   ```

3. Consulta la documentación completa en [INSTALLATION.md](INSTALLATION.md)

---

**Nota**: Estos cambios son permanentes y resuelven definitivamente los errores 403 y 404.
