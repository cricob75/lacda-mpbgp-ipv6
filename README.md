# lacda-mpbgp-ipv6
Laboratorios prácticos de MP-BGPv6 - LAC Digital Academy
# Laboratorio Práctico — Bloque 1: Fundamentos y Adyacencias eBGPv6

**Curso:** Enrutamiento IPv6 en el Mundo Real: MP-BGPv6 para ISPs e IXPs

**Institución:** LAC Digital Academy

**Duración estimada:** 75 minutos

## 1. Topología y Plan de Direccionamiento

En este laboratorio operarás el router de borde de tu propio Sistema Autónomo (**`R1-ISP` — AS 65001**) y establecerás conectividad **eBGPv6** con un proveedor de tránsito (**`R2-TRANSIT` — AS 65002**), el cual ya se encuentra preconfigurado.

```
+-----------------------------------+                     +-----------------------------------+
|              R1-ISP               |                     |            R2-TRANSIT             |
|            (AS 65001)             |                     |            (AS 65002)             |
|       Router-ID: 1.1.1.1          |                     |       Router-ID: 2.2.2.2          |
|                                   |                     |                                   |
|  Loopback (lo100):                |                     |  Loopback (lo200):                |
|  2001:db8:100::1/48               |                     |  2001:db8:200::1/48               |
|                                   |   Enlace /127       |                                   |
|                             [eth1]=======================[eth1]                             |
|                    2001:db8:12::1/127                 2001:db8:12::0/127                    |
+-----------------------------------+                     +-----------------------------------+

```

### Tabla de Direccionamiento IPv6

| Dispositivo | ASN | Router-ID | Interfaz `eth1` (Enlace `/127`) | Interfaz Loopback (Prefijo Cliente `/48`) | 
| ----- | ----- | ----- | ----- | ----- | 
| **`R1-ISP`** *(Tu router)* | `65001` | `1.1.1.1` | `2001:db8:12::1/127` | `2001:db8:100::1/48` (`lo100`) | 
| **`R2-TRANSIT`** *(Proveedor)* | `65002` | `2.2.2.2` | `2001:db8:12::0/127` | `2001:db8:200::1/48` (`lo200`) | 

## 2. Fase 0: Preparación del Entorno y Visualización Gráfica (10 min)

### Paso 0.1: Abrir el laboratorio en la nube (GitHub Codespaces)

1. Ingresa al repositorio del curso en GitHub con tu cuenta personal.

2. Haz clic en el botón verde **`<> Code`**.

3. Selecciona la pestaña **`Codespaces`** y haz clic en **`Create codespace on main`**.

4. Espera aproximadamente 2 minutos mientras se prepara tu entorno virtual con Containerlab y FRRouting. Cuando aparezca la terminal en la parte inferior, estás listo.

### Paso 0.2: Desplegar los routers virtuales

En la terminal inferior de Codespaces, ejecuta el siguiente comando para encender ambos routers:

```
sudo containerlab deploy -t bloque1.clab.yml

```

*(Verás una tabla confirmando que `clab-bloque1-ebgpv6-R1-ISP` y `clab-bloque1-ebgpv6-R2-TRANSIT` están en estado `running`).*

### Paso 0.3: Visualizar la topología en tu navegador

1. En esa misma terminal, inicia el visualizador gráfico de Containerlab:

   ```
   sudo containerlab graph -t bloque1.clab.yml
   
   ```

2. Aparecerá una notificación en la esquina inferior derecha indicando que el puerto **50080** está disponible. Haz clic en **`Open in Browser`** para ver el diagrama interactivo de tus routers en una pestaña nueva.

3. Regresa a la pestaña de Codespaces y abre una **segunda terminal** haciendo clic en el ícono **`+`** (arriba a la derecha del panel de la terminal) para trabajar sin cerrar el mapa visual.

### Paso 0.4: Verificar conectividad física IPv6 (Capa 2 / Capa 3)

Antes de configurar BGP, entra a la consola de tu router **`R1-ISP`**:

```
docker exec -it clab-bloque1-ebgpv6-R1-ISP vtysh

```

Comprueba tus interfaces y haz ping hacia el extremo del proveedor (`2001:db8:12::0`):

