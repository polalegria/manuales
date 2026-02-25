# Autodeploy from GitHub Actions a Servidor
En este manual crearemos una conexión segura desde GitHub hacia el servidor para que al hacer commit al repo haga el deploy automaticamente desde GiHub Actions en nuestro Servidor / Hosting.

## 1. Creación de variables en GitHub
Vamos a nuestro panel de GitHub y entramos a:\
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
  `-----END RSA PRIVATE KEY-----`\





