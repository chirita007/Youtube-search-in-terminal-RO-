# Youtube-search-in-terminal-RO-
Pentru a intelege mai usor ce se intampla , folositi preview  <code>.
Acest repo este gandit in asa fel incat sa elimine BrainRot-ul din cadrul design-ului proiectat de Youtube Shorts pe care eu unul il consider daunator pentru psihic.

Sistemul pe care acest repo a fost conceput este Debian 13 (trixie) x86_64 din cadrul AntiX Linux Core, pe kernel Linux 7.0.12-x64v3-xanmod1, shell bash 5.2.37.

Dependentele acestui repo sunt urmatoarele:
1. Dependențe Core (Critice)
Fără acestea, scriptul nu va porni sau nu va putea prelua datele:

bash: Interpretul de scripturi în care este scris codul. (Vine preinstalat pe orice distribuție Linux/WSL/MSYS2).

yt-dlp: Motorul principal care caută pe YouTube și extrage metadatele (titlu, durată, URL). Trebuie să fie versiunea nouă (2026.06) instalată manual.

fzf (Fuzzy Finder): Utilitarul care generează interfața interactivă de căutare și panoul de preview.

mpv: Player-ul video/audio minimalist care preia URL-ul final trimis de fzf și redă clipul.

2. Utilitare de sistem standard (Posibil deja instalate)
Acestea sunt folosite pentru manipularea textului, animație și sistemul de cache:

awk (sau gawk): Folosit masiv în script pentru plasarea coloanelor, calcularea scorului de relevanță, formatarea secundelor în MM:SS și generarea textului din preview.

coreutils: O suită care conține comenzi de bază deja prezente în sistem, dar esențiale pentru script:

md5sum: Folosit pentru a genera un nume unic criptat pentru fișierul de cache, pe baza textului căutat.

stat: Folosit pentru a verifica când a fost modificat ultima oară fișierul de cache (verificarea celor 30 de minute).

date: Folosit pentru a transforma datele calendaristice în secunde (timestamp) pentru funcția score.

xargs: Trimite link-ul selectat ca argument către mpv.

ncurses-bin (utilitarul tput): Folosit în funcția de animație pentru a ascunde cursorul (tput civis) și a-l reafișa la final (tput cnorm).

jq

Pentru o instalare rapida va las aici comanda: 
{1} sudo apt update && sudo apt install fzf mpv gawk jq coreutils ncurses-bin

Notă: yt-dlp nu îl pune din apt 2026.03, recomand instalare manuala prin curl, la moment este versiunea 2026.06.
{2} sudo curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
{3} sudo chmod a+rx /usr/local/bin/yt-dlp
{4} yt-dlp --version
{5} sudo yt-dlp -U [ NU FACETI UPDATE PRIN apt. FOLOSITI ACEASTA COMANDA ]

Acum ca aveti toate dependentele, trecem la treaba. Creem Youtube Search (ytsrc)
{6} 
mkdir -p ~/.local/bin
nano ~/.local/bin/ytsrc
#In interiorul fisierului ytsrc copiati:
{7}
#!/usr/bin/env bash

q="$*"
[ -z "$q" ] && echo "Usage: ytsrc <search>" && exit 1

CACHE_DIR="${HOME}/.cache/ytsrc"
mkdir -p "$CACHE_DIR"

CACHE_FILE="$CACHE_DIR/$(echo "$q" | md5sum | awk '{print $1}')"