```
show ipv6 interface brief
ping ipv6 2001:db8:12::0

```

* **Resultado esperado:** Debes recibir respuesta de ping exitosa (`5 packets transmitted, 5 received`). Si funciona, el protocolo NDP (ICMPv6 tipos 135/136) resolvió correctamente la dirección MAC del vecino.

## 3. Ejercicio 1: Adyacencia eBGPv6 sobre Global Unicast (25 min)

**Objetivo:** Aplicar las buenas prácticas de configuración base (Checklist Pre-Laboratorio) para establecer una sesión eBGPv6 con `R2-TRANSIT` usando direcciones Global Unicast (GUA), anunciar tu bloque `/48` y verificar el tamaño del `Next-Hop` (16 bytes).

### Paso 1.1: Configurar el proceso BGP en `R1-ISP`

Dentro de `vtysh` en `R1-ISP`, ingresa al modo de configuración y ejecuta los 4 pasos fundamentales:

```
configure terminal
router bgp 65001
 ! 1. Identificador obligatorio de 32 bits
 bgp router-id 1.1.1.1
 ! 2. Desactivar activación automática de IPv4 Unicast
 no bgp default ipv4-unicast
 ! 3. Declarar al vecino externo (Proveedor)
 neighbor 2001:db8:12::0 remote-as 65002
 !
 ! 4. Entrar a la familia IPv6 Unicast, activar vecino y anunciar prefijo
 address-family ipv6 unicast
  neighbor 2001:db8:12::0 activate
  network 2001:db8:100::/48
 end

```

### Paso 1.2: Verificar el estado de la sesión (FSM)

Ejecuta el comando de resumen BGP para IPv6:

```
show bgp ipv6 unicast summary

```

* **Qué debes observar:** En la columna `State/PfxRcd` frente al vecino `2001:db8:12::0`, debe aparecer el número **`1`** (indica que la sesión llegó a estado `Established` y recibiste 1 prefijo desde `R2-TRANSIT`).

### Paso 1.3: Inspeccionar la tabla BGP y el `Next-Hop` de 16 bytes

Ejecuta:

```
show bgp ipv6 unicast

```

* **Qué debes observar:**

  1. Verás tu propia red `2001:db8:100::/48` con `Next Hop ::` (originada localmente, código de origen `i`).

  2. Verás la red del proveedor `2001:db8:200::/48` aprendida por eBGP (`Path: 65002 i`) con un **único `Next Hop` de 16 bytes**: `2001:db8:12::0`.

### Paso 1.4: Prueba de conectividad extremo a extremo

Verifica que tu red de clientes (`2001:db8:100::1`) alcanza la red del proveedor (`2001:db8:200::1`):

```
ping ipv6 2001:db8:200::1 source 2001:db8:100::1

```

* **Resultado esperado:** Ping exitoso (`0% packet loss`).

## 4. Ejercicio 2: Comprobando la "Regla de Originación" (15 min)

**Objetivo:** Demostrar por qué BGP exige una coincidencia exacta de prefijo y longitud de máscara en la tabla de enrutamiento local (RIB) antes de anunciar una ruta.

### Paso 2.1: Intentar anunciar una subred más específica (`/64`)

Supongamos que intentas anunciar un bloque `/64` (`2001:db8:100:1::/64`) que no existe explícitamente en tus interfaces:

```
configure terminal
router bgp 65001
 address-family ipv6 unicast
  network 2001:db8:100:1::/64
 end

```

### Paso 2.2: Verificar si se anuncia

Revisa tu tabla BGP y qué le estás enviando a tu vecino:

```
show bgp ipv6 unicast
show bgp ipv6 unicast neighbors 2001:db8:12::0 advertised-routes

```

* **Pregunta de análisis:** ¿Aparece `2001:db8:100:1::/64` en las rutas anunciadas?

* **Explicación:** **No.** Al revisar `show ipv6 route`, el router solo tiene conectada la red `2001:db8:100::/48`. Como no existe una entrada exacta `/64` en la RIB, BGP ignora el comando `network` silenciosamente.

### Paso 2.3: Limpiar el comando

Elimina la línea errónea para mantener limpia tu configuración:

