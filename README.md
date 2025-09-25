# 🚀 RESTful API en FHIR con OAuth2: Un Laboratorio de Sistemas Distribuidos

Este repositorio contiene la configuración y la guía para desplegar un ecosistema seguro para la interoperabilidad en salud, utilizando **HL7 FHIR** para el estándar de datos, **HAPI FHIR** como servidor de persistencia, y **Keycloak** para la autenticación y autorización mediante **OAuth2**.

El objetivo de este laboratorio es demostrar cómo proteger un API de FHIR, permitiendo operaciones seguras (como la creación de un recurso `Patient`) solo a clientes autenticados con un token válido, y denegando el acceso a peticiones anónimas.

## 🛠️ Tecnologías Utilizadas

- **Docker & Docker Compose**: Para orquestar los servicios de la infraestructura.
- **HAPI FHIR**: Servidor de referencia para el estándar HL7 FHIR.
- **Keycloak**: Servidor de autenticación y autorización open-source.
- **PostgreSQL**: Base de datos para Keycloak.
- **Postman**: Para realizar las pruebas de la API.

---

## 📋 Guía de Implementación Paso a Paso

A continuación, se detalla el proceso para replicar el laboratorio.

### Paso 1: Levantamiento de la Infraestructura con Docker

El primer paso es iniciar todos los servicios necesarios utilizando Docker Compose. El archivo `docker-compose.yml` está configurado para levantar y conectar HAPI FHIR, Keycloak y su base de datos.

1.  Abre una terminal en la raíz del proyecto.
2.  Ejecuta el siguiente comando para construir e iniciar los contenedores en segundo plano:

    ```bash
    docker-compose up -d
    ```

3.  Una vez finalizado, tendrás los siguientes servicios corriendo:
    - **HAPI FHIR Server**: `http://localhost:8080`
    - **Keycloak Server**: `http://localhost:8081`

### Paso 2: Configuración de Keycloak para SMART on FHIR

Para que HAPI FHIR pueda delegar la autenticación, es necesario configurar un *realm*, un *cliente* y un *usuario* en Keycloak.

1.  **Accede a la Consola de Administración de Keycloak**:
    - Ve a `http://localhost:8081` y haz clic en `Administration Console`.
    - Inicia sesión con las credenciales:
      - **Usuario**: `admin`
      - **Contraseña**: `admin`

2.  **Crear un nuevo Realm**:
    - En la esquina superior izquierda, haz clic en `master` y luego en `Create Realm`.
    - **Realm name**: `fhir-realm`
    - Haz clic en `Create`.

3.  **Crear un Cliente**:
    - En el menú de la izquierda, selecciona `Clients` y luego `Create client`.
    - **Client ID**: `fhir-client`
    - **Name**: `HAPI FHIR Client`
    - Haz clic en `Next`.
    - Habilita la opción `Client authentication` y haz clic en `Next`.
    - En `Valid redirect URIs`, añade `*` por simplicidad para este laboratorio.
    - Haz clic en `Save`.

4.  **Obtener el Client Secret**:
    - Dentro de la configuración del cliente `fhir-client`, ve a la pestaña `Credentials`.
    - Copia el valor de `Client secret`. Lo necesitarás para obtener el token.

5.  **Crear un Usuario de Prueba**:
    - En el menú de la izquierda, ve a `Users` y haz clic en `Create user`.
    - **Username**: `testuser`
    - Haz clic en `Create`.
    - Ve a la pestaña `Credentials` del usuario y establece una contraseña (por ejemplo, `testpassword`). Desmarca la opción `Temporary` para que no expire.

### Paso 3: Probar la API con un Token Válido (POST /Patient)

Ahora, simularemos ser una aplicación cliente que obtiene un token de Keycloak y lo usa para crear un paciente en el servidor HAPI FHIR.

1.  **Obtener un Token de Acceso**:
    - Abre Postman y crea una nueva petición **POST** a la siguiente URL para obtener un token:
      ```
      http://localhost:8081/realms/fhir-realm/protocol/openid-connect/token
      ```
    - En la pestaña `Body`, selecciona `x-www-form-urlencoded` y añade los siguientes campos:
      - `grant_type`: `password`
      - `client_id`: `fhir-client`
      - `client_secret`: *<Pega el secret que copiaste de Keycloak>*
      - `username`: `testuser`
      - `password`: `testpassword`
    - Envía la petición. Recibirás una respuesta JSON con un `access_token`. Copia este token.

2.  **Crear un Recurso `Patient`**:
    - Crea una nueva petición **POST** en Postman a la URL del servidor FHIR:
      ```
      http://localhost:8080/fhir/Patient
      ```
    - En la pestaña `Authorization`, selecciona `Bearer Token` y pega el `access_token` que obtuviste.
    - En la pestaña `Body`, selecciona `raw` y `JSON`, y pega el siguiente cuerpo:
      ```json
      {
        "resourceType": "Patient",
        "name": [
          {
            "use": "official",
            "family": "Reyes",
            "given": ["Jaider"]
          }
        ]
      }
      ```
    - Envía la petición. Deberías recibir una respuesta `201 Created`, confirmando que el paciente fue creado exitosamente.

### Paso 4: Intentar Crear un Paciente sin Token

Finalmente, para verificar que la seguridad está funcionando, intentaremos realizar la misma operación sin proporcionar un token de autenticación.

1.  Usa la misma petición **POST** del paso anterior (`http://localhost:8080/fhir/Patient`).
2.  En la pestaña `Authorization`, selecciona `No Auth`.
3.  Envía la petición.

Como resultado, el servidor HAPI FHIR te devolverá un error `401 Unauthorized`, demostrando que el endpoint está correctamente protegido y no permite operaciones a clientes anónimos.