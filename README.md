# EA-Seminari10-DevOps

Orquestacion del Seminario 10: un solo `docker compose` levanta la base de datos, la API con JWT y
WebSockets, el backoffice en Angular y el backoffice en React.

Cada aplicacion vive en su propio repositorio y publica su imagen en GitHub Container Registry en cada
push. Este repo no tiene codigo de aplicacion: solo compone esas imagenes.

## Servicios

| Servicio | Imagen | Puerto en tu maquina | Para que |
|---|---|---|---|
| `mongodb` | `mongo:8` | interno | Base de datos de la API |
| `api` | `ghcr.io/mbakkali28/ea-seminari10-devops-backend:latest` | `1337` | API REST + Socket.IO |
| `angular` | `ghcr.io/alikoubiiss/devop:latest` | `4200` | Backoffice en Angular |
| `react` | `ghcr.io/mbakkali28/ea-seminari10-devops-react:latest` | `4300` | Backoffice en React |

Los cuatro servicios comparten la red `seminario-net`, asi que la API habla con Mongo por el nombre
`mongodb`.

## Como arrancarlo

```
cp env.example .env      # y pon los dos secretos de los JWT
docker compose up -d
```

Despues:

- Backoffice Angular: <http://localhost:4200>
- Backoffice React: <http://localhost:4300>
- API: <http://localhost:1337>, documentacion en <http://localhost:1337/api-docs>

El compose **no arranca** si faltan `JWT_SECRET` o `JWT_REFRESH_SECRET` en el `.env`: es a proposito, para
que no se quede una API con un secreto de mentira.

Las imagenes de ghcr.io nacen **privadas**. Para que `docker compose up` las pueda bajar hay dos caminos:
o se ponen publicas (GitHub > Packages > el paquete > Package settings > Change visibility), o se hace
`docker login ghcr.io` antes con un token que tenga `read:packages`.

Los datos de ejemplo viven en el repo del backend (`src/seed-data.ts`). Como Mongo **no publica** el puerto
27017 al host, el seed se corre dentro de la red del compose, con el mismo `MONGO_DB` que el `.env` (aqui,
`seminari10`):

```
docker run --rm --network seminario10_seminario-net \
  -v "$PWD/../EA-Seminari10-DevOps-Backend:/app" -v /app/node_modules -w /app \
  -e JWT_SECRET=cualquier-cosa-de-32-caracteres -e JWT_REFRESH_SECRET=otra-cosa-de-32-caracteres \
  node:22-bookworm-slim sh -c 'npm ci && MONGO_URL=mongodb://mongodb:27017/seminari10 npm run seed'
```

El seed solo inserta si la base esta vacia; para rehacerla, `npm run seed -- --reset` (borra usuarios,
autores y libros de esa base). Los dos secretos se pasan solo porque la config del backend los exige al
arrancar, el seed no los usa.

Los scripts `seed` y `migrate-indexes` no van dentro de la imagen (necesitan `tsx`, que es dependencia de
desarrollo).

## Datos y persistencia

Mongo guarda sus datos en el volumen de Docker `mongo-data`, montado en `/data/db`. El volumen sobrevive a
`docker compose down` y a recrear el contenedor, asi que los datos no se pierden al volver a levantar el
compose. Solo se borran si se ejecuta `docker compose down -v` (o `docker volume rm seminario10_mongo-data`).

Usuarios de ejemplo para entrar en los backoffices, con la contraseña `seminari7`:

| Correo | Rol |
|---|---|
| `admin@example.com` | `admin` |
| `user@example.com` | `user` |

Para rehacer los datos de ejemplo: `docker compose down -v`, volver a levantar el compose y ejecutar el
seed de arriba.

## Local o produccion

Las imagenes, los puertos y la red estan escritos en el `docker-compose.yml`. Lo unico que se cambia
fuera de el, en el `.env`, es la base de datos y los secretos:

| Que quieres | Que cambias en `.env` |
|---|---|
| Otra base de datos | `MONGO_DB` |
| Otro origen permitido | `CORS_ORIGIN`. El middleware de CORS acepta **un** origen: en local, con los dos backoffices delante, se deja en `*`; en produccion se pone el origen publico |
| Otros secretos o duracion de los tokens | `JWT_SECRET`, `JWT_REFRESH_SECRET`, `JWT_EXPIRES_IN`, `JWT_REFRESH_EXPIRES_IN` |

Las URLs que usan los backoffices **no** estan aqui: van compiladas dentro de cada imagen. Para apuntar la
app a otro servidor se edita el fichero de configuracion de su repo
(`src/environments/environment.prod.ts` en Angular, `src/config/environment.production.ts` en React), se
commitea, y el workflow de ese repo reconstruye y publica la imagen en ese push.

**La API es distinta: se cambia sin reconstruir la imagen.** El backend lee su configuracion del entorno
cada vez que arranca, asi que para cambiar `MONGO_DB`, `CORS_ORIGIN` o los secretos basta con editar el
`.env` y volver a levantar los contenedores:

```
docker compose up -d
```

Compose detecta los valores nuevos y recrea el contenedor de la API con ellos. No hay que construir nada
ni bajar otra imagen: la misma imagen sirve para cualquier entorno.
