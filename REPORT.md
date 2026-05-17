# Reporte Técnico — CVE-2026-31431 "Copy Fail"

## ¿Cuál es el bug raíz?

El bug está en la función `_aead_recvmsg()` del archivo `crypto/algif_aead.c` del kernel Linux. En 2017 se introdujo una optimización que usaba el mismo scatterlist (lista de páginas de memoria) como origen y destino en `aead_request_set_crypt()`. Esto significa que `req->src == req->dst`, lo que permite escribir en páginas del page cache sin pasar por los mecanismos normales de escritura del kernel.

## ¿Por qué el write a dst[assoclen + cryptlen] es peligroso?

Cuando src y dst apuntan al mismo scatterlist, la operación criptográfica escribe el resultado de vuelta en las mismas páginas de memoria que contienen el binario original. En este caso, las páginas del page cache de `/usr/bin/su` (un binario setuid-root) se corrompen en memoria con 4 bytes controlados por el atacante. Esto permite modificar el comportamiento del binario sin tocar el disco.

## ¿Por qué el exploit es "stealthy"?

El exploit no modifica el archivo en disco. Solo corrompe las páginas en el page cache (memoria RAM). El archivo `/usr/bin/su` en disco permanece intacto. Esto significa que herramientas de detección basadas en checksums de archivos no detectarían el ataque. Al reiniciar el sistema, el page cache se limpia y todo vuelve a la normalidad.

## Conexión con conceptos del curso

- **Page cache**: El kernel mantiene en RAM copias de archivos para acceso rápido. El exploit abusa de esto para modificar un binario en memoria.
- **setuid**: `/usr/bin/su` tiene el bit setuid activado (`chmod u+s`), lo que significa que se ejecuta con privilegios de root sin importar quién lo llame.
- **Inodos**: El inodo del archivo apunta a las mismas páginas físicas del page cache que el exploit corrompe.
- **chmod**: Sin el bit setuid en `su`, el exploit no tendría efecto porque no habría escalada de privilegios.

## ¿Qué aprendí?

Este CVE demuestra que múltiples cambios aparentemente razonables pueden crear vulnerabilidades graves. La optimización de 2017 tenía sentido en aislamiento: reutilizar el mismo buffer ahorra memoria. Pero combinada con el hecho de que AF_ALG expone operaciones criptográficas a usuarios sin privilegios, y que el page cache comparte páginas con binarios setuid, creó un vector de ataque devastador que permaneció oculto por casi una década en todas las distribuciones Linux principales.
