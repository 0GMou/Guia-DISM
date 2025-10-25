# 📖Guia sobre como "GENERALIZAR" & "CAPTURAR"📷 tu SO

******Objetivo: Capturar el estado un sistema Windows desde el mismo equipo y generar **una imagen ISO booteable** que lo instale “tal cual” en otros equipos, usaremos [Windows 10 Enterprise LTSC IoT](https://massgrave.dev/windows_ltsc_links) en esta guia.******

---

## Prerrequisitos

* Permisos de **administrador**.
* **Disco externo** con espacio libre ≥ tamaño de tu partición de Windows. Ahí guardarás `install.wim`.
* **Medio base de Windows 10**: un **ISO original** o un **USB instalador clásico** de Windows (el que contiene `setup.exe`, `\sources\boot.wim`, etc.).
* **Scoop** instalado. Se usará para instalar la herramienta de creación de ISO.
* Conexión a Internet para que Scoop descargue paquetes.
* A tener en cuenta: si C: está protegido con **BitLocker**, suspende los protectores antes de generalizar o capturar.

---

## Convenciones de letras de unidad

* En **WinRE** (entorno de recuperación), la letra de tu Windows **no** tiene por qué ser `C:`.
* En este documento:

  * `X:` = volumen donde está **tu Windows offline** (donde exista `X:\Windows\System32`).
  * `E:` = **disco externo** donde guardarás la captura `install.wim`.
  * `C:\ISO_WORK` = carpeta local donde prepararás el “medio base” antes de crear el ISO.
* **Sustituye** `X:` y `E:` por tus letras reales cuando se indique.

---

## Paso 1️ Preparar el sistema de referencia

1. **Prepara tu sistema a tu gusto**, actualizalo, instala las aplicaciones que quieras, borra las que no...
2. **Generalizar el equipo** para instalar en hardware distinto:

   * Abre **CMD como administrador** y ejecuta:

     ```
     %WINDIR%\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown /quiet
     ```
   * El equipo se **apaga** al terminar.

> Si vas a reinstalar **solo en el mismo hardware**, podrías omitir Sysprep, pero es recomendable. Con hardware distinto, Sysprep es **imprescindible**.

---

## Paso 2️ Entrar en WinRE para capturar el sistema **sin montarlo**:

1. Enciende el equipo, aparecerá en OOBE por Sysprep.
2. Pulsa `Shift`+`F10` para abrir **CMD** y ejecuta:

  ```
  shutdown /r /o /f /t 0
  ```
3. Tras el reinicio y en la pantalla azul, sigue **exactamente** este camino:

  - **Solucionar problemas**
  - **Opciones avanzadas**
  - **Símbolo del sistema**
  - **Selecciona tu cuenta de administrador** y escribe **la contraseña** (no el PIN).

---

## Paso 3️ Identificar letras de las unidades en WinRE

En la consola de **WinRE**:

******Inicia la herramienta de administracion de discos, volumenes y particiones de windows.******
```
diskpart
```
******Lista los volumenes actuales en el sistema.******
```
list vol
```
******Para salir de diskpart.******
```
exit
```

* Localiza la particion de tu **Windows**. `X:` no es tu Windows. Cambia la `X:` por otra letra la que corresponde.
* Localiza tu **disco externo** y anota su letra, que será `Y:` en los comandos.
* **Es obligatorio** cambiar `X:` y `Y:` en los comandos de captura por **tus letras reales**.
  * Para comprobar si el volumen es el que contiene **Windows**, utiliza este comando:

    ```
    dir X:\Windows\System32
    ```

---

## Paso 4️ Capturar Windows a WIM con DISM
> Cambia `X:\` por la **letra real** de la unidad donde está tu Windows y cambia `Y:` por la **letra real** de tu disco externo.

En la consola de **WinRE**:

******Crea el directorio donde se guardara la imagen******
```
md Y:\CAP
```
******Captura la imagen en formato WIM******
```
dism /Capture-Image /ImageFile:Y:\CAP\install.wim /CaptureDir:X:\ /Name:"NOMBRE_DE_TU_ISO" /Description:"DESCRIPCION_DE_TU_ISO" /Compress:max /CheckIntegrity /Bootable /ScratchDir:Y:\CAP
```
******Verifica el WIM creado:******
```
dism /Get-WimInfo /WimFile:Y:\CAP\install.wim
```



---

## Paso 5️ Salir de WinRE y completar OOBE

* En la misma consola de **WinRE**:

  ```
  wpeutil reboot
  ```
* Tras el reinicio, aparecerá **OOBE** debido a Sysprep. **Complétalo** con lo mínimo para entrar al escritorio.

  * Este OOBE **no afecta** al contenido ya capturado en `install.wim`.

---

## Paso 6️ Preparar la “BASE” en `C:\ISO_WORK`

******Crea la carpeta:******
```
New-Item -ItemType Directory -Force C:\ISO_WORK\sources | Out-Null
```
******Copia los archivos de la imagen ISO a la carpeta "BASE" (sustituye `X:` por la letra de la imagen ISO montada):******
```
robocopy X:\ C:\ISO_WORK /e
```
******Comprueba que el medio base sea válido:******
```
if(!(Test-Path C:\ISO_WORK\setup.exe) -or !(Test-Path C:\ISO_WORK\boot\etfsboot.com) -or !((Test-Path C:\ISO_WORK\efi\microsoft\boot\efisys.bin) -or (Test-Path C:\ISO_WORK\efi\microsoft\boot\efisys_noprompt.bin))){Write-Error 'Medio base incompleto en C:\ISO_WORK';exit}
```

---

## Paso 7️ Insertar tu captura en el medio base

> Cambia `Y:` por la letra real de tu disco externo.

******Copia el WIM capturado **desde el disco externo** a `C:\ISO_WORK\sources`:******

```
Copy-Item Y:\CAP\install.wim C:\ISO_WORK\sources\install.wim -Force
```

---

## Paso 8️ Instalar herramienta de creación de ISO

******Instala **cdrtools** con **Scoop** para disponer de `mkisofs`:******

```
scoop install cdrtools
```

---

## Paso 9️ Generar el ISO booteable BIOS+UEFI

******Crea `C:\NOMBRE_DE_TU_ISO.iso`:******

```
$SRC='C:\ISO_WORK';$OUT='C:\NOMBRE_DE_TU_ISO.iso';$EFIPATH=$(if(Test-Path "$SRC\efi\microsoft\boot\efisys.bin"){'efi/microsoft/boot/efisys.bin'}else{'efi/microsoft/boot/efisys_noprompt.bin'}); mkisofs -iso-level 3 -UDF -J -l -D -relaxed-filenames -allow-lowercase -V NOMBRE_DE_TU_ISO -b boot/etfsboot.com -no-emul-boot -boot-load-size 8 -boot-info-table -eltorito-alt-boot -eltorito-platform efi -eltorito-boot $EFIPATH -no-emul-boot -o $OUT $SRC
```

Validación opcional del archivo resultante:

```
Get-FileHash C:\NOMBRE_DE_TU_ISO.iso -Algorithm SHA256
```

---

## Resultado

* Has capturado tu Windows en `install.wim`.
* Has ensamblado un **ISO instalable** que incluye tu `install.wim` dentro del medio base de Windows.
* Ese ISO es booteable en **BIOS y UEFI**.
