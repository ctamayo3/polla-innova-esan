# Polla Innova ESAN

App de quiniela/polla de fútbol entre 8 amigos para predecir resultados del Mundial 2026
(Cuartos de Final, Semifinales, Tercer Lugar y Final). Cesar es el admin y también participa.

## Stack técnico
- **Frontend:** un solo archivo `index.html` con HTML + CSS + JS vanilla (sin frameworks, sin build step)
- **Backend:** Supabase (Postgres + REST + Realtime), cliente `supabase-js` cargado desde CDN
- **Hosting:** Vercel, deploy automático al hacer `git push` a `main`
- **Repo:** https://github.com/ctamayo3/polla-innova-esan

## Estructura de datos (Supabase)
- `participants`: los 8 nombres fijos (Cesar, Mariana, Gabriela, Esther, Marilyn, Marita, Kyomi, Kiara), con `pin_hash` (login sin contraseña, PIN de 4 dígitos) y `is_admin`. `name` es la clave estable (liga predicciones, PIN, admin) y nunca se edita desde la UI. `display_name`, `avatar_emoji` y `profile_prompted` son personalización opcional: cada participante puede poner un nombre para mostrar y un emoji libre desde "Editar perfil"; se muestran en vez de `name`/👑🙋 en toda la UI, pero `name` sigue siendo el identificador real por debajo.
- `matches`: 8 partidos (`ref1` referencia sin puntuar + `qf2/qf3/qf4` cuartos + `sf1/sf2` semis + `third` tercer lugar + `final`), con `lock_at` (hora de cierre de predicciones, hora Perú), `result_home_score/away_score/penalty_winner`, `result_corners/result_cards/result_red_card` (booleanos, resultado real de las preguntas extra clásicas — no aplican a `ref1`), y `cancelled` (booleano). `home_flag`/`away_flag` son NOT NULL (texto, ej. emoji de bandera) aunque no se usan para renderizar — hay que incluirlos en cualquier INSERT nuevo o falla. `third` (Inglaterra vs Francia, agregado 2026-07-18) quedó `cancelled=true` por anuncio de último momento, luego se deshizo (`cancelled=false, scores_flag=true`) porque ya había gente con predicción puesta. `extended_predictions` (booleano, `true` solo en `final`) activa el modo de predicciones ampliado: en ese modo se ignoran `result_corners/result_cards` y se usan en su lugar `result_corners_range`/`result_cards_range` (entero 1-7 y 1-6, ver `CORNER_RANGES`/`CARD_RANGES` en el código) más `result_first_half_goal` (booleano) y `result_first_scorer` (`'home'/'away'/'none'`)
- `predictions`: una fila por participante+partido, con `home_score`, `away_score`, `penalty_winner`, `pred_corners/pred_cards/pred_red_card` (booleanos nullable, preguntas extra clásicas), y — solo relevantes cuando `matches.extended_predictions=true` — `pred_corners_range`/`pred_cards_range` (enteros nullable) y `pred_first_half_goal`/`pred_first_scorer` (nullable)

RLS está deshabilitado a propósito (app entre amigos, sin datos sensibles reales). El PIN protege el acceso solo a nivel de interfaz, no es seguridad real.

## Lógica de puntaje
- Marcador exacto (tiempo regular): 3 pts
- Solo acertar ganador/empate: 1 pt
- Bonus si acierta quién gana en penales (cuando hubo empate): +1 pt
- Extras clásicos (todos los partidos salvo `ref1`), opcionales, +1 pt cada uno si acierta, sin castigo si falla: ¿8+ córners?, ¿4+ tarjetas amarillas?, ¿hubo tarjeta roja?
- Extras ampliados (**solo en `final`**, `matches.extended_predictions=true`), todos opcionales:
  - Rango exacto de córners totales (7 opciones de a 3, última "19 o más") y rango de amarillas totales (6 opciones de a 2, última "11 o más"): +1 pt si acierta, **sin castigo** si falla (adrede — acertar 1-de-7/1-de-6 ya es difícil).
  - ¿Tarjeta roja?, ¿gol en el primer tiempo?, ¿quién anota primero? (equipo local/visitante/nadie): +1 pt si acierta, **-0.5 pt si respondió y falló** (el castigo NO aplica si dejó la pregunta sin responder). Este castigo de -0.5 en tarjeta roja es exclusivo de `final` — no aplica retroactivamente a los partidos ya jugados con el `pred_red_card`/`result_red_card` clásico.
  - Cada opción tiene un ícono ⓘ (función `infoIcon()`) que explica en un `alert()` qué gana/pierde exactamente esa predicción.
- Los puntos totales de un participante pueden quedar en negativo si le va mal en las categorías con castigo de la final — es el comportamiento esperado, no un bug
- Desempate final: gana quien tenga más "resultados exactos" acertados

## Reglas de negocio importantes
- Las predicciones de cada participante quedan **ocultas** para los demás hasta que se cierra el partido (`lock_at` pasado); después se revelan para todos
- Cada participante puede editar su propia predicción libremente hasta el cierre
- Solo el admin (Cesar) ve el panel Admin: pone `lock_at`, carga resultados reales (con confirmación), resetea PINs de otros participantes
- Todos los usuarios están en zona horaria de Perú (UTC-5) — cuidado con bugs de timezone al mostrar/guardar fechas (ya tuvimos uno, usar hora local del navegador, no `toISOString()`, para poblar inputs `datetime-local`)
- Las banderas de los equipos se dibujan con CSS (función `flagChip()`), no con emoji — los emoji de bandera no se ven bien en Windows
- Un partido con `cancelled=true` se saca de "Partidos" y del puntaje/tabla (se combina con `scores_flag=false`), pero se sigue mostrando en el Bracket con una tarjeta atenuada y un aviso "❌ Cancelado por anuncio de último momento"
- Cada participante puede personalizar su nombre para mostrar y su emoji desde el botón "✏️ Editar perfil" (topbar). La primera vez que entra, se le muestra automáticamente un modal invitándolo a hacerlo, pero puede posponerlo ("Ahora no") y hacerlo después; ese aviso solo se dispara una vez por participante (`profile_prompted`). El nombre/emoji personalizados son texto libre del usuario, así que se escapan (`escapeHtml()`) antes de insertarse en el HTML
- `avatar_emoji` también acepta una ruta/URL de imagen en vez de un emoji de texto (detectado por `isImageAvatar()`: empieza con `http`, `/` o `data:image`) — se renderiza como `<img class="avatar-img">` del mismo tamaño que el texto (1em). Las imágenes custom viven en `avatars/` (ej. `avatars/marita.png`, agregada a mano por Cesar, no autoservicio); conviene mantenerlas livianas (~200-300px, no el archivo original)

## Flujo de trabajo
- Cesar no tiene mucha experiencia con git/terminal, así que hay que dar pasos claros y verificar en qué carpeta está parado antes de correr comandos
- Después de cualquier cambio: `git add`, `git commit`, `git push` — Vercel redespliega solo
- Siempre confirmar antes de correr comandos que modifiquen o suban algo
