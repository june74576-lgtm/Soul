# Soul

<div align="center">
  <link rel="icon" type="image/svg+xml" href="data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 200 200'%3E%3Ccircle cx='85' cy='115' r='45' fill='none' stroke='%234B0082' stroke-width='10'/%3E%3Ccircle cx='115' cy='85' r='45' fill='none' stroke='%23FFFFFF' stroke-width='10'/%3E%3C/svg%3E" />
  <p><em>Your identity in cards</em></p>
</div>

**Soul** es una aplicación web para crear y compartir tu identidad digital en formato de tarjetas. Perfil personalizable con banner, avatar, bio con HTML y redes sociales, más una integración completa con **Spotify** que muestra tu música en tiempo real: top artistas, top canciones, playlists, artistas seguidos y la canción que estás escuchando ahora.

---

## Características

- **Autenticación con email/contraseña** (backend propio + Supabase Auth).
- **Perfil personalizable**:
  - Banner y avatar subibles (con visor lightbox).
  - Bio con **soporte HTML** (sanitizado con DOMPurify).
  - Redes sociales: Instagram, TikTok, Discord, GitHub, Telegram, Facebook.
  - Fondo general personalizado.
- **Integración con Spotify** (OAuth 2.0):
  - **Now Playing** en tiempo real (actualizado cada 2 segundos).
  - Top artistas y canciones del mes.
  - Canciones guardadas ("Recent Likes").
  - Artistas seguidos, álbumes guardados y playlists propias — en carruseles.
  - Color de acento dinámico según la portada de la canción actual.
- **Perfil público compartible** mediante URL: `profile.html?user=username` — sin necesidad de login.
- **Share Profile**: copia el enlace al portapapeles con un clic.
- **Panel de Settings** para editar username, banner, avatar, fondo, redes y bio desde un solo lugar.
- **Vista previa en vivo** de la bio mientras la escribes.
- **Diseño glassmorphism** con tema oscuro, animaciones suaves y layout responsive.

---

## Stack

