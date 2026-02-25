# Autodeploy from GitHub Actions a Servidor
En este manual crearemos una conexión segura desde GitHub hacia el servidor para que al hacer commit al repo haga el deploy automaticamente desde GiHub Actions en nuestro Servidor / Hosting.

## 1. Creación de variables en GitHub
Vamos a nuestro panel de GitHub y entramos dentro de nuestro repositorio, una vez dentro vamos a:\
`Settings` > `Secrets and Variables` > `Actions`.

Pulsamos sobre `New Repository Secret` y creamos estos 3 parametros:

- `SSH_USER`: Hace referencia al nombre de usuario con el cual se accede por SSH al hosting.
- `SSH_HOST`: Hace referencia a la URL o IP del servidor al cual se quiere acceder.
- `SSH_PRIVATE_KEY`:\
  Esta es más compleja ya que la tenemos que generar nosotros desde SSH.\
  En este caso, la crearemos directamente en nuestro hosting con los siguientes comandos:
  
  ```
  ssh-keygen -t rsa -b 4096 -C "deploy-github"
  ```

  > [!CAUTION]
  > No añadir contraseña, no funcionara la conexión ya que no es compatible.
  
  Ahora le diremos al servidor que confie en esa llave para entrar por SSH:
  
  ```
  cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
  chmod 600 ~/.ssh/authorized_keys
  chmod 700 ~/.ssh
  ```
  Esto metera la `llave pública` en el listado de accesos permitidos.\
  Una vez ejecutado lo anterior vamos a obtener la llave publica para GitHub, ejecutamos esto en el servidor:
  
  ```
  cat ~/.ssh/id_rsa
  ```
  y copiamos todo el texto que aparece, incluyendo las líneas:\
  `-----BEGIN RSA PRIVATE KEY-----`\
  ...\
  `-----END RSA PRIVATE KEY-----`

## 2. Creación del fichero .yml
En nuestra raiz del proyecto o donde tengamos inicializado GIT, creamos el siguiente archivo:\
`.github/workflows/deploy.yml`

Dejo un ejemplo de mi deploy.yml, en mi caso es para un proyecto de Wordpress con Roots / Sage.

```
name: Deploy to Dinahosting

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy via SSH
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            # Forzamos la carga del perfil de usuario para tener los paths correctos
            [ -f ~/.bashrc ] && source ~/.bashrc
            [ -f ~/.bash_profile ] && source ~/.bash_profile

            # 1. Navegar al directorio raíz (Ajusta 'directorio' por tu usuario/carpeta real)
            cd /home/directorio/www

            # 2. Pull del código
            git config pull.rebase false
            git pull origin main

            # 3. Instalación de dependencias de Bedrock (WordPress Core)
            php84 $(which composer) install --no-interaction --optimize-autoloader --no-dev

            # 4. Compilación del Tema Sage
            # Ajustamos a la carpeta del tema actual (Ajusta 'nombre_del_theme' por su nombre real)
            cd web/app/themes/nombre_del_theme
            
            # Instalamos dependencias de PHP del tema
            php84 $(which composer) install --no-interaction --optimize-autoloader --no-dev
            
            # Instalamos dependencias de JS y compilamos assets
            # Usamos npm o yarn según lo que uses en este proyecto
            if command -v npm &> /dev/null
            then
                npm install
                npm run build
            fi

            # 5. WP-CLI y Limpieza de caché
            cd /home/directorio/www
            php84 $(which wp) cache flush
            php84 $(which wp) acorn view:clear
```

## 3. Prepara el hosting para el primer `git clone`
GitHub Actions está configurado para hacer un `git pull`, pero `git pull` **solo funciona si ya existe un repositorio de Git en la carpeta**.\

Si tu web en `/home/directorio/www` todavía no tiene Git, debes inicializarlo una única vez a mano:

```
cd /home/clinicadentalmartorell/www

# Si la carpeta tiene archivos pero no tiene .git:
git init

# Cambiamos el nombre de la rama local a 'main' para que coincida con GitHub
git branch -m main

# Añadimos el origen (sustituye con tu URL de SSH real)
git remote add origin git@github.com:usuario/repo.git

# Traer datos de GitHub (di que "yes" si te pregunta por la autenticidad del host)
git fetch origin

# Cuidado: esto alineará la carpeta con lo que hay en GitHub, solo aplica si ya tenias un repositorio o contenido
git reset --hard origin/main
```

> [!IMPORTANT]
> Dinahosting suele tener varias versiones de PHP. Tu script usa `php84`, así que verifica que el servidor la tiene disponible:
> **Prueba de PHP**: Escribe `php84 -v`. Si te da la versión, perfecto.
> **Prueba de Composer**: Escribe `which composer. Si te devuelve una ruta (ej: `/usr/local/bin/composer`), el script funcionará.

## 4. Primer "Push" de activación

Para que GitHub sepa que tiene que hacer el deploy, debes subir los cambios. Desde tu terminal local (donde tienes el proyecto):

```
# 1. Asegúrate de estar en la rama main
git checkout main

# 2. Añade el archivo del workflow si no lo habías hecho
git add .github/workflows/deploy.yml

# 3. Haz el commit
git commit -m "ci: añadir workflow de despliegue automático"

# 4. Sube los cambios
git push origin main
```

En cuanto hagas el push, GitHub detectará el archivo `.yml`y arrancará la "Action".

1. Ve a tu repositorio en la web de GitHub.
2. Haz clic en la pestaña "Actions" (en la barra superior).
3. Verás un flujo llamado "Deploy to Dinahosting" con un círculo amarillo (indicando que está en proceso).
4. Haz clic en el nombre del commit para ver los logs en tiempo real.








