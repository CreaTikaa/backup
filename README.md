## Serv Web : 

```
sudo apt install apache2 -y
```

```
sudo apt install mariadb-server -y
sudo mysql_secure_installation
Set password et yes a quasi tout le reste (comme je veux)
```

```
sudo apt install php libapache2-mod-php php-mysql php-cli php-curl php-gd php-mbstring php-xml php-zip -y
```

```
sudo systemctl restart apache2
```

```
echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/info.php
```

#### sqldump :  

::: info
Dans : /usr/local/bin/mysqldump-all.sh 

:::

```bash

#!/bin/bash

BACKUP_DIR="/var/backups/mariadb"
mkdir -p "$BACKUP_DIR"

TIMESTAMP=$(date +"%F_%H-%M-%S")
DUMP_FILE="$BACKUP_DIR/all-databases_$TIMESTAMP.sql"

mysqldump --all-databases --single-transaction --quick --lock-tables=false > "$DUMP_FILE"

chmod 600 "$DUMP_FILE"
```

::: info
Dans : /root/.my.cnf

:::

```bash
[client]
user=root
password=TON_MDP_ROOT
chmod 600 /root/.my.cnf
```

#### borgbackup : 

```
sudo apt install borgbackup -y
```

```
borg init --encryption=repokey crea@192.168.56.2:/home/crea/backup/intranet
```

Puis : 

```bash
echo "ma_passphrase" > /root/.borg_pass_daily ou autre
chmod 600 /root/.borg_pass_daily ou autre
```

#### Borgmatic : 

```bash
sudo apt update
sudo apt install pipx
```

```bash
En root : 
sudo pipx ensurepath
sudo pipx install borgmatic
```

Puis toujours en root : 

```bash
which borgmatic, 
trouver le path et l'ajouter dans : 
Defaults    secure_path = /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin (visudo)
```

```bash
sudo borgmatic config generate
cd /etc/borgmatic (puis créer nouvelle conf)
```

Config yaml daily :  

```yaml
source_directories:
  - /var/www
  - /var/log/apache2
  - /etc/apache2

repositories:
  - path: ssh://crea@192.168.56.2/home/crea/backup/intranet_daily

hooks:
  before_backup:
    - /usr/local/bin/mysqldump-all.sh

compression: zstd,6

keep_daily: 14
keep_weekly: 0
keep_monthly: 0

checks:
  - name: repository
  - name: archives
check_last: 3
```

```yaml
source_directories:
  - /var/www
  - /var/log/apache2
  - /etc/apache2

repositories:
  - path: ssh://crea@192.168.56.2/home/crea/backup/intranet_weekly

hooks:
  before_backup:
    - /usr/local/bin/mysqldump-all.sh

compression: zstd,6

keep_monthly: 5

checks:
  - name: repository
  - name: archives
check_last: 3
```

```bash
sudo borgmatic config validate
```

#### Services : 

::: info
sudo nano /etc/systemd/system/borgmatic-daily.service

:::

```
[Unit]
Description=Borgmatic Daily Backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
Environment="BORG_PASSCOMMAND=cat /root/.borg_pass_daily"
ExecStartPost=find /var/backups/mariadb -mtime +5 -delete
ExecStart=/root/.local/bin/borgmatic --config /etc/borgmatic/intranet_daily.yaml
```

::: info
et sudo nano /etc/systemd/system/borgmatic-weekly.service

:::

```
[Unit]
Description=Borgmatic Weekly Backup
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
Environment="BORG_PASSCOMMAND=cat /root/.borg_pass_wekly"
ExecStartPost=find /var/backups/mariadb -mtime +5 -delete
ExecStart=/root/.local/bin/borgmatic --config /etc/borgmatic/intranet_weekly.yaml
```

#### Timer : 

::: info
sudo nano /etc/systemd/system/borgmatic-daily.timer

:::

```
[Unit]
Description=Run daily backup using Borgmatic

[Timer]
OnCalendar=19:30
Persistent=true

[Install]
WantedBy=timers.target
```

::: info
sudo nano /etc/systemd/system/borgmatic-weekly.timer

:::

```
[Unit]
Description=Run weekly backup using Borgmatic 

[Timer]
OnCalendar=Sun 02:00
Persistent=true

[Install]
WantedBy=timers.target
```

Puis, tout reload : 

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now borgmatic-daily.timer
sudo systemctl enable --now borgmatic-weekly.timer
```

### restore_intranet.sh

```bash
#!/bin/bash

