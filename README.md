# Guía de Laboratorio — Bloque 1: Fundamentos de MP-BGPv6 e Intercambio de Rutas

**Curso:** Enrutamiento IPv6 en el Mundo Real: MP-BGPv6 para ISPs e IXPs  
**Institución:** LAC Digital Academy  
**Duración estimada de la práctica:** 60 – 75 minutos

---

## 1. Arquitectura de la Topología

En este laboratorio operarás como el ingeniero de red del sistema autónomo **`AS 65001` (`R1-ISP`)**. Tu objetivo es establecer una sesión **eBGPv6** con tu proveedor de tránsito **`AS 65002` (`R2-TRANSIT`)**, aplicar políticas de enrutamiento seguras (**RFC 8212**) y analizar el comportamiento del atributo `MP_REACH_NLRI` y el `NEXT_HOP` (RFC 2545).

```text
+-----------------------------------+                       +-----------------------------------+
|             R1-ISP                |                       |            R2-TRANSIT             |
|            AS 65001               |                       |             AS 65002              |
|        Router-ID: 1.1.1.1         |                       |         Router-ID: 2.2.2.2        |
|                                   |                       |                                   |
|  lo100: 2001:db8:100::1/48        |  2001:db8:12::/127    |  lo200: 2001:db8:200::1/48        |
|                              eth1 +-----------------------+ eth1                              |
|                   2001:db8:12::1  |      (RFC 6164)       |  2001:db8:12::0                   |
+-----------------------------------+                       +-----------------------------------+
      (Tu router a configurar)                                    (Ya preconfigurado)
```

---

## 2. Preparación del Entorno y Visualización Gráfica (10 min)

### Paso 2.1: Abrir el laboratorio en GitHub Codespaces

1. Ingresa al repositorio oficial del curso: `https://github.com/cricob75/lacda-mpbgp-ipv6`
2. Haz clic en el botón verde **`<> Code`**.
3. Selecciona la pestaña **`Codespaces`** y haz clic en **`Create codespace on main`**.
4. Espera entre 1 y 3 minutos mientras se construye tu máquina virtual en la nube con Docker, Containerlab, la extensión visual de VS Code y FRRouting listos.

### Paso 2.2: Desplegar los routers virtuales

En la terminal inferior de tu Codespace, ejecuta:

```bash
sudo containerlab deploy -t bloque1.clab.yml
```

**Verificación:** En menos de 10 segundos verás una tabla confirmando que `clab-bloque1-ebgpv6-R1-ISP` y `clab-bloque1-ebgpv6-R2-TRANSIT` están en estado `running`.

### Paso 2.3: Visualizar el laboratorio en la barra lateral de VS Code (y mapa web)

Tu entorno ya tiene preinstalada la extensión oficial de **Containerlab** para facilitar la navegación y el acceso a los equipos:

1. **Panel lateral de Containerlab (Acceso rápido):**
   * Haz clic en el ícono de **Containerlab** (nodo de red) en la barra de actividades vertical a la izquierda de VS Code.
   * Despliega el árbol **`bloque1-ebgpv6`** para inspeccionar en vivo tus nodos (`R1-ISP` y `R2-TRANSIT`), su estado (`running`), interfaces (`eth1`) y direcciones IP.
2. *(Opcional)* **Mapa gráfico de topología en el navegador:**
   * Si deseas ver el diagrama de enlaces en pantalla completa, ejecuta `sudo containerlab graph -t bloque1.clab.yml` en una terminal, ve a la pestaña **`Puertos`**, ubica el puerto **`50080`** y haz clic en el ícono del **globo terráqueo ("Abrir en el explorador")**.

### Paso 2.4: Conectarte a `R1-ISP` y verificar conectividad IPv6

Puedes conectarte a la consola de **`R1-ISP`** de dos formas (elige la que te resulte más cómoda):

* **Opción A — Desde la barra lateral de Containerlab (1 clic):**
  1. En el panel izquierdo de **Containerlab**, pasa el cursor sobre **`R1-ISP`** y haz clic en el ícono de **Terminal (`>_`)** que aparece a su derecha.
  2. Se abrirá una consola dentro del contenedor; escribe **`vtysh`** y presiona Enter para entrar al CLI del router (`R1-ISP#`).
* **Opción B — Desde la terminal principal:**
  ```bash
  docker exec -it clab-bloque1-ebgpv6-R1-ISP vtysh
  ```

