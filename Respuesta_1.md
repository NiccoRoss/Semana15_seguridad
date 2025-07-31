# 📱 Evaluación Técnica: Análisis y Mejora de Seguridad en Aplicación Android

## 📌 Introducción

Esta evaluación técnica se basa en una aplicación Android que implementa un sistema de demostración de permisos y protección de datos. La aplicación utiliza tecnologías modernas como **Kotlin**, **Android Security Crypto**, **SQLCipher** y patrones de arquitectura **MVVM**.

---

## 🛡️ Parte 1: Análisis de Seguridad Básico (0–7 puntos)

### 🔍 1.1 Identificación de Vulnerabilidades (2 puntos)

**Archivo analizado:** `DataProtectionManager.kt`

- **¿Qué método de encriptación se utiliza para proteger datos sensibles?**  
  `AES-256-GCM`

- **Posibles vulnerabilidades en el sistema de logging:**
  1. Los logs no están cifrados ni protegidos, lo que permite que servicios externos o usuarios con acceso al dispositivo puedan leer información sensible.
  2. Los mensajes de log son demasiado explícitos, lo que facilita a un atacante detectar puntos débiles o flujos críticos en la aplicación.

- **¿Qué sucede si falla la inicialización del sistema de encriptación?**  
  El `DataProtectionManager` se inicializa de forma perezosa (lazy). Si ocurre un error durante la inicialización y este no es manejado, la aplicación podría cerrarse inesperadamente (`crash`) o funcionar incorrectamente, sin mostrar errores visibles al usuario final.

---

### 🔐 1.2 Permisos y Manifiesto (2 puntos)

**Archivos analizados:** `AndroidManifest.xml`, `MainActivity.kt`

- **Permisos peligrosos declarados en el manifiesto:**

```xml
<uses-permission android:name="android.permission.READ_CONTACTS" />
<uses-permission android:name="android.permission.CALL_PHONE" />
<uses-permission android:name="android.permission.SEND_SMS" />

- **¿Qué patrón se utiliza para solicitar permisos en runtime?**
  Se utiliza el que recomienda google para este tipo de aplicaciones:
   `Activity Result API (registerForActivityResult con RequestPermission())`
  En este extracto:
```xml
permission.status = if (isGranted) PermissionStatus.GRANTED else PermissionStatus.DENIED
        permissionsAdapter.updatePermissionStatus(permission)
        
        val status = if (isGranted) "OTORGADO" else "DENEGADO"
        dataProtectionManager.logAccess("PERMISSION", "${permission.name}: $status")
        
        if (isGranted) {
            openActivity(permission)
        }
        currentRequestedPermission = null
    }
}


- **Identifica qué configuración de seguridad previene backups automáticos**

```xml
<application
    android:name=".PermissionsApplication"
    `android:allowBackup="false" `
    android:dataExtractionRules="@xml/data_extraction_rules"
    android:fullBackupContent="@xml/backup_rules"
    android:icon="@mipmap/ic_launcher"
    android:label="@string/app_name"
    android:roundIcon="@mipmap/ic_launcher_round"
    android:supportsRtl="true"
    android:theme="@style/Theme.Seguridad_priv_a">

La línea resaltada evita que Android respalde automáticamente datos de la aplicación 
(por ejemplo, configuraciones, archivos locales o credenciales) en servicios como Google Drive.

---

### 🔐 1.3 Gestión de Archivos (3 puntos)

**Archivos analizados:** `CameraActivity.kt`, `file_paths.xml`

- **¿Cómo se implementa la compartición segura de archivos de imágenes?**

Se hace utilizando FileProvider, el cual funciona como un intermediario para que se pueda acceder a la información
de diferentes apps sin exponer la privacidad.

private fun takePhoto() {
    try {
        val photoFile = createImageFile()
        currentPhotoUri = `FileProvider.getUriForFile` (
            this,
            "com.example.seguridad_priv_a.fileprovider",
            photoFile
        )
        
        currentPhotoUri?.let { uri ->
            takePictureLauncher.launch(uri)
        }
        dataProtectionManager.logAccess("CAMERA_ACCESS", "Iniciando captura de foto")
        
    } catch (ex: IOException) {
        dataProtectionManager.logAccess("CAMERA_ERROR", "Error al crear archivo de imagen: ${ex.message}")
        Toast.makeText(this, "Error al crear archivo de imagen", Toast.LENGTH_SHORT).show()
    }
}

En el archivo file_paths.xml se definen las rutas especificas para poder acceder a los archivos de imágenes:

<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path name="my_images" path="Pictures" />
    <external-files-path name="my_audio" path="Audio" />
</paths>

- **¿Qué autoridad se utiliza para el FileProvider?**

Se visualiza en AndroidManifest el Provider que se utiliza en nuestro código de CameraActivity.kt:

<provider
    android:name="androidx.core.content.FileProvider"
    ` android:authorities="com.example.seguridad_priv_a.fileprovider" `
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>


- **Explica por qué no se debe usar `file://` URIs directamente**

A partir de Android 7.0 (API 24), el uso de file:// URIs está prohibido por las siguientes razones:

  🔐 Seguridad: Revelan rutas absolutas del sistema de archivos.

  👁️ Privacidad: Pueden permitir acceso a archivos no destinados a otras apps.

  💥 Compatibilidad: Generan FileUriExposedException, lo que provoca que la app se cierre abruptamente si intenta compartir un file:// con otra app.

✅ La alternativa segura es usar content:// URIs proporcionadas por FileProvider, las cuales:

  1.Ocultan la ubicación real del archivo.
  2.Permiten permisos temporales controlados.
  3.Previenen exposición accidental de datos internos.
