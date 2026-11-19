# Informe web — Laboratorio Gestor SNMP

Sitio estático: un solo `index.html` y la carpeta `img/`. No necesita compilación ni dependencias.

```
informe/
├── index.html      # informe completo (HTML + CSS + JS embebidos)
├── img/            # 33 capturas del laboratorio
├── netlify.toml    # configuración para Netlify
└── vercel.json     # configuración para Vercel
```

## Ver localmente
Abrir `index.html` con doble clic en el navegador.

## Publicar

### Netlify (lo más rápido, sin cuenta de Git)
1. Entrar a https://app.netlify.com/drop
2. Arrastrar la carpeta **`informe`** completa.
3. Netlify entrega una URL pública del tipo `https://<nombre>.netlify.app`.

### Vercel
- **Desde GitHub:** subir el repo, importarlo en https://vercel.com/new y en *Root Directory* elegir `informe`. Framework Preset: **Other**. Sin build command.
- **Con la CLI:**
  ```powershell
  cd C:\Universidad\REDES2\Lab-SMNP\informe
  npx vercel --prod
  ```

### Render
1. https://dashboard.render.com → **New → Static Site** → conectar el repositorio de GitHub.
2. **Root Directory:** `informe` · **Build Command:** (vacío) · **Publish Directory:** `.`

## Antes de publicar
- Reemplazar `[Integrante 2]` en `index.html` por el nombre del compañero.
- La captura del firewall (`img/05-firewall.png`) ya tiene ocultos los nombres de redes personales.
