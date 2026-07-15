# Polla Innova ESAN

App de quiniela/polla de fútbol entre 8 amigos para predecir resultados del Mundial 2026
(Cuartos de Final, Semifinales y Final). Cesar es el admin y también participa.

## Stack técnico
- **Frontend:** un solo archivo `index.html` con HTML + CSS + JS vanilla (sin frameworks, sin build step)
- **Backend:** Supabase (Postgres + REST + Realtime), cliente `supabase-js` cargado desde CDN
- **Hosting:** Vercel, deploy automático al hacer `git push` a `main`
- **Repo:** https://github.com/ctamayo3/polla-innova-esan

## Estructura de datos (Supabase)
- `participants`: los 8 nombres fijos (Cesar, Mariana, Gabriela, Esther, Marilyn, Marita, Kyomi, Kiara), con `pin_hash` (login sin contraseña, PIN de 4 dígitos) y `is_admin`. `name` es la clave estable (liga predicciones, PIN, admin) y nunca se edita desde la UI. `display_name`, `avatar_emoji` y `profile_prompted` son personalización opcional: cada participante puede poner un nombre para mostrar y un emoji libre desde "Editar perfil"; se muestran en vez de `name`/👑🙋 en toda la UI, pero `name` sigue siendo el identificador real por debajo.
- `matches`: 7 partidos (`ref1` referencia sin puntuar + `qf2/qf3/qf4` cuartos + `sf1/sf2` semis + `final`), con `lock_at` (hora de cierre de predicciones, hora Perú), `result_home_score/away_score/penalty_winner`, y `result_corners/result_cards/result_red_card` (booleanos, resultado real de las preguntas extra — no aplican a `ref1`)
- `predictions`: una fila por participante+partido, con `home_score`, `away_score`, `penalty_winner`, y `pred_corners/pred_cards/pred_red_card` (booleanos nullable, respuestas opcionales a las preguntas extra)

RLS está deshabilitado a propósito (app entre amigos, sin datos sensibles reales). El PIN protege el acceso solo a nivel de interfaz, no es seguridad real.

## Lógica de puntaje
- Marcador exacto (tiempo regular): 3 pts
- Solo acertar ganador/empate: 1 pt
- Bonus si acierta quién gana en penales (cuando hubo empate): +1 pt
- Extras opcionales, +1 pt cada uno si acierta (independientes entre sí, hasta +3 por partido): ¿8+ córners?, ¿4+ tarjetas amarillas?, ¿hubo tarjeta roja? — todo en formato Sí/No fijo (sin línea configurable), solo disponibles en los partidos que faltan por jugar cuando se agregó la función (sf1, sf2, final); no se agregaron a los cuartos ya jugados
- Desempate final: gana quien tenga más "resultados exactos" acertados

## Reglas de negocio importantes
- Las predicciones de cada participante quedan **ocultas** para los demás hasta que se cierra el partido (`lock_at` pasado); después se revelan para todos
- Cada participante puede editar su propia predicción libremente hasta el cierre
- Solo el admin (Cesar) ve el panel Admin: pone `lock_at`, carga resultados reales (con confirmación), resetea PINs de otros participantes
- Todos los usuarios están en zona horaria de Perú (UTC-5) — cuidado con bugs de timezone al mostrar/guardar fechas (ya tuvimos uno, usar hora local del navegador, no `toISOString()`, para poblar inputs `datetime-local`)
- Las banderas de los equipos se dibujan con CSS (función `flagChip()`), no con emoji — los emoji de bandera no se ven bien en Windows
- Cada participante puede personalizar su nombre para mostrar y su emoji desde el botón "✏️ Editar perfil" (topbar). La primera vez que entra, se le muestra automáticamente un modal invitándolo a hacerlo, pero puede posponerlo ("Ahora no") y hacerlo después; ese aviso solo se dispara una vez por participante (`profile_prompted`). El nombre/emoji personalizados son texto libre del usuario, así que se escapan (`escapeHtml()`) antes de insertarse en el HTML
- `avatar_emoji` también acepta una ruta/URL de imagen en vez de un emoji de texto (detectado por `isImageAvatar()`: empieza con `http`, `/` o `data:image`) — se renderiza como `<img class="avatar-img">` del mismo tamaño que el texto (1em). Las imágenes custom viven en `avatars/` (ej. `avatars/marita.png`, agregada a mano por Cesar, no autoservicio); conviene mantenerlas livianas (~200-300px, no el archivo original)

## Flujo de trabajo
- Cesar no tiene mucha experiencia con git/terminal, así que hay que dar pasos claros y verificar en qué carpeta está parado antes de correr comandos
- Después de cualquier cambio: `git add`, `git commit`, `git push` — Vercel redespliega solo
- Siempre confirmar antes de correr comandos que modifiquen o suban algo