# --- FUNCȚIE PENTRU ANIMAȚIA DE ÎNCĂRCARE ---
num_spinner() {
    local delay=0.1
    # Caracterele din care e compusă animația ([ | ], [ / ], [ - ], [ \ ])
    local spinstr='|/-\'
    # Ascundem cursorul ca să arate curat
    tput civis 
    echo -n " Căutare în desfășurare...  "
    while true; do
        local temp=${spinstr#?}
        printf "[ %c ]" "$spinstr"
        local spinstr=$temp${spinstr%"$temp"}
        sleep $delay
        printf "\b\b\b\b\b"
    done
}

# Funcție care va readuce cursorul pe ecran dacă scriptul e întrerupt cu Ctrl+C
cleanup() {
    tput cnorm
    exit 0
}
trap cleanup SIGINT SIGTERM
# --------------------------------------------

# Cache 30 min
if [ -f "$CACHE_FILE" ] && [ $(($(date +%s) - $(stat -c %Y "$CACHE_FILE"))) -lt 1800 ]; then
    cat "$CACHE_FILE"
else
    # 1. Pornim animația în fundal (&)
    num_spinner &
    # 2. Salvăm ID-ul procesului animației ca să o putem opri mai târziu
    SPINNER_PID=$!

    # 3. Rulăm yt-dlp (totul e silențios ca să nu strice animația)
    yt-dlp "ytsearch30:$q" \
      --extractor-args "youtube:player_client=android" \
      --quiet --no-warnings \
      --print "%(title)s|%(_old_fields)s%(uploader)s|%(upload_date)s|%(duration)s|%(view_count)s|%(webpage_url)s" \
      > "$CACHE_FILE" 2>/dev/null

    # 4. yt-dlp a terminat! Oprim animația din fundal
    kill "$SPINNER_PID" 2>/dev/null
    wait "$SPINNER_PID" 2>/dev/null
    
    # Curățăm linia unde era scris textul de încărcare și reafișăm cursorul
    printf "\r\033[K"
    tput cnorm
fi

cat "$CACHE_FILE" |
awk -F"|" '
function score(date, views,    y,m,d,cmd,ts,age) {
    if (length(date) != 8) return 0
    y=substr(date,1,4)
    m=substr(date,5,2)
    d=substr(date,7,2)

    cmd="date +%s -d \"" y "-" m "-" d "\""
    cmd | getline ts
    close(cmd)

    age = systime() - ts
    # Scor echilibrat: vechimea contează mult, vizualizările adaugă un bonus
    return (views / 100000) + (8640000 / (age + 1))
}
{
    sc = score($3, $5)
    # Rețonem toate datele originale ordonate după scor
    print sc "|" $1 "|" $2 "|" $3 "|" $4 "|" $5 "|" $6
}
' |
sort -t"|" -k1 -nr |
awk -F"|" '
function format_duration(sec,    h,m,s) {
    if (sec == "") return "00:00"
    h=int(sec/3600)
    m=int((sec%3600)/60)
    s=sec%60
    if (h > 0) 
        return sprintf("%d:%02d:%02d", h, m, s)
    else 
        return sprintf("%02d:%02d", m, s)
}
function timeago(d,    y,m,da,cmd,ts,now,days) {
    if (length(d) != 8) return "unknown date"
    y=substr(d,1,4)
    m=substr(d,5,2)
    da=substr(d,7,2)

    cmd="date +%s -d \"" y "-" m "-" da "\""
    cmd | getline ts
    close(cmd)

    now=systime()
    days=int((now-ts)/86400)

    if (days < 1) return "Today"
    else if (days < 7) return days "d ago"
    else if (days < 30) return int(days/7) "w ago"
    else if (days < 365) return int(days/30) "mo ago"
    else return int(days/365) "y ago"
}
function format_views(v) {
    if (v == "") return "0"
    if (v >= 1000000) return sprintf("%.1fM", v/1000000)
    if (v >= 1000) return sprintf("%.1fk", v/1000)
    return v
}
{
    # Intră: 1:sc|2:title|3:uploader|4:date|5:duration|6:views|7:url
    # Ieșire curată pentru fzf separat prin "|"
    print $2 "|" $3 "|" format_duration($5) "|" timeago($4) "|" format_views($6) "|" $7
}
' |
fzf \
  --delimiter="|" \
  --with-nth=1,2,3,4 \
  --prompt="YTSRC > " \
  --preview='echo {} | awk -F"|" "{
    print \"=== VIDEO INFO ===\"
    print \"Titlu:       \" \$1
    print \"Canal:       \" \$2
    print \"Durată:      \" \$3
    print \"Postat:      \" \$4
    print \"Vizualizări: \" \$5
  }"' \
| awk -F"|" '{print $6}' \
| xargs -r mpv --ytdl-format="best[height<=480]"

{8} chmod +x ~/.local/bin/ytsrc

{9}
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

Gata!

Comanda pentru a cauta pe youtube este:
ytsrc <cautare>
ex: ytsrc IRaphahell
ex: ytsrc how to degoogled
ex: ytsrc miami music 80s
