# ProfileTracker
Este workflow de GitHub Actions permite realizar un seguimiento de las visitas a un perfil de GitHub y enviar una notificación solo cuando se detecten nuevas visitas.

## 📌 Características
- Ejecuta la verificación automáticamente dos veces al día (a las 10:00 AM y 10:00 PM UTC).
- Permite ejecución manual mediante `workflow_dispatch`.
- Almacena el número de visitas previas en un archivo `views.txt` para evitar notificaciones repetidas.
- Envía una notificación a `ntfy.sh` cuando se detectan nuevas visitas.
- Realiza un commit en el repositorio con el número de visitas actualizado.
- Envía mensajes personalizados según el número de visitas generadas para mejorar la experiencia del usuario.

## 🚀 Instalación y Configuración

### 1️⃣ Habilitar GitHub Actions en el Repositorio
Asegúrate de que GitHub Actions esté habilitado en tu repositorio. Puedes verificarlo en la pestaña **Actions** de GitHub.

### 2️⃣ Crear el Archivo del Workflow
Puedes crear el archivo de dos maneras:

#### ✅ Opción 1: Automática desde GitHub
1. Ve a la pestaña **Actions** en tu repositorio de GitHub.
2. Busca y selecciona **Simple workflow**.
3. Haz clic en **Configure**.
4. Reemplaza el contenido generado con el código que se muestra a continuación.
5. Guarda el archivo con el nombre `.github/workflows/profile_views.yml`.

#### ✅ Opción 2: Manual
Crea un archivo en `.github/workflows/profile_views.yml` con el siguiente contenido:

```yaml
name: Notify Profile View Only on New Visits (Persistent)

on:
  schedule:
    - cron: '0 2,14 * * *'
  workflow_dispatch:  # Permite ejecución manual

jobs:
  send_notification:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Load Previous Views
        id: load_views
        run: |
          if [ -f views.txt ]; then
            PREV_VIEWS=$(cat views.txt)
          else
            PREV_VIEWS=0
          fi
          echo "PREV_VIEWS=$PREV_VIEWS" >> $GITHUB_ENV

      - name: Check Current Profile Views
        id: check_views
        run: |
          CURRENT_VIEWS=$(curl -s "https://komarev.com/ghpvc/?username='tu-usuario'" | grep -oE '[0-9]+' | tail -1)
          if [[ -z "$CURRENT_VIEWS" ]]; then
            echo "Error: No se pudo obtener el número de visitas."
            exit 1
          fi
          echo "CURRENT_VIEWS=$CURRENT_VIEWS" >> $GITHUB_ENV

      - name: Debugging Values
        run: |
          echo "Prev Views: $PREV_VIEWS"
          echo "Current Views: $CURRENT_VIEWS"

      - name: Compare and Notify
        run: |
          VISITS_DIFF=$((CURRENT_VIEWS - PREV_VIEWS))

          if [[ "$CURRENT_VIEWS" =~ ^[0-9]+$ && "$PREV_VIEWS" =~ ^[0-9]+$ ]]; then
            if [ "$CURRENT_VIEWS" -gt "$PREV_VIEWS" ]; then
              echo "Nueva visita detectada. Enviando notificación..."

              # Mensajes personalizados según el crecimiento de visitas
              if [ "$VISITS_DIFF" -eq 1 ]; then
                MESSAGE="👀 ¡Tienes 1 nueva visita, 📊 Total: $CURRENT_VIEWS"
              elif [ "$VISITS_DIFF" -gt 1 ] && [ "$VISITS_DIFF" -le 5 ]; then
                MESSAGE="🚀 ¡Tu perfil ha recibido $VISITS_DIFF nuevas visitas, 📊 Total: $CURRENT_VIEWS"
              else
                MESSAGE="🔥 ¡BOOM! Tu perfil explotó con $VISITS_DIFF visitas nuevas, 📊 Total: $CURRENT_VIEWS"
              fi

              curl -v -H "Title: 🚀 Nuevo visitante en el perfil!" \
                   -H "Tags: profile,visitor" \
                   -d "$MESSAGE" \
                   ntfy.sh/'tu-topic'
              echo "$CURRENT_VIEWS" > views.txt
            else
              echo "No hay nuevas visitas. No se enviará notificación."
            fi
          else
            echo "Error: Valores no numéricos detectados en CURRENT_VIEWS o PREV_VIEWS."
            exit 1
          fi

      - name: Commit Updated Views
        run: |
          if [[ $(git status --porcelain) ]]; then
            git config --local user.name "github-actions"
            git config --local user.email "actions@github.com"
            git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/${{ github.repository }}.git
            git add views.txt
            git commit -m "Update profile views: $CURRENT_VIEWS"
            git push
          else
            echo "No changes to commit."
          fi
```

### 3️⃣ Reemplazar Variables
- Cambia `'tu-usuario'` en la URL de `komarev.com` por tu nombre de usuario de GitHub.
- Reemplaza `'tu-topic'` con el nombre del topic en `ntfy.sh` donde deseas recibir las notificaciones.

### 4️⃣ Configurar un Secreto en GitHub  
Para permitir el commit automático, GitHub Actions necesita autenticación:

1. GitHub proporciona automáticamente un token de autenticación llamado `GITHUB_TOKEN` para cada flujo de trabajo de Actions.
2. Este token se utiliza automáticamente en las acciones de GitHub, por lo que no es necesario crearlo ni configurarlo manualmente.

### 5️⃣ Agregar la URL de Komarev en tu `README.md`
Para que las visitas a tu perfil de GitHub se registren correctamente, debes incrustar la URL de Komarev en tu `README.md`. Agrega el siguiente código en tu archivo `README.md`:

```html
<p align="left"> 
  <img src="https://komarev.com/ghpvc/?username='tu-usuario'&label=Profile%20views&color=0e75b6&style=flat" alt="'tu-usuario'" /> 
</p>
```
Reemplaza `'tu-usuario'` con tu nombre de usuario de GitHub. Este código mostrará el número de visitas a tu perfil y actualizará el contador cada vez que alguien visite tu perfil.

## 🔥 Uso Manual
Si deseas ejecutar la acción de forma manual:
1. Ve a la pestaña **Actions** de tu repositorio en GitHub.
2. Selecciona el workflow **Notify Profile View Only on New Visits**.
3. Haz clic en **Run workflow**.

## 🎯 Notificaciones con ntfy.sh
Este workflow utiliza [ntfy.sh](https://ntfy.sh) para enviar notificaciones push cuando se detectan nuevas visitas.
Puedes recibirlas en:
- Navegador: [https://ntfy.sh/tu-topic](https://ntfy.sh/tu-topic)
- Android: Descarga la app de ntfy.sh en Google Play.
- Apple: Fescarga la app de ntfy.sh en App Store.

## 📊 Notas Adicionales
- Este workflow verifica las visitas cada 12 horas.
- Se almacena el número de visitas en `views.txt` para evitar notificaciones repetidas.
- Si el número de visitas no cambia, no se enviará ninguna notificación.

## 📜 Licencia
Este proyecto es de código abierto bajo la licencia **MIT**. Puedes usarlo, modificarlo y compartirlo libremente.

## 🤝 Contribuciones
Si quieres mejorar el código o agregar nuevas funciones, ¡las contribuciones son bienvenidas! Puedes hacer un **fork**, modificarlo y abrir un **pull request**.
