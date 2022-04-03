This is just a translation from the [original guide](https://github.com/t0mmysm1th/embassy-os/blob/master/BuildGuide.md)
Esta es solo una traduccion de la [guia original](https://github.com/t0mmysm1th/embassy-os/blob/master/BuildGuide.md)

##### Notas Iniciales & Recomendaciones
* Debido a los problemas para compilar la imagen desde la computadora, esta guia te llevara paso a paso a traves del proceso de compilacion de EmbassyOS directamente en una Raspberry Pi 4 (4GB u 8GB)
* Este proceso sera mas rapido si tienes una unidad SSD/NVMe USB disponible.
* Esta guia **no** requiere una tarjeta microSD de gran capacidad, especialmente si montaras la imagen final en una unidad SSD/NVMe.
* Conocimiento basico de la linea de comandos en terminal de Linux es recomendado.
* Sigue esta guia cuidadosamente y no te saltes ningun paso.

# :hammer_and_wrench: Guia de Compilation
1. Flashea [Raspberry Pi OS Lite](https://www.raspberrypi.org/software/operating-systems/) a una tarjeta microSD y configuta tu raspi para iniciar desde una unidad USB SSD/NVMe.
   1. Luego de flashear la imagen, crea un documento vacio llamado `ssh` en la particion `boot` de la tarjeta microSD, luego procede a iniciar el raspi con la tarjeta microSD flasheada (revisa tu router para obtener la direccion IP asignada a tu raspi)
   1. Realiza la actualizacion normal al inicio
      ```
      sudo apt update
      sudo raspi-config
      ```
   1. Cambia `Advanced Options->Boot Order`
   1. Selecciona `USB Boot` *(tratara de iniciar desde la tarjeta microSD primero si esta disponible)*
   1. Selecciona `Finish`, luego `Yes` para reiniciar
   1. Despues de reiniciar, `sudo shutdown now` para apagar el raspi y remover la tarjeta microSD
  
2. Flashea el *Raspi OS Lite* (del paso 1) a tu unidad SSD/NVMe.
   > :information_source: No te preocupes hacerca del tamano de la particion rootfs (raspi la aumentara al arranque inicial)
   
   > :information_source: Cada vez que re-flashees tu unidad SSD/NVMe necesitas iniciar con una tarjeta microSD y cambiar el *Boot order* denuevo

   1. No olvides crear un archivo `ssh` vacio
   1. Conecta la unidad SSD/NVMe (recuerda remover la tarjeta microSD) al raspi e inicialo
   1. Usa `sudo raspi-config` para cambiar la contrasena predeterminada
   1. Opcional: `sudo apt upgrade`
   1. Opcional: `sudo nano /etc/apt/sources.list.d/vscode.list` comenta la ultima linea cual contiene `packages.microsoft.com`

3. Instala GHC
   ```
   sudo apt update
   sudo apt install ghc
   
   #prueba:
   ghc --version
   
   #el comando de arriba deberia dar como ejemplo:
   The Glorious Glasgow Haskell Compilation System, version 8.4.4
   ```

4. Compila Stack:
   1. Instala Stack v2.1.3
      ```
      cd ~/
      wget -qO- https://raw.githubusercontent.com/commercialhaskell/stack/v2.1.3/etc/scripts/get-stack.sh | sh
      
      #prueba con
      stack --version
      
      #el comando de arriba deberia dar como ejemplo:
      Version 2.1.3, Git revision 636e3a759d51127df2b62f90772def126cdf6d1f (7735 commits) arm hpack-0.31.2
      ```
    
   1. Usa el Stack actual para compilar Stack v2.5.1:
      ```
      git clone --depth 1 --branch v2.5.1 https://github.com/commercialhaskell/stack.git
      cd stack

      #Construye del codigo fuente (>=3.5h total... Quizas necesites usar VPN para evitar problemas de timeout)
      #Nota: Repite el mismo comando si aparece algun mensaje "ConnectionFailure"
      stack build --stack-yaml=stack-ghc-84.yaml --system-ghc
      
      #Instala
      stack install --stack-yaml=stack-ghc-84.yaml --system-ghc
      export PATH=~/.local/bin:$PATH
      ```

5. Clona EmbassyOS & trata de *make* el `agent`:
   1. Primer intento
      > :information_source: La primera vez que ejecutes **make** va a resultar en un error
      
      ```
      sudo apt install llvm-9 libgmp-dev
      cd ~/
      git clone https://github.com/Start9Labs/embassy-os.git
      cd embassy-os/
      make agent
      ```
      > :memo: Esto instalara ghc-8.10.2, luego tratara de compilar pero dara error (en el paso siguiente lidiamos con los errores)
   1. Confirma la informacion de tu cpu
      ```
      cat /proc/cpuinfo | grep Hardware
      ```
   1. Si tu "Hardware" es [BCM2711](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2711/README.md) entonces:
      1. Cambia `C compiler flags` a `-marm -mcpu=cortex-a72` en la configuracion de GHC:
         ```
         nano ~/.stack/programs/arm-linux/ghc-8.10.2/lib/ghc-8.10.2/settings
         ```
   1. Para prevenir errores en gcc eliminamos la carpeta `setup-exe-src`
      ```
      rm -rf ~/.stack/setup-exe-src/
      ```

6. Instala los requisitos para el paso 7
   1. Instala NVM
      ```
      cd ~/ && curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
      #cierra y reconnecta al raspi y prueba:
      nvm --version
      ```
   1. Instala Node.js & NPM
      ```
      nvm install node
      ```
   1. Instala Ionic CLI
      ```
      npm install -g @ionic/cli
      ```
   1. Instala Dependencias
      ```
      sudo apt-get install -y build-essential openssl libssl-dev libc6-dev clang libclang-dev libavahi-client-dev upx ca-certificates
      ```
   1. Instala Rust
      ```
      cd ~/ && curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs -o- | bash
      
      #Elije la opcion 1
      source $HOME/.cargo/env

      #Revisa las versiones de rust & cargo 
      rustc --version
      cargo --version
      ```

7. Finalmente, llegamos a construir la **.img**
   1. En este punto, tienes un entorno de desarrollo que funciona para construir tu **embassy.img**.
      Antes de hacer eso puedes elegir habiliar el inicio de session SSH para el usuario `pi` en caso de que algo vaya mal o simplemente salta al siguiente paso
      ```
      cd ~/embassy-os
      sed -e '/passwd -l pi/ s/^#*/#/' -i setup.sh
      ```
      > :warning: La contrasena predeterminada para el usuario `pi` es `raspberry`, cambiala en tu proximo acceso.
   1. Construye la `embassy.img`
      ```
      cd ~/embassy-os
      make
      
      #Dependiendo de tu hardware, esto puede tomar 1-2h+
      #Espera el mensaje "DONE!" y anota la llave de tu producto product_key
      exit
      ```
8. Flashea la `embassy.img` a una tarjeta microSD
   1. Copia `embassy.img` desde el raspi a tu computadora con scp (secure copy)
      ```
      scp pi@raspi_IP:~/embassy-os/embassy.img .
      ```
   1. Conectate a tu raspi denuevo para hacer `sudo shutdown now`, luego de un apagado completo, desconnecta tu unidad SSD/NVMe
   1. Flashea `embassy.img` a una tarjeta microSD (hace esto antes de flashearlo a una unidad SSD/NVMe, para asegurarnos de que funciona)

9. Prepara para la configuracion inicial
   1. Inicia el raspi utilizando la tarjeta microSD flasheada
   1. Luego de unos minutos, el raspi deberia reinicarse automaticamente y hacer su primer [sonido](#embassy-sounds-explained).
      > :information_source: Si es necesario, puedes revisar el log `agent` con: `journalctl -u agent -ef`
   1. Procede con el [proceso de configuracion inicial de EmbassyOS](https://docs.start9labs.com/user-manual/initial-setup.html)
   1. Si todo salio correcto, puedes flashear `embassy.img` a una unidad SSD/NVMe de manera segura y repetir el paso 9

### Embassy Sonidos explicados
Sonido :notes: | Indicando
------- | --------
Bep | Dispositivo esta iniciando
Campana | Dispositivo listo para configuar
Mario "Moneda" | EmbassyOS ha empezado
Mario "Muerte" | Dispositivo esta apunto de Apagar/Reiniciar
Mario "Encendido" | EmbassyOS secuencia de actualizacion
Beethoven | Actualizacion fallida :(
