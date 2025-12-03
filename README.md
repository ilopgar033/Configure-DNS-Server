### 2\. Guía paso a paso: Servidor DNS (Bind9) 🌐

Esta guía sigue el PDF `configuracion-servidor-dns.pdf`. Vamos a configurar un **Servidor Maestro** y sus zonas.

#### **Paso 0: Preparativos e Instalación** 📦

Entra en tu máquina servidor (por ejemplo `vagrant ssh server`).

1.  **Instala los paquetes necesarios:**

    Bash

    ```
    sudo apt update
    sudo apt install bind9 bind9utils bind9-doc

    ```

2.  **Modifica el modo IPv4:** Edita el archivo `/etc/default/named` para forzar el uso de IPv4 (más fácil para la práctica).

    Bash

    ```
    sudo nano /etc/default/named
    # Añade "-4" al final de OPTIONS:
    OPTIONS="-u bind -4"

    ```

* * * * *

#### **Paso 1: Configuración Global (Options)** ⚙️

Aquí definimos quién puede consultar y reenviadores.

1.  **Haz copia de seguridad** (¡Muy importante en examen!):

    Bash

    ```
    sudo cp /etc/bind/named.conf.options /etc/bind/named.conf.options.backup

    ```

2.  **Edita el archivo:**

    Bash

    ```
    sudo nano /etc/bind/named.conf.options

    ```

3.  **Contenido a incluir:** (Crea una lista de acceso `acl` al principio y configura las `options`).

    DNS Zone file

    ```
    // 1. Definimos quién es confiable (tu red interna)
    acl confiables {
        192.168.57.0/24;  // CAMBIA ESTO por tu red interna real
        127.0.0.0/8;      // localhost
    };

    options {
        directory "/var/cache/bind";

        // 2. Reenviadores (si no sabes la respuesta, pregunta a Google)
        forwarders {
            8.8.8.8;
            8.8.4.4;
        };

        // 3. Seguridad y escuchas
        listen-on port 53 { 192.168.57.10; }; // TU IP del Servidor
        allow-transfer { none; };             // Nadie puede copiar tu zona
        recursion yes;                        // Permitir recursión
        allow-recursion { confiables; };      // Solo a tus amigos (acl)
        dnssec-validation yes;

        // Comenta esta línea si no usas IPv6
        // listen-on-v6 { any; };
    };

    ```

4.  **Verifica la sintaxis:**

    Bash

    ```
    sudo named-checkconf /etc/bind/named.conf.options

    ```

    *(Si no sale nada, está bien)*.

* * * * *

#### **Paso 2: Declarar las Zonas (Local)** 📍

Aquí le dices al servidor: "Oye, yo soy el jefe (maestro) del dominio `micasa.es`".

1.  **Edita el archivo local:**

    Bash

    ```
    sudo nano /etc/bind/named.conf.local

    ```

2.  **Añade tus zonas (Directa e Inversa):** Supongamos que tu dominio es `micasa.es` y tu red `192.168.57.0`.

    DNS Zone file

    ```
    // ZONA DIRECTA (Nombre -> IP)
    zone "micasa.es" {
        type master;
        file "/var/lib/bind/db.micasa.es";  // Ruta absoluta recomendada
    };

    // ZONA INVERSA (IP -> Nombre)
    // Fíjate: la IP va al revés (57.168.192) y acaba en .in-addr.arpa
    zone "57.168.192.in-addr.arpa" {
        type master;
        file "/var/lib/bind/db.192.168.57"; // Ruta absoluta recomendada
    };

    ```

* * * * *

#### **Paso 3: Crear los Archivos de Zona** 📝

Ahora creamos los "listines telefónicos" reales.

**A) Zona Directa (`/var/lib/bind/db.micasa.es`)**

1.  Crea el archivo (puedes copiar uno de ejemplo, pero mejor escríbelo limpio):

    Bash

    ```
    sudo nano /var/lib/bind/db.micasa.es

    ```

2.  Contenido (¡Respeta los puntos finales! `.`):

    DNS Zone file

    ```
    ; BIND data file for micasa.es
    $TTL    86400
    @       IN      SOA     server.micasa.es. admin.micasa.es. (
                            1         ; Serial (incrementar si cambias algo)
                            604800    ; Refresh
                            86400     ; Retry
                            2419200   ; Expire
                            86400 )   ; Negative Cache TTL
    ;
    @       IN      NS      server.micasa.es.  ; Servidor de Nombres
    @       IN      A       192.168.57.10      ; IP del dominio micasa.es

    ; Registros de máquinas
    server  IN      A       192.168.57.10      ; El propio servidor
    c1      IN      A       192.168.57.25      ; Cliente 1 (si tiene IP fija)
    c2      IN      A       192.168.57.4       ; Cliente 2

    ```

**B) Zona Inversa (`/var/lib/bind/db.192.168.57`)**

1.  Crea el archivo:

    Bash

    ```
    sudo nano /var/lib/bind/db.192.168.57

    ```

2.  Contenido (Usamos PTR):

    DNS Zone file

    ```
    ; BIND reverse data file
    $TTL    86400
    @       IN      SOA     server.micasa.es. admin.micasa.es. (
                            1         ; Serial
                            604800    ; Refresh
                            86400     ; Retry
                            2419200   ; Expire
                            86400 )   ; Negative Cache TTL
    ;
    @       IN      NS      server.micasa.es.

    ; Punteros (Solo la última parte de la IP)
    10      IN      PTR     server.micasa.es.  ; 192.168.57.10
    4       IN      PTR     c2.micasa.es.      ; 192.168.57.4

    ```

* * * * *

#### **Paso 4: Verificación Final y Reinicio** ✅

1.  **Comprueba la zona directa:**

    Bash

    ```
    sudo named-checkzone micasa.es /var/lib/bind/db.micasa.es

    ```

    *Debe decir OK*.

2.  **Comprueba la zona inversa:**

    Bash

    ```
    sudo named-checkzone 57.168.192.in-addr.arpa /var/lib/bind/db.192.168.57

    ```

    *Debe decir OK*.

3.  **Reinicia el servicio:**

    Bash

    ```
    sudo systemctl restart named
    sudo systemctl status named

    ```

* * * * *

#### **Paso 5: Comprobación desde el Cliente** 💻

Vete a la máquina cliente (`c1` o `c2`).

1.  **Configura el DNS del cliente:** Si no lo has hecho por DHCP, edita temporalmente:

    Bash

    ```
    sudo nano /etc/resolv.conf
    # Añade:
    nameserver 192.168.57.10

    ```

2.  **Prueba con `dig` o `nslookup`:**

    Bash

    ```
    nslookup server.micasa.es
    # Debe devolver Address: 192.168.57.10

    dig @192.168.57.10 c2.micasa.es
    # Debe devolver la IP del cliente 2
    ```
