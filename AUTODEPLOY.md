# Autodeploy desde GitHub Actions a Servidor (Dinahosting)

En este manual crearemos una conexión segura desde GitHub hacia el servidor para que, al hacer *push* al repositorio, se realice el despliegue automáticamente mediante **GitHub Actions**.

## 1. Configuración de variables en GitHub

Vamos al panel de GitHub, entramos en nuestro repositorio y navegamos a:
`Settings` > `Secrets and Variables` > `Actions`.

Pulsamos sobre `New repository secret` y creamos estos 3 parámetros:

* **`SSH_USER`**: Nombre de usuario con el cual se accede por SSH al hosting.
* **`SSH_HOST`**: URL o IP del servidor de destino.
* **`SSH_PRIVATE_KEY`**: Contenido de la llave privada. La generaremos directamente en el hosting para asegurar la compatibilidad.

### Generación de llaves en el servidor

Ejecuta el siguiente comando en la terminal de tu hosting:
  
```bash
ssh-keygen -t rsa -b 4096 -C "deploy-github"
```

> [!WARNING]
> **No añadir contraseña (passphrase)**; de lo contrario, la conexión fallará ya que el proceso automatizado no podrá introducirla.
  
Ahora autorizaremos la llave en el servidor para permitir el acceso SSH:
  
```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```
Finalmente, obtenemos la llave privada para pegarla en el Secret de GitHub:
  
```bash
cat ~/.ssh/id_rsa
```
Copia todo el texto que aparece, incluyendo las líneas de inicio y fin:\
`-----BEGIN RSA PRIVATE KEY-----`\
...\
`-----END RSA PRIVATE KEY-----`

## 2. Creación del fichero de Workflow
En la raíz del proyecto local, crea el archivo en la siguiente ruta:\
`.github/workflows/deploy.yml`

Contenido del archivo (ajustado para Roots / Sage):

```YAML
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
            # Forzamos la carga del perfil de usuario para los paths
            [ -f ~/.bashrc ] && source ~/.bashrc
            [ -f ~/.bash_profile ] && source ~/.bash_profile

            # 1. Navegar al directorio raíz (Ajusta la ruta)
            cd /home/tu_usuario/www

            # 2. Pull del código
            git config pull.rebase false
            git pull origin main

            # 3. Instalación de dependencias de Bedrock (WordPress Core)
            php84 $(which composer) install --no-interaction --optimize-autoloader --no-dev

            # 4. Compilación del Tema Sage
            cd web/app/themes/nombre_del_tema
            
            php84 $(which composer) install --no-interaction --optimize-autoloader --no-dev
            
            if command -v npm &> /dev/null
            then
                npm install
                npm run build
            fi

            # 5. WP-CLI y Limpieza de caché
            cd /home/tu_usuario/www
            php84 $(which wp) cache flush
            php84 $(which wp) acorn view:clear
```

## 3. Prepara el hosting para el primer `git clone`
GitHub Actions ejecuta un `git pull`, el cual solo funciona si ya existe un repositorio Git vinculado en el servidor. Si tu carpeta `/www` aún no tiene Git, ejecuta estos comandos en el servidor **una única vez**:

```bash
cd /home/tu_usuario/www

# Inicializamos el repo
git init

# Renombramos la rama local a 'main'
git branch -m main

# Añadimos el remote (usa la URL de SSH de tu repo)
git remote add origin git@github.com:usuario/repo.git

# Validamos la conexión (escribe "yes" cuando pregunte)
git fetch origin

# Sincronizamos el contenido inicial
git reset --hard origin/main
```

> [!IMPORTANT]
> Dinahosting suele tener varias versiones de PHP. Tu script usa `php84`, así que verifica que el servidor la tiene disponible:\
> **Prueba de PHP**: Escribe `php84 -v`. Si te da la versión, perfecto.\
> **Prueba de Composer**: Escribe `which composer. Si te devuelve una ruta (ej: `/usr/local/bin/composer`), el script funcionará.

## 4. Activación del despliegue

Desde tu terminal local, sube el archivo del workflow:

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: añadir workflow de despliegue automático"
git push origin main
```

En cuanto hagas el push, GitHub detectará el archivo `.yml`y arrancará la "Action".

1. Ve a tu repositorio en la web de GitHub.
2. Haz clic en la pestaña "Actions" (en la barra superior).
3. Verás un flujo llamado "Deploy to Dinahosting" con un círculo amarillo (indicando que está en proceso).
4. Haz clic en el nombre del commit para ver los logs en tiempo real.



