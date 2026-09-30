# Practica 1 - target i servei

## Creacio i configuracio de taget

<img width="845" height="630" alt="Captura de pantalla de 2026-09-29 22-03-51" src="https://github.com/user-attachments/assets/4f5e0bad-8549-443e-a85c-37402e180dfe" />

<img width="728" height="55" alt="image" src="https://github.com/user-attachments/assets/818f4b24-5742-4941-8c5f-4563e3098620" />

## Creacio del servei
<img width="1923" height="1080" alt="Captura de pantalla de 2026-09-29 21-36-40" src="https://github.com/user-attachments/assets/7b6396b9-3487-41a9-9afa-707164c1adbd" />

<img width="817" height="89" alt="image" src="https://github.com/user-attachments/assets/4e3488d9-fb09-4909-8666-90b989764e72" />

<img width="850" height="137" alt="Captura de pantalla de 2026-09-29 21-37-08" src="https://github.com/user-attachments/assets/280e1b7e-228d-42c1-a420-0a416885ceae" />

<img width="1550" height="317" alt="Captura de pantalla de 2026-09-29 21-47-19" src="https://github.com/user-attachments/assets/2a5f2379-5533-49f1-bd29-db2ad95f5051" />
### Creacio de script per a servei

<pre>
#!/bin/bash
PORT=8080
PIPE="/tmp/web_terminal_pipe"
WORK_DIR_FILE="/tmp/web_terminal_pwd"

# Inicializar directorio de trabajo si no existe
[ -f "$WORK_DIR_FILE" ] || echo "$HOME" > "$WORK_DIR_FILE"

# Limpieza al salir
trap 'rm -f $PIPE; exit' INT TERM EXIT

[ -e "$PIPE" ] || mkfifo "$PIPE"

echo "Servidor Bash iniciado en 0.0.0.0:$PORT (Escuchando en todas las interfaces)"

while true; do
    # Cambio principal: escuchar en 0.0.0.0
    cat "$PIPE" | nc -l 0.0.0.0 $PORT | (
        
        # 1. Leer primera línea
        read -r method path proto
        
        # 2. Leer cabeceras
        content_length=0
        while read -r line && [ "$line" != $'\r' ] && [ -n "$line" ]; do
            if [[ "$line" =~ Content-Length:\ ([0-9]+) ]]; then
                content_length="${BASH_REMATCH[1]}"
            fi
        done

        # 3. Leer cuerpo
        body=""
        if [ "$content_length" -gt 0 ]; then
            read -n "$content_length" body
        fi

        # Ruta API (peticiones AJAX)
        if [ "$path" == "/api" ]; then
            cmd=""
            if [[ "$body" =~ command=([^&]*) ]]; then
                raw_cmd="${BASH_REMATCH[1]}"
                cmd=$(echo "$raw_cmd" | sed 's/+/ /g' | python3 -c "import sys, urllib.parse; print(urllib.parse.unquote(sys.stdin.read().strip()))" 2>/dev/null || echo "$raw_cmd")
            fi

            CURRENT_DIR=$(cat "$WORK_DIR_FILE" 2>/dev/null || echo "$HOME")

            output=""
            if [ -n "$cmd" ]; then
                EXEC_OUT=$(cd "$CURRENT_DIR" && eval "$cmd" 2>&1; pwd > "$WORK_DIR_FILE")
                output="$EXEC_OUT"
            fi

            NEW_DIR=$(cat "$WORK_DIR_FILE")

            echo -e "HTTP/1.1 200 OK\r"
            echo -e "Content-Type: text/plain; charset=utf-8\r"
            echo -e "Connection: close\r"
            echo -e "\r"
            echo -e "[$NEW_DIR]$ $cmd\n\n$output"

        else
            # Interfaz HTML
            echo -e "HTTP/1.1 200 OK\r"
            echo -e "Content-Type: text/html; charset=utf-8\r"
            echo -e "Connection: close\r"
            echo -e "\r"
            cat <<'EOF'
<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <title>Terminal Bash</title>
    <style>
        body { font-family: monospace; background: #121212; color: #00ff00; padding: 20px; }
        h2 { color: #fff; }
        form { margin-bottom: 20px; }
        input[type="text"] { width: 70%; padding: 10px; background: #222; color: #00ff00; border: 1px solid #444; font-size: 16px; }
        input[type="submit"] { padding: 10px 20px; background: #00ff00; color: #000; font-weight: bold; cursor: pointer; }
        pre { background: #000; padding: 15px; border: 1px solid #333; white-space: pre-wrap; word-wrap: break-word; min-height: 100px; }
	.contenedor {display: flex; justify-content: center; /* Centrado horizontal */ align-content: center;   /* Opcional: centrado vertical */}   
 </style>
</head>
<body>
    <h2>Terminal Bash</h2>
    <form id="cmdForm">
        <input type="text" id="commandInput" name="command" placeholder="Ejemplo: pwd" autofocus required>
        <input type="submit" value="Ejecutar">
    </form>
    <h3>Sortida:</h3>
    <pre id="output">Esperan comanda...</pre>

    <script>
        document.getElementById('cmdForm').addEventListener('submit', async function(e) {
            e.preventDefault();
            
            const input = document.getElementById('commandInput');
            const outputArea = document.getElementById('output');
            const cmd = input.value;
            
            outputArea.textContent = 'Executan...';

            try {
                const response = await fetch('/api', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
                    body: 'command=' + encodeURIComponent(cmd)
                });
                
                const data = await response.text();
                outputArea.textContent = data;
                input.value = '';
            } catch (err) {
                outputArea.textContent = 'Error: ' + err.message;
            }
        });
    </script>
<div class="contenedor"> 
   <img src="https://preview.redd.it/ryan-beckford-black-hackerman-matrix-hack-meme-hd-template-v0-jbcj0uqbihg41.jpg?width=640&amp;crop=smart&amp;auto=webp&amp;s=41837e2debb5dfb69ba45b614e1434665cc63cf8"  width="200" height="200">
</div>

</body>
</html>
EOF
        fi
    ) > "$PIPE"
done
</pre>
