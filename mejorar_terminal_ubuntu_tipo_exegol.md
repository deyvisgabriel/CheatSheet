# Mejorar la terminal de Ubuntu como la predictiva de Exegol

Sí, puedes dejar la terminal de **Ubuntu** muy parecida a la de **Exegol**, especialmente con autocompletado predictivo, colores, historial inteligente y una barra más cómoda.

Lo más parecido es instalar:

- **Zsh**
- **Oh My Zsh**
- **Plugins predictivos**
- **Powerlevel10k** opcional

---

## 1. Instalar Zsh y dependencias

```bash
sudo apt update
sudo apt install zsh git curl wget -y
```

Verifica que Zsh esté instalado:

```bash
zsh --version
```

---

## 2. Instalar Oh My Zsh

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

Cuando pregunte si deseas cambiar la shell por defecto, responde:

```text
Y
```

Si no te pregunta o quieres hacerlo manualmente:

```bash
chsh -s $(which zsh)
```

Luego cierra y abre nuevamente la terminal.

---

## 3. Instalar autocompletado predictivo

Este plugin te muestra sugerencias basadas en comandos anteriores, parecido a Exegol:

```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```

---

## 4. Instalar resaltado de sintaxis

Esto colorea comandos válidos, errores, rutas, parámetros, etc.

```bash
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

---

## 5. Activar plugins

Edita el archivo:

```bash
nano ~/.zshrc
```

Busca esta línea:

```bash
plugins=(git)
```

Y cámbiala por:

```bash
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

Guarda con:

```text
CTRL + O
ENTER
CTRL + X
```

Recarga configuración:

```bash
source ~/.zshrc
```

---

## 6. Usar un tema bonito tipo Exegol

Puedes usar un tema simple y funcional.

Edita:

```bash
nano ~/.zshrc
```

Busca:

```bash
ZSH_THEME="robbyrussell"
```

Puedes cambiarlo por:

```bash
ZSH_THEME="agnoster"
```

O por uno más liviano:

```bash
ZSH_THEME="bureau"
```

Luego aplica los cambios:

```bash
source ~/.zshrc
```

---

## 7. Instalar Powerlevel10k, opción más profesional

Si quieres una terminal más moderna, con iconos, estado de Git, usuario, ruta, hora, etc.:

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes/powerlevel10k
```

Edita:

```bash
nano ~/.zshrc
```

Cambia el tema a:

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

Recarga:

```bash
source ~/.zshrc
```

Aparecerá un asistente de configuración. Responde las preguntas según cómo quieras que se vea la terminal.

---

## 8. Herramientas recomendadas para hacking ético

Instala también herramientas útiles para una terminal más cómoda:

```bash
sudo apt install bat eza fzf ripgrep htop tree tmux -y
```

En Ubuntu, a veces `bat` se instala como `batcat`.

Puedes crear algunos alias útiles:

```bash
echo "alias cat='batcat'" >> ~/.zshrc
echo "alias ll='eza -la --icons'" >> ~/.zshrc
echo "alias la='eza -la --icons'" >> ~/.zshrc
echo "alias grep='grep --color=auto'" >> ~/.zshrc
source ~/.zshrc
```

---

## 9. Configuración recomendada final

Tu línea de plugins debería quedar así:

```bash
plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
```

Y si usas Powerlevel10k:

```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

---

## Resultado esperado

Con esta configuración, tu Ubuntu tendrá una terminal mucho más parecida a Exegol:

- Predictiva
- Visual
- Cómoda
- Con historial inteligente
- Con colores
- Más rápida para trabajar en laboratorios de hacking ético

---

## Nota adicional

Si usas Ubuntu en una máquina virtual, también es recomendable instalar una fuente compatible con iconos, como **MesloLGS NF**, especialmente si vas a usar **Powerlevel10k**. Esto evita que aparezcan símbolos raros o cuadros vacíos en la terminal.
