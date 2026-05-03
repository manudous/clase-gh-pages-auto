# 02 Github actions

En este ejemplo vamos a crear un servidor de producción usando Github pages y Github actions.

Partiremos de `03-github-branch`.

## Pasos para construirlo

`npm install` para instalar los paquetes del ejemplo anterior:

```bash
npm install
```

En vez de crear manualmente la rama `gh-pages` y subir los archivos de producción, vamos a automatizar el proceso de despliegue cada vez que hagamos un push a la rama `main`. Vamos a ver dos formas de hacerlo, una usando el paquete `gh-pages` y otra usando la acción oficial de Github `deploy-pages`.

Vamos a empezar con el enfoque de `gh-pages`.

Para ello, vamos a usar una librería, que no es oficial de Github, pero es muy popular para desplegar en Github pages.

Usaremos el mismo enfoque que en el ejemplo de `gh-pages`, pero utilizaremos [Github Actions](https://docs.github.com/en/free-pro-team@latest/actions) para despliegues automáticos.

Crea un repositorio nuevo y sube los archivos,

Nombre del repositorio: `clase-gh-pages-auto`

```bash
git init
git remote add origin git@github.com...
git add .
git commit -m "initial commit"
git push -u origin main
```

>> NOTA: Aquí asegurarse que todo los alumnos se hayan creado su propio repositorio y hayan subido los archivos.

Instala [gh-pages](https://github.com/tschaub/gh-pages) como dependencia de desarrollo para desplegar en Github pages:

poner en el buscador -d dist y así se localiza en el navegador donde sale en la documentación de gh-pages.

```bash
npm install gh-pages --save-dev
```

Primero lo vamos a probar aquí en local para asegurarnos que funciona correctamente y luego lo automatizaremos con Github actions.

Añade el comando de despliegue:

_./package.json_

```diff
  "scripts": {
    "start": "run-p -l type-check:watch start:dev",
    "start:dev": "vite --port 8080",
    "prebuild": "npm run type-check",
    "build": "vite build",
+   "prebuild:dev": "npm run prebuild",
+   "build:dev": "vite build --mode development",
+   "deploy": "gh-pages -d dist",
    ...
  },
```

Ejecuta la build de desarrollo y despliégala:

```bash
npm run build:dev
```

Aquí mirar los assets dentro de dist y buscar lemoncode y decir que estamos todavía en desarrollo.

```bashbash
npm run deploy
```

> NOTA: Podemos ejecutar el despliegue porque tenemos acceso al repositorio
>
> ya que hemos iniciado sesión en Github

Aquí es donde nos vamos a crear nuestra carpeta de workflow de Github actions, pero que es un workflow? Es un proceso automatizado que se ejecuta en respuesta a eventos específicos en tu repositorio, como un push a una rama o la creación de un pull request. Los workflows se definen en archivos YAML dentro del directorio `.github/workflows` de tu repositorio. Estos archivos describen los pasos que se deben seguir para ejecutar tareas específicas, como construir tu proyecto, ejecutar pruebas o desplegar tu aplicación.

¿Qué creeis que vamos a crear en nuestro workflow? Un proceso que se ejecute cada vez que hagamos un push a la rama `main` y que ejecute el comando de despliegue.

Añade el workflow de CD de GitHub actions:

_./.github/workflows/cd.yml_

```yml
name: CD workflow

on:
  push:
    branches:
      - main

jobs:
  cd:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

      - name: Install
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy
        run: npm run deploy
```

**Explicación de cada paso:**

- **`Checkout repository`**: Descarga el código del repositorio en la máquina virtual limpia donde corre el job. Sin este paso, la VM no tendría acceso al código fuente.
- **`Install`**: Ejecuta `npm ci` para instalar las dependencias. A diferencia de `npm install`, `npm ci` es más estricto y reproducible: borra `node_modules` y lo instala todo desde cero respetando exactamente el `package-lock.json`.
- **`Build`**: Compila la aplicación y genera los archivos estáticos de producción en la carpeta `dist`.
- **`Deploy`**: Usa el paquete `gh-pages` para subir el contenido de `dist` a la rama `gh-pages` del repositorio, que es la que GitHub Pages sirve como sitio web.

Añade un commit con los cambios:

```bash
git add .
git commit -m "add continuos deployment"
git push
```

Como vimos, el workflow falló. ¿Por qué? Porque cada vez que se ejecuta un job de Github, es una máquina nueva y limpia fuera del repositorio. Siempre los mayores problemas que vamos a tener en los despliegues automáticos es que el job no tiene permisos para hacer push al repositorio. Es decir, necesitamos permitir que el job haga git push. El mejor enfoque es crear una nueva clave ssh en local:

¿Sabéis lo que es una clave ssh? Es un par de claves, una pública y otra privada, que se utilizan para autenticar a los usuarios sin necesidad de usar contraseñas. La clave pública se añade a la configuración del repositorio (en este caso, en Github) y la clave privada se mantiene segura en tu máquina local. Cuando el job de Github intenta hacer push, utiliza la clave privada para autenticarse con la clave pública que hemos añadido a Github, permitiendo así el acceso al repositorio.

Aquí podéis ver la documentación oficial de Github sobre cómo crear una clave SSH: [Generating a new SSH key and adding it to the ssh-agent](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

```bash
ssh-keygen -t ed25519 -C "cd-user@my-app.com"
```

```bash
> Enter file in which to save the key: `./id_rsa`
> Enter passphrase (empty for no passphrase): `Pulse Enter for empty`
> Enter same passphrase again: `Pulse Enter for empty`
```
Se podría poner contraseña pero tened en cuenta que aqui no va a ver interacción humana, por lo que no se podría introducir la contraseña en el momento de hacer el push. Por eso, es recomendable dejar la passphrase vacía para que el proceso de despliegue pueda ejecutarse sin problemas.

> NOTES:
>
> -t ed25519: El algoritmo Ed25519 es una opción moderna y segura para claves SSH, que proporciona mejor seguridad y rendimiento en comparación con algoritmos más antiguos como RSA.
>
> Introduce `./id_rsa` para guardar los archivos en el directorio actual
>
> Puedes dejar vacío el campo de passphrase.

Copia el contenido de `id_rsa.pub` en la sección `Github Settings` > `Deploy keys`:

![01-public-ssh-key](./readme-resources/01-public-ssh-key.png)

![02-public-ssh-key](./readme-resources/02-public-ssh-key.png)

Nombre: SSH-PUBLIC-KEY

Elimina el archivo `id_rsa.pub`.

Copia el contenido de `id_rsa` en la sección `Github Settings` > `Secrets`:

![03-private-ssh-key](./readme-resources/03-private-ssh-key.png)

![04-private-ssh-key](./readme-resources/04-private-ssh-key.png)

Elimina el archivo `id_rsa`.

Ahora podemos usar esta clave privada ssh para hacer un commit/push en el job de Github:

_./.github/workflows/cd.yml_

```diff
name: CD workflow

on:
  push:
    branches:
      - main

jobs:
  cd:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

+     - name: Use SSH key
+       run: |
+         mkdir -p ~/.ssh/
+         echo "${{secrets.SSH_PRIVATE_KEY}}" > ~/.ssh/id_rsa
+         sudo chmod 600 ~/.ssh/id_rsa

+     - name: Git config
+       run: |
+         git config --global user.email "cd-user@my-app.com"
+         git config --global user.name "cd-user"

      - name: Install
        run: npm ci

      - name: Build
        run: npm run build

      - name: Deploy
-       run: npm run deploy
+       run: npm run deploy -- -r git@github.com:<your-repo>.git

```

Si hubiera usado gh-pages -r flag, no haría falta hacer nada más, pero como hemos usado un comando de npm, tenemos que usar -- porque sino npm no lo reconoce como un argumento para el comando de deploy.

> NOTES:
>
> Paso "Use SSH key": crea id_rsa con la clave privada ssh en la carpeta ssh por defecto y añade permisos de escritura.
>
> Paso "Deploy": actualiza el comando de despliegue con la URL SSH del repositorio.
>
> [gh-pages -r flag](https://github.com/tschaub/gh-pages#optionsrepo)

```bash
git add .
git commit -m "configure git cd-user permits"
git push
```

![05-open-gh-pages-url](./readme-resources/05-open-gh-pages-url.png)

## deploy-pages Github action

Como alternativa al paquete `gh-pages`, podemos usar [deploy-pages Github action](https://github.com/actions/deploy-pages).

Esta es una acción oficial de Github para desplegar sitios estáticos en Github pages y es más fácil de configurar que el paquete `gh-pages`.

Primero, necesitamos cambiar la configuración de `gh-pages` en nuestro repositorio:

![06-gh-pages-settings](./readme-resources/06-gh-pages-settings.png)

Después podemos actualizar nuestro workflow así:

_./.github/workflows/cd.yml_

```diff
name: CD workflow

on:
  push:
    branches:
      - main

jobs:
  cd:
    runs-on: ubuntu-latest
+   permissions:
+     pages: write
+     id-token: write
+   environment:
+     name: github-pages
+     url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Checkout repository
        uses: actions/checkout@v6

-     - name: Use SSH key
-       run: |
-         mkdir -p ~/.ssh/
-         echo "${{secrets.SSH_PRIVATE_KEY}}" > ~/.ssh/id_rsa
-         sudo chmod 600 ~/.ssh/id_rsa

-     - name: Git config
-       run: |
-         git config --global user.email "cd-user@my-app.com"
-         git config --global user.name "cd-user"

      - name: Install
        run: npm ci

      - name: Build
        run: npm run build

+     - name: Upload artifact
+       uses: actions/upload-pages-artifact@v4
+       with:
+         path: dist

      - name: Deploy
+       id: deployment
-       run: npm run deploy -- -r git@github.com:nasdan/to-rm-gh-auto.git
+       uses: actions/deploy-pages@v4
```

Sube los cambios:

```bash
git add .
git commit -m "using deploy-pages Github action"
git push
```

> Usando esta acción, no necesitamos instalar el paquete `gh-pages` ni crear una clave SSH.

# Acerca de Basefactor + Lemoncode

Somos un equipo innovador de expertos en Javascript, apasionados por convertir tus ideas en productos robustos.

[Basefactor, consultancy by Lemoncode](http://www.basefactor.com) ofrece servicios de consultoría y coaching.

[Lemoncode](http://lemoncode.net/services/en/#en-home) ofrece servicios de formación.

Para la audiencia de LATAM/España ofrecemos un Máster Front End Online, más información: http://lemoncode.net/master-frontend
