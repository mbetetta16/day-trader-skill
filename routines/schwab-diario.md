Análisis diario de MIS acciones de Charles Schwab. Solo análisis: NUNCA ejecutes
trader-broker ni coloques, apruebes o simules una orden.

0. PREPARACIÓN
   - El repo se clona limpio en cada ejecución y la rama de trabajo puede no tener
     mis archivos. Antes de buscar la fuente, tráelos de main:
       git fetch origin main
       git checkout origin/main -- trader/schwab-positions.csv 2>/dev/null || true
       git checkout origin/main -- trader/schwab.txt 2>/dev/null || true
     Si siguen sin aparecer, búscalos en las demás ramas remotas:
       for b in $(git branch -r | grep -v HEAD); do git ls-tree -r --name-only $b | grep -i schwab | sed "s|^|$b: |"; done
     y tráelos con git checkout <rama> -- <ruta> (la más reciente por fecha de commit).
   - Fuente: trader/schwab-positions.csv (export de Schwab → Accounts → Positions → Export).
     Si no existe, usa trader/schwab.txt. Si no existe ninguno en ninguna rama, DETENTE
     y notifícame. Di qué ramas revisaste y recuérdame subir el CSV a main.
     No uses trader/universe.txt ni otra lista como sustituto.
   - Indica la fecha del último commit del CSV (git log -1 --format=%cs -- <ruta>).
     Si tiene más de 7 días, avísame en el resumen de que las posiciones pueden estar
     desactualizadas.
   - El CSV de Schwab trae 1–2 líneas de título antes del encabezado: busca la fila
     que empieza con "Symbol". Columnas útiles: Symbol, Description, Quantity, Price,
     Cost Basis, Gain $ / Gain %. Limpia "$", "," y "%"; "--" o "N/A" = vacío.
   - Descarta: "Cash & Cash Investments", "Account Total", fondos money market
     (SWVXX, SNVXX, SNOXX, SWTXX, SNSXX u otro cuya Description diga "Money Market"),
     opciones (símbolo con fecha/strike, p. ej. "AAPL 12/19/2026 200.00 C") y bonos/CDs.
     Conserva ETFs y acciones.
   - Costo por acción = Cost Basis / Quantity. Escribe un ticker por línea en
     trader/schwab.txt. Muestra una tabla: ticker, cantidad, costo/acción, precio,
     ganancia %.
   - Hora actual en Nueva York: python3 trader-alerts/alerts.py clock
   - User-Agent de la SEC: usa SEC_USER_AGENT o "sec_user_agent" en trader/config.json.
     No inventes uno ni pongas mi correo. Si no hay ninguno:
       · news.py se corre con --no-sec (solo titulares, sin 8-K)
       · facts.py se omite
       · el próximo reporte se toma de las noticias, o se escribe "desconocido"
     Anótalo en el resumen como paso omitido. Esto solo no amerita notificación.

1. ESCANEO (sin filtros que oculten mis posiciones)
   python3 trader-scanner/scan.py daily --watchlist trader/schwab.txt \
       --min-price 0 --min-volume 0 --out trader/schwab-daily.json
   Solo si el mercado está abierto y son entre 9:50 y 15:30 ET:
   python3 trader-scanner/scan.py orb --watchlist trader/schwab.txt \
       --min-price 0 --max-confirm 0 --out trader/schwab-orb.json
   Si no, escribe "ORB: fuera de ventana (<motivo>)" y sigue.

2. CONTEXTO (todos con los tickers de schwab.txt)
   python3 trader-news/news.py --tickers $(cat trader/schwab.txt) --hours 24 --all [--no-sec]
   python3 trader-analyst/facts.py $(cat trader/schwab.txt) --quarters 4   # solo con User-Agent
   python3 trader-pelosi/ptr.py new --since $(date -d '-90 days' +%F) --read \
       --tickers $(cat trader/schwab.txt)
   Próximo reporte: no hay un tool que lo dé. Estímalo como la fecha del último 10-Q/10-K
   + ~90 días, o tómalo de las noticias, y márcalo "(estimado)". Si cae en los
   próximos 7 días, márcalo ⚠.

3. POSIBILIDADES (solo para tickers con setup en el paso 1)
   Para cada hit ORB:
     python3 trader-entry/entry.py check TICKER --setup orb --write
     python3 trader-exit/exit.py plan --from trader/entry.json --stop or-mid --target 2R --write
     python3 trader-timemachine/timemachine.py run --plan trader/plan.json --until 15:55
   (Si entry.py no confirma la entrada, usa exit.py plan --symbol TICKER --entry <precio>
    --or-high <x> --or-low <y> y pasa a timemachine --entry-time con la hora de cierre
    de la barra de entrada, p. ej. "10:15 AM".)
   Para cada hit daily: si el mercado aún no abre, haz un plan con exit.py --entry <cierre de ayer>
   y corre timemachine, marcado "hipotético hasta la apertura". Si el mercado ya abrió,
   usa el precio actual y márcalo "hipotético (intradía)".
   Si algún paso propone un ticker que no está en trader/schwab.txt, descártalo y
   anota cuál y por qué. No ejecutes budget, risk ni broker.

4. RESUMEN (una línea por acción, en español)
   TICKER | costo → precio (±%) | setup (o "ninguno") + odds del timemachine
   (target/stop/abierto %) | noticia clave o "sin noticias" | próximo reporte |
   congresista que compró/vendió en 90 días (o "ninguno")
   Al final: errores, tickers descartados y pasos omitidos, cada uno con su motivo.
   mkdir -p trader/reports y guarda el resumen en trader/reports/schwab-<YYYY-MM-DD>.md.
   Haz commit (solo el reporte, schwab.txt y los JSON de schwab-*) y push a la rama de
   trabajo de la sesión.

5. NOTIFICACIÓN
   Envíame una notificación push solo si: hay un setup con target ≥ 40 % en el
   timemachine, alguna acción se mueve ±3 %, hay un 8-K o un reporte en ≤ 7 días,
   un congresista operó alguna de mis acciones, o la rutina falló (incluye la falta de
   fuente). Que te falte el User-Agent de la SEC no cuenta como falla.
   Pon lo más importante en la primera línea.