```
configure terminal
router bgp 65001
 address-family ipv6 unicast
  no network 2001:db8:100:1::/64
 end

```

## 5. Ejercicio 3: Peering por Link-Local (`fe80::`) y Next-Hop de 32 Bytes (25 min)

**Objetivo:** Migrar la adyacencia eBGPv6 para utilizar direcciones **Link-Local (`fe80::`)** en lugar de Global Unicast y comprobar en la tabla BGP cómo el atributo `MP_REACH_NLRI` transporta **32 bytes (GUA + Link-Local)** según el estándar **RFC 2545**.

### Paso 3.1: Descubrir la dirección Link-Local del vecino

En `R1-ISP` (`vtysh`), consulta la tabla de vecinos NDP para ver la dirección `fe80::` asociada a `eth1`:

```
show ipv6 neighbors

```

* Toma nota de la dirección que empieza con **`fe80::`** en la interfaz `eth1` (por ejemplo, `fe80::a8c1:abff:fe...`).

También consulta cuál es tu propia dirección Link-Local en `eth1` (la necesitarás para configurar el otro extremo):

```
show interface eth1

```

* Copia la dirección `inet6 fe80::.../64` de tu `R1-ISP`.

### Paso 3.2: Reconfigurar `R1-ISP` para usar vecino Link-Local

En `R1-ISP` (`vtysh`), elimina el vecino GUA anterior y configura el nuevo vecino usando su dirección `fe80::` e indicando explícitamente la interfaz `eth1`:

```
configure terminal
router bgp 65001
 no neighbor 2001:db8:12::0
 neighbor <FE80_DE_R2> remote-as 65002
 neighbor <FE80_DE_R2> interface eth1
 !
 address-family ipv6 unicast
  neighbor <FE80_DE_R2> activate
  network 2001:db8:100::/48
 end

```

*(Reemplaza `<FE80_DE_R2>` por la dirección Link-Local de `R2-TRANSIT` que obtuviste en el paso 3.1).*

### Paso 3.3: Actualizar el extremo del proveedor (`R2-TRANSIT`)

Abre una **tercera terminal** (clic en `+`) o sal momentáneamente de `R1-ISP` (`exit`) y entra a la consola de `R2-TRANSIT`:

```
docker exec -it clab-bloque1-ebgpv6-R2-TRANSIT vtysh

```

Actualiza la vecindad en `R2-TRANSIT` apuntando a la dirección `fe80::` de `R1-ISP`:

```
configure terminal
router bgp 65002
 no neighbor 2001:db8:12::1
 neighbor <FE80_DE_R1> remote-as 65001
 neighbor <FE80_DE_R1> interface eth1
 !
 address-family ipv6 unicast
  neighbor <FE80_DE_R1> activate
  network 2001:db8:200::/48
 end

```

### Paso 3.4: Comprobar el Next-Hop doble (32 bytes) en `R1-ISP`

Regresa a la terminal de **`R1-ISP`** (`vtysh`) y ejecuta:

```
show bgp ipv6 unicast summary
show bgp ipv6 unicast

```

* **Qué debes observar:**

  1. La sesión vuelve a estado `Established` (`State/PfxRcd = 1`).

  2. Al inspeccionar la ruta `2001:db8:200::/48` (o ejecutando `show bgp ipv6 unicast 2001:db8:200::/48`), verás que ahora el campo **Next-Hop contiene dos direcciones (32 bytes en total)**:

     * La dirección **GUA** de la interfaz `eth1` de `R2-TRANSIT` (`2001:db8:12::0` — 16 bytes).

     * La dirección **Link-Local** (`fe80::...` — 16 bytes), que el router utiliza directamente para resolver la MAC por NDP sin hacer búsquedas recursivas adicionales.

## 6. Cierre del Laboratorio (Importante para ahorrar horas gratuitas)

Al finalizar la sesión del día:

1. En la terminal de Codespaces, destruye el laboratorio limpio con:

   ```
   sudo containerlab destroy -t bloque1.clab.yml
   
   ```

2. Ve a [github.com/codespaces](https://github.com/codespaces), haz clic en el menú **`...`** de tu Codespace y selecciona **`Stop codespace`**.