| Capa | Tecnología |
|------|------------|
| Frontend | HTML5 + CSS3 + JavaScript (vanilla) |
| Backend | Node.js + Express 5 |
| Base de datos / Storage | [Supabase](https://supabase.com/) (Postgres + Storage) |
| Auth | Supabase Auth + JWT propio |
| Integración musical | Spotify Web API (OAuth 2.0) |
| Sanitización | DOMPurify |
| Iconografía | SVG inline |

---

## Estructura del proyecto

```
soul/
├── index.html          # Login / Register
├── profile.html        # Perfil (privado o público según ?user=)
├── script.js           # Lógica del frontend
├── styles.css          # Estilos (Material You dark + glassmorphism)
├── server.js           # Backend Express (auth, perfiles, Spotify)
├── package.json
├── logo.svg            # Dos círculos superpuestos (morado + blanco)
└── .env                # Variables de entorno (NO commitear)
```

---

## Puesta en marcha

### 1. Clona e instala

```bash
git clone https://github.com/tu-usuario/soul.git
cd soul
npm install
```

### 2. Configura las variables de entorno

Crea un archivo `.env` en la raíz:

```env
SUPABASE_URL=https://tu-proyecto.supabase.co
SUPABASE_ANON_KEY=tu-anon-key
SUPABASE_SERVICE_ROLE_KEY=tu-service-role-key
JWT_SECRET=una_clave_larga_y_segura
PORT=8888
```

> ⚠️ El `SERVICE_ROLE_KEY` solo va en el backend. Nunca lo expongas al cliente.

### 3. Configura Supabase

**Tabla `profiles`:**

| Columna | Tipo | Notas |
|---------|------|-------|
| `user_id` | uuid (FK → auth.users) | PK |
| `username` | text | único |
| `bio` | text | HTML permitido |
| `banner_url` | text | |
| `avatar_url` | text | |
| `background_url` | text | |
| `social_links` | jsonb | `{ "github": "user", ... }` |
| `spotify_connected` | boolean | default `false` |
| `spotify_token` | text | |
| `spotify_refresh_token` | text | |
| `updated_at` | timestamptz | |

**Bucket de Storage:** `images` (público, máx. 5 MB por archivo).

### 4. Configura Spotify

1. Ve a [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
2. Crea una app y agrega como **Redirect URI**:
   ```
   https://tu-backend.onrender.com/callback
   ```
3. Copia el **Client ID** y **Client Secret** y pégalos en `server.js`:
   ```js
   const spotifyClientId = 'TU_CLIENT_ID';
   const spotifyClientSecret = 'TU_CLIENT_SECRET';
   const spotifyRedirectUri = 'https://tu-backend.onrender.com/callback';
   ```

### 5. Ajusta el frontend

En `script.js`, cambia la URL del backend:

```js
const API_URL = 'https://tu-backend.onrender.com';
```

Y en `server.js`, actualiza el `redirect` del callback a la URL de tu frontend:

```js
res.redirect('https://tu-frontend.vercel.app?spotify=connected');
```

### 6. Arranca

```bash
npm start
```

Abre `http://localhost:8888` para la API, y sirve los archivos estáticos (`index.html`, `profile.html`, etc.) desde cualquier servidor web o plataforma (Vercel, Netlify).

---

## Cómo funciona

### Autenticación

1. El usuario se registra con email + contraseña.
2. El backend crea el usuario en **Supabase Auth** y un registro en `profiles`.
3. Se firma un **JWT propio** (7 días) que el frontend guarda en `localStorage`.
4. Cada petición protegida viaja con `Authorization: Bearer <token>` y el backend lo verifica con `verifyToken`.

### Spotify OAuth

1. El usuario pulsa **Connect** → el backend genera la URL de autorización con `state = userId`.
2. Spotify devuelve el `code` al endpoint `/callback`.
3. El backend intercambia el `code` por `access_token` + `refresh_token` y los guarda en `profiles`.
4. Cada consulta a la API usa `spotifyRequest()`, que refresca el token automáticamente si expira (401).
5. El **Now Playing** se actualiza cada 2 s con `/currently-playing` (usuario propio) o `/public-nowplaying/:username` (perfil público).

### Perfil público

Cualquier perfil puede compartirse con `profile.html?user=username`. El frontend detecta el parámetro y:

- Llama a `/public-profile/:username` para los datos básicos.
- Llama a `/public-spotify/:username` para los datos musicales (con caché de 30 s).
- Oculta todos los botones de edición (`data-public="true"` en `<body>`).

### Caché de Spotify

Las respuestas de `/public-spotify/:username` se cachean 30 segundos para evitar el rate limit de la API de Spotify. Si la API falla, se devuelve el caché expirado como *fallback*.

---

## Paleta de colores

| Variable | Valor |
|----------|-------|
| `--bg` | `#0A0A0E` |
| `--surface-1` | `#1A1A24` |
| `--surface-2` | `#222230` |
| `--accent` | `#C6B8FF` |
| `--text-primary` | `#F0F0F5` |
| `--text-secondary` | `#90909E` |

El color de acento del reproductor (`--track-accent`) se calcula dinámicamente a partir de la portada de la canción que suena.

---

## Endpoints principales

| Método | Ruta | Descripción |
|--------|------|-------------|
| `POST` | `/auth/register` | Crear cuenta |
| `POST` | `/auth/login` | Iniciar sesión |
| `GET`  | `/auth/me` | Perfil del usuario autenticado |
| `PUT`  | `/profile` | Actualizar username / bio / social_links |
| `POST` | `/profile/upload` | Subir banner, avatar o fondo |
| `GET`  | `/public-profile/:username` | Perfil público sin login |
| `GET`  | `/spotify/connect` | Iniciar flujo OAuth de Spotify |
| `POST` | `/spotify/disconnect` | Desconectar Spotify |
| `GET`  | `/currently-playing` | Canción actual (auth) |
| `GET`  | `/all-spotify-data` | Todos los datos de Spotify (auth) |
| `GET`  | `/public-spotify/:username` | Todos los datos (público, con caché) |
| `GET`  | `/public-nowplaying/:username` | Solo la canción actual (público) |

---

## Limitaciones conocidas

- El username no se valida contra duplicados con mensaje claro (el error de Supabase se propaga).
- La API de Spotify tiene rate limit — de ahí el caché de 30 s en modo público.
- Los archivos subidos se guardan con nombre `<userId>_<timestamp>.<ext>`; no hay limpieza automática de huérfanos.
- No hay verificación de email al registrarse.
- El `background_url` solo funciona si el usuario lo sube; el fondo por defecto es el gradiente radial.

---

## Ideas para futuras versiones

- Buscador de perfiles públicos.
- Comentarios o "me gusta" en perfiles.
- Temas de color predefinidos + color picker.
- PWA con modo offline.
- Previsualización Open Graph al compartir el perfil.
- Rate limiting en el backend.
- Job de limpieza de imágenes huérfanas en Storage.

---

## Licencia

Proyecto personal. Úsalo, modifícalo y hazlo tuyo — pero respeta las credenciales de Spotify y no subas tu `.env` al repositorio.
Hecho con ❤︎
