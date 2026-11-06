# Integración Firebase — Splash

Fecha: 6 de noviembre de 2026 (America/Santiago)
Proyecto Firebase/GCP: `splash-8b08f`
Cuenta: `proyectosplash1@gmail.com`
Código de referencia: `D:\PROYECTOS\Splash` (repo [keiturra-ux/Splash](https://github.com/keiturra-ux/Splash))
Consola: https://console.firebase.google.com/project/splash-8b08f/overview

Este documento describe lo que ya quedó configurado en la plataforma y cómo integrarlo en el código. No sustituye una implementación: es la guía para hacerla sin pelearse con la arquitectura que el proyecto ya tiene.

## ¿Esto contempla el acceso con Google Auth?

Sí. En Firebase Authentication ya quedó activo el proveedor **Google Sign-In**, además de **email/contraseña**. La marca OAuth es "Splash" y el correo de soporte es `proyectosplash1@gmail.com`.

Eso es configuración de plataforma. Las apps todavía no usan Firebase: el login de cliente (`app/app/auth/login.tsx`) y el de trabajador (`app-trabajador/app/(auth)/login.tsx`) llaman a `POST /api/auth/login` con email y contraseña, y el backend responde con el JWT propio (15 minutos) más un refresh token (7 días).

El camino recomendado no es tirar ese JWT y reemplazar todo por sesiones de Firebase. Es usar Firebase Auth solo para el handshake de Google y, después, emitir el mismo JWT que el resto del sistema ya entiende (middleware `verificarToken`, Socket.IO, panel admin, roles CLIENTE / LAVADOR / ADMIN y el cierre de sesión con `sesionesInvalidadasEn`).

El login por teléfono quedó pendiente: en Android necesita el SHA-1 del keystore, y en este PC no había un `debug.keystore`.

## 1. Lo que ya está configurado en Firebase

### 1.1 Apps registradas

Los package names salen de los `app.json` del repo.

| App | Plataforma | Package / Bundle ID | App ID de Firebase |
| --- | --- | --- | --- |
| Splash | Android | `com.splash.app` | `1:5330024660:android:0fb1a98f7e271019a42341` |
| Splash | iOS | `com.splash.app` | `1:5330024660:ios:b783590e609a5842a42341` |
| Splash Trabajador | Android | `cl.splash.trabajador` | `1:5330024660:android:f49ca6e590394dc9a42341` |
| Splash Trabajador | iOS | `cl.splash.trabajador` | `1:5330024660:ios:8833ca6a1dc4fedba42341` |

Número de proyecto: `5330024660`.

Client IDs de OAuth que Firebase ya creó (van en la app; no son secretos):

- Web (necesario para obtener el `idToken` de Google en Android): `5330024660-1ltb39plae85q7j9acqo1ds14h1lbf4a.apps.googleusercontent.com`
- iOS Splash: `5330024660-7eq9b225ir40dev21555q3ssjjqer1u6.apps.googleusercontent.com`
- iOS Splash Trabajador: `5330024660-2dc6ku1jg95a4qkljg73mudcvn17rc73.apps.googleusercontent.com`

La API key de cada app no va en este documento. Se descarga desde [Ajustes del proyecto](https://console.firebase.google.com/project/splash-8b08f/settings/general): `google-services.json` (Android) y `GoogleService-Info.plist` (iOS).

### 1.2 Authentication

Proveedores activos:

- Email/contraseña
- Google Sign-In (habilitado, marca "Splash")

Dominios autorizados: `localhost`, `splash-8b08f.firebaseapp.com`, `splash-8b08f.web.app`.

Pendiente: Phone Auth (SHA-1 de Android) y la huella SHA-1/SHA-256 en cada app Android, sin la cual Google Sign-In en Android suele fallar con un error de API o de SHA-1.

### 1.3 Cloud Storage

- Bucket: `splash-8b08f.firebasestorage.app`
- Región: `southamerica-east1` (São Paulo), la más cercana a Chile
- Consola: https://console.firebase.google.com/project/splash-8b08f/storage

Reglas desplegadas:

- `evidencia/{lavadoId}/**`: lectura solo si hay usuario autenticado en Firebase; escritura denegada al cliente
- Cualquier otra ruta: lectura y escritura denegadas

La idea es que el backend suba las fotos con el Admin SDK (el mismo camino que hoy usa `POST /api/lavados/:id/foto`). Así se cumple la inmutabilidad de evidencia del protocolo SPL-EVF-001: la app no puede sobrescribir ni borrar.

### 1.4 Cloud Messaging (FCM)

Al registrar las apps, FCM queda preparado (en iOS aparece GCM habilitado). El backend hoy no habla con FCM: `backend/src/utils/notificaciones.ts` envía a `https://exp.host/--/api/v2/push/send` con tokens `ExponentPushToken[...]`. Se puede seguir así hasta un build nativo con EAS.

### 1.5 Lo que deliberadamente no se creó

No hay instancia de Realtime Database. El seguimiento en vivo ya está en Socket.IO (`lavador:ubicacion` → el servidor reenvía `lavado:ubicacion` al room `lavado_{id}` en `backend/src/server.ts`). Tampoco se habilitó Firestore: el backend usa Prisma (hoy SQLite) y el plan apunta a PostgreSQL.

## 2. Cómo está el código hoy

- Auth: `POST /api/auth/registro`, `/login`, `/refresh`, `/logout`, `/perfil` en `backend/src/controllers/authController.ts`. Passwords con `bcryptjs` (salt 12). Access token JWT de 15 minutos y refresh de 7 días.
- Middleware: `backend/src/middlewares/auth.ts` (`verificarToken`, `soloAdmin`, `soloCliente`, `soloLavador`, `soloLavadorAprobado`).
- Modelo `Usuario` en `backend/prisma/schema.prisma`: `email` único, `contrasena` obligatorio (hash), `rol` (`CLIENTE` | `LAVADOR` | `ADMIN`), `pushToken`, `sesionesInvalidadasEn`. No existe un campo `firebaseUid`.
- Cliente: `app/app/auth/login.tsx` llama `/auth/login` y solo deja pasar `rol === 'CLIENTE'`.
- Trabajador: `app-trabajador/app/(auth)/login.tsx` solo deja pasar `LAVADOR` o `ADMIN`.
- Fotos: `subirFoto` guarda un JPEG en `backend/uploads` y escribe `fotoAntesUrl` o `fotoDespuesUrl`. Un lavado solo admite una foto "antes" y una "después". Las fotos de incidencia aún no se persisten.
- Push: Expo Push, no Firebase.
- No hay SDK de Firebase en `app`, `app-trabajador` ni `backend`.

El plan del repo ubica Firebase Auth y FCM en la Fase 4. Storage no aparece con ese nombre: los docs hablan de Cloudinary o S3 para fotos. Storage es la alternativa nativa ya lista en este mismo proyecto GCP.

## 3. Arquitectura recomendada

Usar un flujo híbrido:

1. La app abre el selector de cuentas de Google.
2. Firebase Authentication valida esa identidad y entrega un `idToken`.
3. La app envía ese `idToken` a un endpoint nuevo del backend, por ejemplo `POST /api/auth/google`.
4. El backend verifica el token con Firebase Admin, busca o crea el `Usuario` de Prisma y responde con el JWT y el refresh token actuales.
5. A partir de ahí no cambia nada: `authStore`, `apiPost`, Socket.IO y el panel admin siguen igual.

Por qué no reemplazar el JWT por Firebase Auth de un día para otro:

- El middleware, el refresh, el logout y la invalidación de sesiones ya están hechos.
- El rol vive en Prisma, no en Firebase. Un lavador no puede entrar a la app de cliente, y al revés tampoco.
- El panel admin no está registrado como app Firebase y hoy autentica contra el mismo backend.

Email/contraseña de Firebase no hace falta activarlo en las apps si el registro actual (`POST /api/auth/registro`) se mantiene. En la plataforma quedó encendido por si más adelante se quiere unificar; no es un requisito para Google Sign-In.

## 4. Cómo integrar Google Sign-In

### 4.1 Prerrequisito: SHA-1 de Android

Sin esto, Google Sign-In en Android suele fallar. En este equipo no existía `%USERPROFILE%\.android\debug.keystore`.

Para el keystore de debug, cuando exista:

```bat
keytool -list -v -keystore %USERPROFILE%\.android\debug.keystore -alias androiddebugkey -storepass android
```

Para un build de EAS: `eas credentials` y copiar los fingerprints que muestre.

Esas huellas SHA-1 y SHA-256 hay que agregarlas en Firebase, en cada app Android (Splash y Splash Trabajador), en Ajustes del proyecto. Hace falta la de debug y, antes de publicar, la del keystore de producción.

En iOS hay que agregar el esquema URL `REVERSED_CLIENT_ID` que viene en `GoogleService-Info.plist` (también se puede leer en la consola). En Expo eso se declara en `app.json` / `app.config.js` dentro de `ios.infoPlist.CFBundleURLTypes`.

### 4.2 Archivos de configuración (cuando se integre, no antes)

No se copiaron al repo a propósito.

- App cliente Android: `google-services.json` de la app `com.splash.app`, normalmente en `app/google-services.json`, y en `app.json` la clave `"googleServicesFile": "./google-services.json"`.
- App trabajador Android: lo mismo en `app-trabajador/`, con el archivo de `cl.splash.trabajador`.
- iOS: `GoogleService-Info.plist` de cada bundle en la carpeta de cada app.

El `google-services.json` que entrega Firebase puede listar las dos apps Android del proyecto; el plugin elige la que coincide con el `applicationId`. Aun así, conviene bajar el archivo desde la ficha de cada app.

### 4.3 Dependencias

En cada app Expo (`app` y `app-trabajador`):

```bash
npx expo install firebase expo-auth-session expo-crypto expo-web-browser
```

En el backend:

```bash
npm install firebase-admin
```

El Admin SDK se inicializa una sola vez, con una cuenta de servicio descargada desde [Cuentas de servicio](https://console.firebase.google.com/project/splash-8b08f/settings/serviceaccounts/adminsdk). Esa clave no se commitea: va en una variable de entorno, por ejemplo `FIREBASE_SERVICE_ACCOUNT` (JSON) o `GOOGLE_APPLICATION_CREDENTIALS` (ruta al archivo, fuera del repo).

Ejemplo de inicialización:

```ts
import admin from 'firebase-admin';

if (!admin.apps.length) {
  admin.initializeApp({
    credential: admin.credential.cert(JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT!)),
    storageBucket: 'splash-8b08f.firebasestorage.app',
  });
}

export default admin;
```

### 4.4 Cambio en Prisma

Hoy `contrasena` es `String` obligatorio. Un usuario que entra solo con Google no eligió contraseña. Hay que:

- Agregar `firebaseUid String? @unique`
- Agregar algo como `proveedorAuth String @default("LOCAL")` (`LOCAL` | `GOOGLE` | `AMBOS` si más adelante se vinculan)
- Dejar `contrasena` opcional (`String?`) o guardar un hash aleatorio inutilizable si se prefiere no tocar tantos llamados

También conviene copiar `nombre` (y avatar, si viene) desde el perfil de Google en el primer alta.

### 4.5 Endpoint nuevo: `POST /api/auth/google`

Ruta pública, al lado de `/login` y `/registro` en `backend/src/routes/auth.ts`.

Cuerpo sugerido: `{ "idToken": "<token de Firebase Auth>" }`.

Lógica sugerida:

1. `admin.auth().verifyIdToken(idToken)`.
2. Exigir `decoded.email`. Sin email no se puede cruzar con `Usuario.email`.
3. Buscar usuario por `firebaseUid` y, si no aparece, por `email`.
4. Si no existe, crearlo con `nombre` de `decoded.name`, `email`, `firebaseUid`, `verificado` según `email_verified` y `rol` por defecto `CLIENTE` (un lavador no debería auto-registrarse como trabajador solo por apretar Google; el alta de lavador sigue el flujo que ya tenga el proyecto).
5. Si existe y no tiene `firebaseUid`, vincularlo (mismo email).
6. Si existe y el `firebaseUid` no coincide, rechazar.
7. Respetar `usuario.activo` igual que en el login actual.
8. Responder con el mismo JSON que `/login`: `token`, `refreshToken` y `usuario` (usar `generarAccessToken` y el refresh que ya existen).

Las apps siguen filtrando el rol después, igual que ahora: la de cliente exige `CLIENTE` y la de trabajador exige `LAVADOR` o `ADMIN`.

Esquema ilustrativo (no es un parche listo para pegar sin revisar tipos y validación Zod):

```ts
export async function loginGoogle(req: Request, res: Response) {
  const { idToken } = req.body ?? {};
  if (!idToken || typeof idToken !== 'string') {
    res.status(400).json({ error: 'idToken es obligatorio' });
    return;
  }

  let decoded: admin.auth.DecodedIdToken;
  try {
    decoded = await admin.auth().verifyIdToken(idToken);
  } catch {
    res.status(401).json({ error: 'Token de Google invalido' });
    return;
  }

  const email = decoded.email?.toLowerCase();
  if (!email) {
    res.status(401).json({ error: 'La cuenta de Google no trae un correo' });
    return;
  }

  let usuario = await prisma.usuario.findFirst({
    where: { OR: [{ firebaseUid: decoded.uid }, { email }] },
  });

  if (!usuario) {
    usuario = await prisma.usuario.create({
      data: {
        email,
        nombre: decoded.name || email.split('@')[0],
        firebaseUid: decoded.uid,
        verificado: !!decoded.email_verified,
        proveedorAuth: 'GOOGLE',
        contrasena: null,
      },
    });
  } else if (usuario.firebaseUid && usuario.firebaseUid !== decoded.uid) {
    res.status(401).json({ error: 'Esta cuenta no coincide con el proveedor de Google' });
    return;
  } else if (!usuario.firebaseUid) {
    usuario = await prisma.usuario.update({
      where: { id: usuario.id },
      data: { firebaseUid: decoded.uid, proveedorAuth: 'AMBOS' },
    });
  }

  if (!usuario.activo) {
    res.status(401).json({ error: 'Cuenta desactivada' });
    return;
  }

  const accessToken = generarAccessToken(usuario.id, usuario.rol, usuario.subrolAdmin);
  const refreshToken = generarRefreshToken(usuario.id);
  res.json({
    mensaje: 'Login exitoso',
    token: accessToken,
    refreshToken,
    usuario: formatUsuario(usuario),
  });
}
```

Si se deja `contrasena` como `String` obligatorio, en el `create` hay que guardar un hash aleatorio (por ejemplo `bcrypt.hash` de 32 bytes aleatorios) para que nadie pueda loguearse por `/auth/login` adivinando una contraseña vacía.

### 4.6 Cambio en las apps

Inicializar Firebase una sola vez (por ejemplo `src/services/firebase.ts`) con `projectId: "splash-8b08f"`, `authDomain: "splash-8b08f.firebaseapp.com"`, `storageBucket: "splash-8b08f.firebasestorage.app"`, `messagingSenderId: "5330024660"` y el `appId` de esa app. El `apiKey` se copia del `google-services.json` o de la consola, no de este documento.

Botón "Continuar con Google" en cada pantalla de login, usando `expo-auth-session`:

```ts
import * as Google from 'expo-auth-session/providers/google';
import * as WebBrowser from 'expo-web-browser';
import { GoogleAuthProvider, signInWithCredential, getAuth } from 'firebase/auth';

WebBrowser.maybeCompleteAuthSession();

const [request, response, promptAsync] = Google.useAuthRequest({
  webClientId: '5330024660-1ltb39plae85q7j9acqo1ds14h1lbf4a.apps.googleusercontent.com',
  iosClientId: 'ID_IOS_DE_ESTA_APP.apps.googleusercontent.com',
  androidClientId: 'revisar en la consola si aparece un cliente Android dedicado',
});
```

Cuando `response.type === 'success'`:

1. Tomar `response.authentication.idToken` (en Android el `webClientId` es lo que permite obtenerlo).
2. `signInWithCredential(getAuth(), GoogleAuthProvider.credential(idToken))`.
3. `const firebaseToken = await getAuth().currentUser.getIdToken()`.
4. `apiPost('/auth/google', { idToken: firebaseToken })`.
5. Si `resultado.ok`, hacer el mismo `iniciarSesion({ token, refreshToken, usuario })` y el mismo `router.replace('/(tabs)')` de siempre, incluyendo el filtro de rol.

En la app cliente, un usuario Google con rol distinto de `CLIENTE` debe ver el mismo mensaje que hoy: que los lavadores usan Splash Trabajador. En la app trabajador, al revés.

`iosClientId` de Splash: `5330024660-7eq9b225ir40dev21555q3ssjjqer1u6.apps.googleusercontent.com`.
`iosClientId` de Splash Trabajador: `5330024660-2dc6ku1jg95a4qkljg73mudcvn17rc73.apps.googleusercontent.com`.

### 4.7 Registro con Google

Si la persona no tiene cuenta, el endpoint de arriba la crea como `CLIENTE`. Eso sirve en la app de cliente. En la de trabajador no debería crear un lavador automáticamente: si no hay usuario con rol `LAVADOR` o `ADMIN`, la app muestra acceso denegado, igual que ahora. El alta de lavadores sigue el flujo de postulación o aprobación que ya existe.

## 5. Cómo integrar Storage para las fotos

Hoy `subirFoto` escribe en disco y guarda `/uploads/lavado-{id}-{tipo}-{timestamp}.jpg`. Eso no escala y no cumple SPL-EVF-001 (varias fotos por momento, inmutables, mínimo 1600 px, JPEG de hasta 1.5 MB).

Integración mínima, sin cambiar el contrato de la app:

1. Después de decodificar el base64, validar tipo, tamaño y (si se puede) dimensiones.
2. Subir con el Admin SDK a `evidencia/{lavadoId}/{tipo}/{timestamp}.jpg`.
3. Guardar en Prisma la URL del objeto (o una URL firmada de lectura).
4. Dejar de escribir en `backend/uploads` para evidencia nueva.

```ts
const bucket = admin.storage().bucket();
const dest = `evidencia/${id}/${tipo}/${Date.now()}.jpg`;
const file = bucket.file(dest);
await file.save(buffer, {
  contentType: 'image/jpeg',
  resumable: false,
  metadata: { cacheControl: 'private, max-age=0' },
});
```

Como las reglas niegan `write` al cliente, nadie desde la app puede pisar o borrar esa foto. El backend (Admin SDK) ignora esas reglas, que es el punto.

Para cumplir SPL-EVF-001 de verdad hace falta más que dos campos (`fotoAntesUrl`, `fotoDespuesUrl`): una tabla de evidencias con momento (M1–M6), tipo de lavado, juego de fotos (8 a 19) y hallazgos (2 fotos cada uno). Eso es un cambio de modelo, no solo de storage. Storage ya está listo para esa carpeta `evidencia/{lavadoId}/...`.

Avatares (`avatarUrl`) y `cedulaUrl` pueden ir a prefijos distintos (`perfiles/` , `verificacion/`) cuando se quiera; las reglas actuales los dejan cerrados al cliente, así que también tendrían que subir por el backend.

## 6. Cómo integrar FCM (más adelante)

No es bloqueante para Google Auth ni para Storage.

1. Seguir con Expo Push mientras se desarrolle en Expo Go.
2. En el build nativo con EAS, incluir `google-services.json` / `GoogleService-Info.plist`.
3. En iOS, subir una clave APNs de Apple Developer a Firebase Cloud Messaging. Sin eso el push nativo en iPhone no funciona.
4. Cuando se quiera dejar Expo Push, el backend pasaría a `admin.messaging().send(...)` y el campo `pushToken` guardaría un token FCM en lugar de un `ExponentPushToken`.

## 7. Lo que no hay que migrar a Firebase

- Ubicación del lavador: se queda en Socket.IO (`lavador:ubicacion`). No hace falta Realtime Database.
- Roles, lavados, pagos, disputas y el resto del dominio: se quedan en Prisma.
- El mapa de la app cliente usa MapLibre y el admin usa Leaflet. Google Maps Platform se configuró aparte (Maps, Geocoding, Routes, Places) y no se mezcla con este documento.

## 8. Orden sugerido de implementación

1. Agregar SHA-1/SHA-256 de Android en las dos apps.
2. Meter `firebase-admin` en el backend, variable de entorno de la cuenta de servicio y `POST /api/auth/google`.
3. Migración Prisma (`firebaseUid`, `contrasena` opcional o hash aleatorio).
4. Botón de Google en las dos pantallas de login, con el filtro de rol que ya existe.
5. (Opcional, distinto ticket) cambiar `subirFoto` para que use Storage.
6. (Fase 4 / build nativo) `google-services.json`, plist y APNs para FCM.

## 9. Pendientes de plataforma

- Cargar SHA-1 y SHA-256 en las apps Android (debug y release).
- Clave APNs para push iOS nativo.
- Restringir la API key de Maps con package name + SHA-1, cuando exista esa huella.
- Activar la verificación en dos pasos de `proyectosplash1@gmail.com` (Google la pidió antes del 10 de noviembre de 2026).
- Phone Auth, si se quiere el login por teléfono de la Fase 4.

## 10. Enlaces útiles

- Resumen del proyecto: https://console.firebase.google.com/project/splash-8b08f/overview
- Apps y descarga de configs: https://console.firebase.google.com/project/splash-8b08f/settings/general
- Authentication: https://console.firebase.google.com/project/splash-8b08f/authentication/providers
- Storage: https://console.firebase.google.com/project/splash-8b08f/storage
- Cloud Messaging: https://console.firebase.google.com/project/splash-8b08f/settings/cloudmessaging
- Credenciales de GCP: https://console.cloud.google.com/apis/credentials?project=splash-8b08f
