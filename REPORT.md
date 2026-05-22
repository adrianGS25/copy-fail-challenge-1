# Análisis Técnico — CVE-2026-31431 "Copy Fail"

## Bug raíz

La vulnerabilidad reside en el subsistema criptográfico del kernel Linux,
específicamente en el archivo crypto/algif_aead.c dentro de la función
_aead_recvmsg(). El problema fue introducido en 2017 como una optimización
de rendimiento: en lugar de mantener dos scatterlists separados (uno de
entrada y uno de salida), el código los unificó en uno solo usando
sg_chain() y luego asignó req->src = req->dst. Esta decisión, aparentemente
inocente, tuvo consecuencias graves durante casi una década.

## El write peligroso

El kernel de Linux mantiene en RAM una copia de los archivos que se están
usando, llamada page cache. Cuando un programa abre /usr/bin/su, el kernel
carga sus páginas en memoria. El bug permite que esas páginas del page cache
queden incluidas dentro del scatterlist de escritura de la operación AEAD.
La escritura en dst[assoclen + cryptlen] corresponde exactamente al tag de
autenticación, y ese offset cae dentro de las páginas cacheadas de su.
El resultado: el kernel escribe datos controlados por el atacante directamente
en el código ejecutable de su, sin verificar permisos ni pasar por el
sistema de archivos.

## Por qué no deja rastros

La clave está en entender la diferencia entre el archivo en disco y su
representación en memoria. El exploit nunca llama a write(), nunca modifica
inodos, nunca toca bloques de disco. Solo manipula páginas en RAM a través
del subsistema criptográfico. Si un investigador forense revisa el disco
después del ataque, encontrará el binario su completamente intacto. El hash
SHA256, los timestamps, los permisos, todo igual. El ataque vive y muere
en RAM, desapareciendo con el siguiente reinicio.

## Conexión con conceptos del curso

El page cache es el puente entre el sistema de archivos y la ejecución de
programas. Normalmente, escribir en un archivo requiere permisos (chmod),
pasa por el VFS, y actualiza el inodo. El exploit salta todo ese proceso.

El bit setuid es lo que convierte este bug en escalada de privilegios. Sin
setuid, corromper su no serviría de nada. Pero porque su tiene chmod 4755,
cualquier usuario que lo ejecute obtiene uid=0. El exploit corrompe su en
memoria, luego lo ejecuta, y el kernel obedientemente eleva los privilegios.

Los inodos son los metadatos del sistema de archivos: permisos, propietario,
tamaño, timestamps. El exploit no modifica ninguno de estos. Opera a un nivel
más bajo, directamente en las páginas físicas de RAM que el kernel usa para
ejecutar el binario.

## Reflexión final

Lo más llamativo de CVE-2026-31431 es que cada pieza del rompecabezas es
completamente normal. AF_ALG es una API legítima para operaciones criptográficas.
splice() es una llamada al sistema estándar para mover datos eficientemente.
La optimización in-place de 2017 mejoró el rendimiento sin romper ningún test.
Pero la combinación de estas tres piezas, en el orden correcto, crea una
primitiva de escritura arbitraria en el kernel. Esto enseña que la seguridad
no puede evaluarse función por función: hay que considerar cómo interactúan
los componentes entre sí, especialmente en sistemas tan complejos como el kernel.