Una vez dentro del prompt **`R1-ISP#`**, verifica tus interfaces y prueba conectividad hacia el extremo remoto:

```text
show interface brief
ping ipv6 2001:db8:12::0
```

**Verificación:**
* `eth1` debe aparecer `up` con `2001:db8:12::1/127` y `lo100` `up` con `2001:db8:100::1/48`.
* El `ping ipv6` debe responder con `0% packet loss`.

---

## 3. Ejercicio 1: Adyacencia eBGPv6 y Políticas RFC 8212 (25 min)

### Fase A: Levantamiento de la sesión MP-BGPv6

Configura el proceso BGP en `R1-ISP` desactivando la familia IPv4 por defecto y activando explícitamente `ipv6 unicast`:

```text
configure terminal
router bgp 65001
 bgp router-id 1.1.1.1
 no bgp default ipv4-unicast
 neighbor 2001:db8:12::0 remote-as 65002
 !
 address-family ipv6 unicast
  neighbor 2001:db8:12::0 activate
  network 2001:db8:100::/48
 exit-address-family
end
```

Verifica el estado de la sesión:

```text
show bgp ipv6 unicast summary
```

**Análisis esperado:** Verás que la sesión TCP/BGP estableció (`Up/Down` con contador de tiempo activo), pero en las columnas `State/PfxRcd` y `PfxSnt` aparece **`(Policy)`**. Esto ocurre porque FRRouting implementa **RFC 8212**: por seguridad, ningún router eBGP intercambia prefijos hasta tener políticas explícitas de entrada y salida.

### Fase B: Aplicación de Políticas Seguras (`ipv6 prefix-list` + `route-map`)

Siguiendo las mejores prácticas de enrutamiento (MANRS / RFC 8212):
* **Salida (`OUT`):** Anunciamos estrictamente nuestro bloque agregado `/48`.
* **Entrada (`IN`):** Aceptamos prefijos globales válidos (`2000::/3`) con máscaras entre `/12` y `/48`, descartando rutas más específicas (`/49`–`/128`).

Aplica estas políticas en `R1-ISP`:

```text
configure terminal
ipv6 prefix-list PL-MIS-PREFIJOS seq 10 permit 2001:db8:100::/48
ipv6 prefix-list PL-RECIBIR-IN seq 10 permit 2000::/3 ge 12 le 48
!
route-map RM-TRANSIT-OUT permit 10
 match ipv6 address prefix-list PL-MIS-PREFIJOS
!
route-map RM-TRANSIT-IN permit 10
 match ipv6 address prefix-list PL-RECIBIR-IN
!
router bgp 65001
 address-family ipv6 unicast
  neighbor 2001:db8:12::0 route-map RM-TRANSIT-IN in
  neighbor 2001:db8:12::0 route-map RM-TRANSIT-OUT out
 exit-address-family
end
```

### Fase C: Verificación de tabla BGP y plano de datos

Ejecuta los siguientes comandos en `R1-ISP#`:

```text
show bgp ipv6 unicast summary
show bgp ipv6 unicast
ping ipv6 2001:db8:200::1 source 2001:db8:100::1
```

**Verificación:**
* En `show bgp ipv6 unicast summary`, `State/PfxRcd` y `PfxSnt` ahora muestran **`1`**.
* En `show bgp ipv6 unicast`, el prefijo `2001:db8:200::/48` tiene como `Next Hop` la dirección Global Unicast **`2001:db8:12::0`** (Next-Hop de **16 bytes**).
* El `ping ipv6` entre loopbacks responde con `0% packet loss`.

---

## 4. Ejercicio 2: Comprobación de la Regla de Originación BGP (10 min)

BGP nunca origina un prefijo si no existe una ruta con **máscara exacta** en la tabla de enrutamiento local (RIB).

1. Revisa tu tabla de enrutamiento actual en `R1-ISP#`:
   ```text
   show ipv6 route
   ```
   *(Observa que tienes `2001:db8:100::/48`, pero no tienes ninguna subred `/64`).*

2. Intenta anunciar una subred `/64` inexistente en BGP:
   ```text
   configure terminal
   router bgp 65001
    address-family ipv6 unicast
     network 2001:db8:100:1::/64
   end
   ```

