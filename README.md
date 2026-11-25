# NN_AlertaMesh

## Descripción
AlertaMesh es una aplicación Android de mensajería de emergencia que funciona **sin Internet ni red celular**. Tras un sismo de gran magnitud, cuando las antenas se saturan o colapsan, los teléfonos cercanos se conectan directamente entre sí por **Bluetooth y Wi-Fi Direct** y forman una red en malla (mesh): cada teléfono envía, recibe y **reenvía** los mensajes de los demás, ampliando el alcance más allá de un solo enlace.

Características:
- Pantalla de ingreso con nombre y DNI (se guardan solo en el teléfono).
- Chat de grupo: los mensajes escritos por el usuario llegan a los demás teléfonos de la red.
- Botón **SOS**: los mensajes de auxilio salen siempre antes que los normales (cola de prioridad) y se muestran en rojo.
- Reenvío multisalto con control de duplicados y TTL.
- **Cifrado extremo a extremo**: intercambio de claves ECDH (P-256) entre teléfonos y cifrado AES-256-GCM con la clave de emisor de cada autor. El nombre, el DNI y el texto nunca viajan en claro y los nodos intermedios no pueden leerlos (ver la sección "Cifrado extremo a extremo").
- Pantalla **🔒 Seguridad**: contactos con el estado de su intercambio de claves, algoritmos, tu clave pública y una demostración del texto original frente a los bytes cifrados que viajan por el aire.
- Modo de bajo consumo que espacia la frecuencia de envío.
- Si no hay vecinos cerca, los mensajes esperan en cola y salen cuando aparece uno.

Proyecto del curso **Computación en Red III**. Versión 1.2 (prueba de concepto, TRL 3–4).

## Integrantes
- Alexandra Arce
- Farid Cardenas
- Jhrolan Puma
- Matías Valdivia

Universidad Católica de Santa María — Escuela Profesional de Ingeniería de Sistemas.

## Tecnologías utilizadas
- Kotlin 1.9 y Android SDK (minSdk 26, targetSdk 34)
- Google Nearby Connections API (`play-services-nearby`), que usa Bluetooth LE para descubrir y Bluetooth / Wi-Fi Direct para transferir
- Criptografía estándar de Java: ECDH P-256, AES-256-GCM, HMAC-SHA-256 y PBKDF2-HMAC-SHA256
- Gradle 8 con Android Gradle Plugin 8.5
- JUnit 4 (pruebas unitarias)

## Requisitos
- Windows, macOS o Linux, con JDK 17+, Git y **Android Studio** (Koala o superior) con Android SDK 34
- **Dos o más teléfonos Android físicos** (8.0 o superior) con **Google Play Services**. El emulador no sirve para probar la comunicación real, porque no tiene radios Bluetooth/Wi-Fi.
- En cada teléfono: Bluetooth y Wi-Fi activados, y Ubicación activada
- Cable USB y depuración USB activada en los teléfonos
- No requiere Internet para funcionar, ni base de datos, ni variables de entorno (no aplica `.env.example`). Solo necesita Internet la primera vez, para descargar dependencias con Gradle.

## Instalación y ejecución
```bash
git clone https://github.com/<usuario>/NN_AlertaMesh.git
```
1. Abrir la carpeta en Android Studio (File → Open) y esperar el Gradle Sync.
2. Conectar un teléfono por USB (con depuración USB activada) y pulsar **Run ▶**. Repetir en el segundo teléfono (cambiar el dispositivo en la barra superior y Run de nuevo).
3. En cada teléfono, aceptar los permisos de Bluetooth / dispositivos cercanos / ubicación.
4. En la pantalla de ingreso, escribir nombre, DNI (8 dígitos) y el **mismo código de red** en todos los teléfonos.

## Prueba de funcionamiento
1. Con ambos teléfonos ya dentro del chat, esperar unos segundos hasta que el estado diga **"● 1 vecino(s) conectado(s)"**.
2. Escribir un mensaje en el teléfono A y pulsar **Enviar**: debe aparecer en el teléfono B con el nombre y DNI del remitente.
3. Pulsar **SOS** en B: el mensaje llega a A resaltado en rojo.
4. Con un tercer teléfono, ubicado de modo que solo alcance a uno de los otros, comprobar el reenvío multisalto (el mensaje muestra el número de saltos).
5. Con un código de red distinto en un teléfono, ese teléfono no puede unirse ni leer los mensajes de la otra red (solo reenvía bytes cifrados).
6. **Cifrado extremo a extremo**: al conectarse dos teléfonos, la franja verde del chat pasa a "🔒 Cifrado extremo a extremo · 1 de 1 contacto(s) con clave". Pulsar **🔒 Seguridad** y comprobar que el contacto aparece como "🔒 … · clave intercambiada". Enviar un mensaje y revisar en la misma pantalla la **Demostración**: el texto original y los bytes cifrados que viajan por el aire.
7. Si un mensaje de un contacto llega alterado, no se muestra y aparece el aviso "⚠ Mensaje no verificado".