# Configuration
REMOTE_USER="crea"
REMOTE_HOST="192.168.56.105"
REMOTE_BASE_PATH="/home/crea/backup/intranet"
LOCAL_RESTORE_DIR="$HOME/restore_test"

# Daily ou Weekly
echo "Quel type de backup voulez-vous restaurer ?"
select backup_type in "daily" "weekly" "Quitter"; do
  case $backup_type in
    daily )
      REPO_NAME="daily"
      break
      ;;
    weekly )
      REPO_NAME="weekly"
      break
      ;;
    Quitter )
      echo "Annulation."
      exit 0
      ;;
    * )
      echo "Choix invalide."
      ;;
  esac
done

# Path du repo
REPO="ssh://${REMOTE_USER}@${REMOTE_HOST}${REMOTE_BASE_PATH}/${REPO_NAME}"
export BORG_REPO="$REPO"

/!\ NE PAS METTRE MOT DE PASSE EN CLAIR
 export BORG_PASSPHRASE="mdp"


echo "Voici les archives disponibles dans le dépôt $REPO_NAME :"
mapfile -t archives < <(borg list --short "$REPO")
if [[ ${#archives[@]} -eq 0 ]]; then
  echo "Aucune archive trouvée."
  exit 1
fi

echo
for i in "${!archives[@]}"; do
  echo "[$i] ${archives[$i]}"
done

# Choisir l’archive à restaurer
echo
read -p "Entrez le numéro de l’archive que vous voulez restaurer : " index
selected_archive="${archives[$index]}"

if [[ -z "$selected_archive" ]]; then
  echo "Index invalide. Fin du script."
  exit 1
fi

echo
echo "Restauration de l'archive : $selected_archive"
echo "Destination : $LOCAL_RESTORE_DIR"

mkdir -p "$LOCAL_RESTORE_DIR"
cd "$LOCAL_RESTORE_DIR"

borg extract "$REPO::$selected_archive" --progress --verbose

if [[ $? -eq 0 ]]; then
  echo "✅ Restauration terminée dans : $LOCAL_RESTORE_DIR"
else
  echo "❌ Erreur lors de la restauration."
fi
```

# Pour Windows : 

::: info
Téléchargement Restic zip pour Windows

Unzip et ajout au path (pas la peine de .exe juste le path ou est le .exe)

Se co en ssh crea@192.168.56.2

sudo apt-get install restic

:::

Aller dans les bons paths et :  

```
restic -r /backup/secretaire/daily init
restic -r /backup/secretaire/weekly init
restic -r /backup/autocad/daily init
restic -r /backup/autocad/weekly init
restic -r /backup/gdrive/daily init
restic -r /backup/gdrive/weekly init
```

::: warn
Mettre tout les scripts dans un dossier /restic/backup/ pour que ce soit propre

:::

# restic_env.ps1

```
# restic_env.ps1
$env:RESTIC_PASSWORD = "mdp"
$env:RESTIC_REPOSITORY_SD = "sftp:crea@192.168.56.2:/home/crea/backup/secretaire/daily"
$env:RESTIC_REPOSITORY_SW = "sftp:crea@192.168.56.2:/home/crea/backup/secretaire/weekly"
$env:RESTIC_REPOSITORY_AD = "sftp:crea@192.168.56.2:/home/crea/backup/autocad/daily"
$env:RESTIC_REPOSITORY_SW = "sftp:crea@192.168.56.2:/home/crea/backup/autocad/weekly"
$env:RESTIC_REPOSITORY_GD = "sftp:crea@192.168.56.2:/home/crea/backup/gdrive/daily"
$env:RESTIC_REPOSITORY_GW = "sftp:crea@192.168.56.2:/home/crea/backup/gdrive/weekly"
$env:RESTIC_KEY_HINT = ""
$env:RESTIC_CACHE_DIR = "$env:LOCALAPPDATA\restic-cache"
```

::: info
Différents scripts de save, juste a changer la last line pour passer de daily a weekly ou l'inverse :  (et oui tout est chiffré en AES et la clé c'est le mdp)

:::

::: warn
include.txt pour chacuns : 

:::

```
# Secrétaire
C:\Users\Secrétaire\Documents*.pdf
C:\Users\Secrétaire\Documents*.docx
C:\Users\Secrétaire\Documents*.zip
```

```
# Ingénieurs Autocad
C:\Users\AppData\Local\Temp*.sv$C:\Users\Ingénieur1\Documents*.dwg
C:\Users\Ingénieur2\Documents*.dwg
C:\Users\Ingénieur3\Documents*.dwg
```

```
# Comptabilité
C:\Users\Compta\Documents*.pdf
C:\Users\Compta\Documents*.docx
C:\Users\Compta\Documents*.zip
C:\Users\Compta\Documents*.csv
C:\Users\Compta\Documents*.xml
C:\Users\Compta\Documents*.txt
```

```
# Secretaire Daily
. "C:\restic\restic_env.ps1"

$env:RESTIC_REPOSITORY = $env:RESTIC_REPOSITORY_SD

restic backup --files-from-verbatim "C:\restic\backup\secretaire_include.txt"`

restic forget --keep-daily 7 --prune
```

```
# Autocad Weekly
. "C:\restic\restic_env.ps1"

$env:RESTIC_REPOSITORY = $env:RESTIC_REPOSITORY_AW

restic backup --files-from-verbatim "C:\restic\backup\autocad_include.txt"

restic forget --keep-weekly 4 --prune
```

::: warn
 Possible de changer le cache pour les weekly pour faire moins de dédupliquement : $env:RESTIC_CACHE_DIR = "$env:LOCALAPPDATA\\restic-cache-weekly"

:::

```
# Comptabilité Weekly
. "C:\restic\restic_env.ps1"

$env:RESTIC_REPOSITORY = $env:RESTIC_REPOSITORY_CD (compta)

restic backup --files-from-verbatim "C:\restic\backup\comptabilite_include.txt"

restic forget --keep-weekly 4 --prune
```

## Planification via Task Scheduler

### 🗓️ Tâche 1 : Daily

- Nom : Restic Daily Backup Secretaire/Autocad/Compta/gdrive
- Déclencheur : tous les jours à 03:00
- Action :
  - Programme : powershell.exe
  - Arguments : -ExecutionPolicy Bypass -File "C:\\restic\\backup-daily.ps1"
- Options :
  - Coche : “Exécuter avec les autorisations maximales”
  - Déclenche même si utilisateur non connecté

::: success
Edit le nom du script et le time en fonction

:::

#### restore_backup.ps1 

```powershell
$profile = Read-Host "Quel profil veux-tu restaurer ? (secretaire/autocad/comptabilite)"
if ($profile -ne "secretaire") {
    Write-Host "La restauration automatique est disponible uniquement pour 'secretaire'." -ForegroundColor Red
    exit 1
}

# Daily ou Weekly
$repoType = Read-Host "Quelle sauvegarde veux-tu restaurer ? (daily/weekly)"
if ($repoType -ne "daily" -and $repoType -ne "weekly") {
    Write-Host "Choix invalide. Abandon." -ForegroundColor Red
    exit 1
}

# Variable d'env
$env:RESTIC_REPOSITORY = "sftp:crea@192.168.56.105:/home/crea/backup/secretaire/$repoType"
$env:RESTIC_PASSWORD = "mdp"

# Liste des snapshots
Write-Host ""
Write-Host "Liste des snapshots disponibles ($repoType):" -ForegroundColor Cyan
try {
    $snapshots = & restic snapshots --json | ConvertFrom-Json
} catch {
    Write-Host "Erreur lors de la récupération des snapshots." -ForegroundColor Red
    exit 1
}

if (-not $snapshots -or $snapshots.Count -eq 0) {
    Write-Host "Aucun snapshot trouvé." -ForegroundColor Red
    exit 1
}

$i = 1
foreach ($snap in $snapshots) {
    Write-Host "[$i] $($snap.short_id) - $($snap.time) - $($snap.hostname) - $($snap.paths -join ', ')"
    $i++
}

$choice = Read-Host "`nQuel snapshot veux-tu restaurer ? (1 à $($snapshots.Count))"
if ($choice -notmatch '^\d+$' -or [int]$choice -lt 1 -or [int]$choice -gt $snapshots.Count) {
    Write-Host "Choix invalide." -ForegroundColor Red
    exit 1
}

$selected = $snapshots[([int]$choice - 1)].short_id

$restoreTarget = Read-Host "Dans quel dossier veux-tu restaurer ? (ex: C:\Restaurations\restic)"
if (-not (Test-Path $restoreTarget)) {
    New-Item -ItemType Directory -Path $restoreTarget -Force | Out-Null
}

# Restauration
Write-Host ""
Write-Host "Restauration en cours depuis '$repoType', snapshot $selected..." -ForegroundColor Yellow
& restic restore $selected --target "$restoreTarget"

Write-Host ""
Write-Host "Restauration terminée dans : $restoreTarget" -ForegroundColor Green
```

Lancer le script de réupération : 
```
powershell -ExecutionPolicy Bypass -File restore_backup.ps1
```