3. Verifica tu tabla BGP:
   ```text
   show bgp ipv6 unicast
   ```
   **Observación:** El prefijo `2001:db8:100:1::/64` **no aparece** en la tabla BGP porque falla la regla de originación.

4. Limpia el comando de prueba:
   ```text
   configure terminal
   router bgp 65001
    address-family ipv6 unicast
     no network 2001:db8:100:1::/64
   end
   ```

---

## 5. Ejercicio 3: Peering por Link-Local (`fe80::`) y Next-Hop de 32 Bytes (20 min)

En enlaces punto a punto (especialmente en IXPs o enlaces directos), establecer BGP sobre direcciones **Link-Local (`fe80::`)** desacopla el plano de control del direccionamiento global y cumple con **RFC 2545** (envío de Next-Hop de 32 bytes: GUA + Link-Local).

### Paso 5.1: Descubrir la dirección Link-Local del vecino

En `R1-ISP#`, consulta la tabla de vecinos NDP (Neighbor Discovery):

```text
show ipv6 neighbors
```

Anota la dirección **`fe80::...`** asociada a `2001:db8:12::0` en la interfaz `eth1`. También puedes verificar cuál es tu propia Link-Local en `R1-ISP` ejecutando `show interface eth1`.

### Paso 5.2: Migrar el vecino en `R1-ISP` a Link-Local

Como las direcciones `fe80::` son significativas únicamente dentro de cada enlace físico, es obligatorio especificar `interface eth1`:

```text
configure terminal
router bgp 65001
 no neighbor 2001:db8:12::0
 neighbor <FE80_DE_R2> remote-as 65002
 neighbor <FE80_DE_R2> interface eth1
 !
 address-family ipv6 unicast
  neighbor <FE80_DE_R2> activate
  neighbor <FE80_DE_R2> route-map RM-TRANSIT-IN in
  neighbor <FE80_DE_R2> route-map RM-TRANSIT-OUT out
 exit-address-family
end
```

*(Reemplaza `<FE80_DE_R2>` por la dirección `fe80::` real de `R2-TRANSIT` que anotaste en el paso anterior).*

### Paso 5.3: Migrar el vecino en `R2-TRANSIT`

Abre la consola de **`R2-TRANSIT`** haciendo clic en el ícono de **Terminal (`>_`)** junto a `R2-TRANSIT` en la barra lateral de **Containerlab** y ejecutando `vtysh` (o bien ejecuta `docker exec -it clab-bloque1-ebgpv6-R2-TRANSIT vtysh`).

Configura la sesión apuntando a la dirección Link-Local de `R1-ISP`:

```text
configure terminal
router bgp 65002
 no neighbor 2001:db8:12::1
 neighbor <FE80_DE_R1> remote-as 65001
 neighbor <FE80_DE_R1> interface eth1
 !
 address-family ipv6 unicast
  neighbor <FE80_DE_R1> activate
  neighbor <FE80_DE_R1> route-map RM-ISP-IN in
  neighbor <FE80_DE_R1> route-map RM-ISP-OUT out
 exit-address-family
end
```

### Paso 5.4: Inspeccionar el atributo `MP_REACH_NLRI` (32 bytes)

Regresa a la pestaña de terminal de **`R1-ISP`** y analiza en detalle la ruta aprendida:

```text
show bgp ipv6 unicast 2001:db8:200::/48
```

**Verificación clave (RFC 2545):** En la sección `Nexthop` observarás **dos direcciones IPv6** enviadas dentro del mismo atributo `MP_REACH_NLRI` (32 bytes en total):
1. La dirección Global Unicast (`2001:db8:12::0` o `::`) — 16 bytes.
2. La dirección Link-Local (`fe80::...`) — 16 bytes, que es la utilizada efectivamente para resolver la capa 2 y reenviar los paquetes.

---

## 6. Cierre y Apagado del Laboratorio (5 min)

Para conservar tus horas gratuitas de GitHub Codespaces para las siguientes sesiones:

1. Si abriste el mapa web (`containerlab graph`), presiona `Ctrl + C` en esa terminal.
2. Destruye los contenedores del laboratorio:
   ```bash
   sudo containerlab destroy -t bloque1.clab.yml --cleanup
   ```
3. Abre `https://github.com/codespaces` en tu navegador, haz clic en los tres puntos (`...`) junto a tu Codespace activo y selecciona **Stop codespace**.