## Pruebas automáticas
```bash
./gradlew test          # Linux/macOS
gradlew.bat test        # Windows
```
Cubren prioridad SOS, multisalto, duplicados, TTL, mensajes malformados, modo de bajo consumo, cola sin vecinos, acuerdo de claves y cifrado/descifrado, y el protocolo extremo a extremo completo sobre una red simulada: intercambio de claves, mensajes que no se ven en claro en la radio, nodos con otro código de red, mensajes alterados o con prioridad cambiada, mensajes atribuidos a otro autor, reinicio de un contacto y anuncios repetidos.

## Estructura del proyecto
```
app/src/main/java/pe/edu/ucsm/alertamesh/
├── model/       MeshMessage, ChatPacket, WireView (formatos de mensaje)
├── crypto/      CryptoManager (ECDH, AES-256-GCM, HMAC, PBKDF2) y SecureChannel (protocolo extremo a extremo)
├── routing/     MeshRouter (multisalto, duplicados, TTL, cola SOS)
├── power/       PowerMode (normal / bajo consumo)
├── transport/   Transport, NearbyTransport (real), SimulatedNetwork (pruebas)
├── MeshNode.kt  Une transporte, enrutador y canal seguro
└── MainActivity.kt  Pantalla de ingreso, chat y pantalla de seguridad
app/src/test/    Pruebas unitarias
```

## Cifrado extremo a extremo
Implementado en `crypto/SecureChannel.kt` (esquema de "claves de emisor", como los grupos de Signal):

1. **Claves propias**: al arrancar, cada teléfono genera un par de claves ECDH (P-256) y una clave de emisor AES-256 aleatoria. Las claves privadas nunca salen del teléfono.
2. **HELLO (anuncio)**: al conectarse un vecino, el teléfono anuncia su clave pública. El anuncio va autenticado con un HMAC-SHA-256 derivado del código de red; un teléfono con otro código es ignorado.
3. **KEYSHARE (intercambio)**: cada contacto responde enviando su clave de emisor cifrada con la clave de sesión ECDH de la pareja. Ningún nodo intermedio la puede abrir, porque no tiene la clave privada de ninguno de los dos.
4. **CHAT**: cada mensaje (nombre, DNI y texto) se cifra con AES-256-GCM usando la clave de emisor de su autor. El autor y la prioridad (SOS) van como datos autenticados: si un nodo intermedio los cambia, el mensaje se rechaza.

Resultado: solo los teléfonos de la conversación leen los mensajes; la radio y los nodos que solo reenvían ven bytes ilegibles. Un teléfono con otro código de red reenvía los mensajes pero no puede unirse ni leerlos.

## Limitaciones conocidas
- La confianza en los contactos se basa en el código de red: quien lo conozca puede unirse a la red y, por tanto, leer sus mensajes. No se pide comparar huellas de clave (decisión de diseño del equipo).
- Las claves de cifrado se generan en cada arranque de la app (no se guardan); al reiniciar, los contactos reciben claves nuevas automáticamente. No hay renovación continua de claves (ratchet).
- Siguen visibles por radio los datos de enrutamiento: identificador aleatorio del nodo, prioridad y tamaño del mensaje.
- La app debe estar abierta (en primer plano) para participar en la malla.
- El modo de bajo consumo reduce la frecuencia de envío; el escaneo de radio lo administra Nearby Connections.
- El chat es de grupo (broadcast). Los mensajes directos y la ubicación GPS quedan como trabajo futuro.

## Seguridad
El repositorio no contiene contraseñas, tokens, certificados ni claves privadas. El DNI y el nombre se guardan solo en el teléfono y viajan únicamente dentro de mensajes cifrados de extremo a extremo. Lo único que se anuncia por radio es un identificador aleatorio y la clave pública (que es pública por diseño).
