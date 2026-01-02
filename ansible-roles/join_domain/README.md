# Ansible Role: join_domain

Ruolo Ansible per joinare sistemi Linux ad Active Directory utilizzando `realmd`.

## Descrizione

Questo ruolo automatizza il processo di join di sistemi Linux (RHEL, CentOS, Rocky, Alma, Ubuntu, Debian) ad un dominio Active Directory. Utilizza `realmd` e `sssd` per una gestione moderna e affidabile dell'integrazione con AD.

## Caratteristiche

- **Sicurezza migliorata**: Gestione password tramite file temporaneo cifrato (no esposizione in process list)
- **Controllo accessi granulare**: Limita login solo a gruppi AD specifici (consigliato per produzione)
- **Configurazione sudo**: Assegna privilegi sudo a gruppi AD specifici con validazione automatica
- **Installazione automatica**: Tutti i pacchetti necessari installati automaticamente
- **Multi-distribuzione**: Supporto completo per RHEL/CentOS/Rocky/Alma/Ubuntu/Debian
- **Idempotenza garantita**: Controlla se il sistema è già joinato prima di procedere
- **Tags per esecuzione selettiva**: Esegui solo i task necessari (packages, join, configure, access, verify)
- **Parametri configurabili**: Home directory, shell, SSSD settings personalizzabili
- **Validazione completa**: Verifica variabili richieste e successo operazioni
- **Handlers ottimizzati**: Riavvio servizi solo quando necessario

## Requisiti

- Ansible >= 2.9
- Privilegi sudo/root sui target
- Connettività di rete verso i Domain Controller
- Credenziali di un utente AD con permessi di join al dominio

## Variabili del Ruolo

### Variabili Richieste

```yaml
domain_name: corp.example.com        # Nome FQDN del dominio AD
domain_user: svc_domain_join         # Utente AD con permessi di join
domain_password: "{{ vault_password }}"  # Password (da fornire in modo sicuro)
```

### Variabili Opzionali

```yaml
# Opzioni aggiuntive per realm join (es. specificare OU)
realm_join_options: "--computer-ou=OU=Linux,OU=Servers,DC=corp,DC=example,DC=com"

# Configurazioni SSSD
sssd_configure_sudo: false               # Abilita integrazione sudo con AD
sssd_configure_ssh: false                # Abilita integrazione SSH con AD

# Configurazione home directory
sssd_home_dir_template: /home/%u         # Template per home directory (%u = username)
sssd_default_shell: /bin/bash            # Shell di default per utenti AD
sssd_override_homedir: false             # Override home directory in SSSD config
sssd_override_shell: false               # Override shell in SSSD config

# Impostazioni dominio SSSD
sssd_use_fully_qualified_names: false    # Usa nomi FQDN (user@domain.com)
sssd_fallback_homedir: /home/%u@%d       # Home directory fallback
sssd_access_provider: "permit"           # Provider controllo accessi (permit/deny/simple)

# Configurazione PAM mkhomedir
pam_mkhomedir_umask: "0077"              # Umask per creazione home directory
pam_mkhomedir_skel: /etc/skel            # Directory skeleton per nuovi utenti

# Controllo accessi (IMPORTANTE per sicurezza)
ad_restrict_access: false                # Abilita restrizione accesso per gruppi specifici
ad_login_groups: []                      # Lista gruppi AD autorizzati al login
# Example: ad_login_groups: ["linux-users", "administrators"]

# Configurazione sudo per gruppi AD
ad_configure_sudo: false                 # Abilita configurazione sudo per gruppi AD
ad_sudo_groups: []                       # Lista gruppi AD con privilegi sudo
# Example: ad_sudo_groups: ["domain admins", "linux-admins"]
ad_sudoers_file: "ad_sudoers"           # Nome file in /etc/sudoers.d/
```

## Dipendenze

Nessuna dipendenza esterna.

## Utilizzo

### 1. Installazione del Ruolo

#### Metodo A: Copia locale

Copia la directory `join_domain` nel tuo progetto Ansible:

```bash
cp -r ansible-roles/join_domain /path/to/your/ansible/project/roles/
```

#### Metodo B: Via requirements.yml (se pubblicato su Galaxy)

```yaml
# requirements.yml
- src: alessio.leopardi.join_domain
  version: 1.0.0
```

```bash
ansible-galaxy install -r requirements.yml
```

### 2. Creazione del Playbook

Crea un playbook che utilizza il ruolo:

```yaml
# site.yml
---
- name: Join servers to Active Directory
  hosts: linux_servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
```

### 3. Protezione delle Credenziali

#### Opzione A: Ansible Vault (Raccomandato)

1. Crea un file vault:
```bash
ansible-vault create group_vars/all/vault.yml
```

2. Aggiungi la password:
```yaml
# group_vars/all/vault.yml
vault_domain_password: YourSecurePassword123
```

3. Esegui il playbook:
```bash
ansible-playbook site.yml --ask-vault-pass
```

#### Opzione B: Variabile d'ambiente

```bash
export AD_PASSWORD="YourSecurePassword123"
ansible-playbook site.yml --extra-vars "domain_password=$AD_PASSWORD"
```

#### Opzione C: Prompt interattivo

```yaml
# site.yml
---
- name: Join servers to Active Directory
  hosts: linux_servers
  become: yes
  vars_prompt:
    - name: domain_password
      prompt: "Enter AD password"
      private: yes

  roles:
    - role: join_domain
```

### 4. Utilizzo dei Tags

Il ruolo supporta tags per esecuzione selettiva dei task:

```bash
# Solo installazione pacchetti
ansible-playbook site.yml --tags packages

# Solo join al dominio (salta installazione)
ansible-playbook site.yml --tags join

# Solo configurazione (assume pacchetti installati e sistema già joinato)
ansible-playbook site.yml --tags configure

# Solo verifica (controlla stato join)
ansible-playbook site.yml --tags verify

# Salta la configurazione
ansible-playbook site.yml --skip-tags configure

# Esegui solo validazione e verifica
ansible-playbook site.yml --tags validate,verify

# Solo configurazione controllo accessi e sudo
ansible-playbook site.yml --tags access,sudo

# Solo configurazione sicurezza (access + sudo)
ansible-playbook site.yml --tags security
```

**Tags disponibili:**
- `always`: Task eseguiti sempre (validazione e verifica)
- `validate`: Validazione variabili richieste
- `packages`, `install`: Installazione pacchetti
- `join`, `domain`: Operazioni di join al dominio
- `configure`, `config`: Configurazione post-join
- `access`: Controllo accessi per gruppi AD
- `sudo`: Configurazione sudo per gruppi AD
- `security`: Tutti i task di sicurezza (access + sudo)
- `verify`: Verifica finale

### 5. Esempi Avanzati

#### Specificare OU personalizzata

```yaml
---
- name: Join to AD with custom OU
  hosts: webservers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
        realm_join_options: "--computer-ou=OU=WebServers,OU=Linux,DC=corp,DC=example,DC=com"
```

#### Configurazione avanzata SSSD

```yaml
---
- name: Join with custom SSSD settings
  hosts: production_servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
        # Home directory in /data invece di /home
        sssd_home_dir_template: /data/users/%u
        # Shell zsh invece di bash
        sssd_default_shell: /bin/zsh
        # Usa nomi completi (user@domain.com)
        sssd_use_fully_qualified_names: true
        # Umask più restrittivo
        pam_mkhomedir_umask: "0027"
```

#### Controllo accessi con gruppi AD (CONSIGLIATO per produzione)

```yaml
---
- name: Join with access restrictions
  hosts: servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"

        # LIMITA ACCESSO - Solo questi gruppi possono fare login
        ad_restrict_access: true
        ad_login_groups:
          - "linux-users"
          - "system-administrators"
          - "developers"

        # CONFIGURA SUDO - Solo questi gruppi hanno privilegi sudo
        ad_configure_sudo: true
        ad_sudo_groups:
          - "domain admins"
          - "linux-admins"
        ad_sudoers_file: "ad_sudoers"
```

**IMPORTANTE**: Senza `ad_restrict_access: true`, **TUTTI** gli utenti del dominio possono fare login! In produzione è **fortemente consigliato** abilitare le restrizioni.

#### Configurazione per diversi ambienti

```yaml
---
- name: Join development servers (accesso permissivo)
  hosts: dev_servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
        ad_restrict_access: true
        ad_login_groups:
          - "developers"
          - "qa-team"

---
- name: Join production servers (accesso ristretto)
  hosts: prod_servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
        # Solo admin in produzione
        ad_restrict_access: true
        ad_login_groups:
          - "production-admins"
        ad_configure_sudo: true
        ad_sudo_groups:
          - "production-admins"
```

#### Inventory con gruppi diversi

```ini
# inventory.ini
[rhel_servers]
server1.corp.example.com
server2.corp.example.com

[ubuntu_servers]
ubuntu1.corp.example.com
ubuntu2.corp.example.com

[linux_servers:children]
rhel_servers
ubuntu_servers
```

```yaml
# site.yml
---
- name: Join all Linux servers to AD
  hosts: linux_servers
  become: yes

  roles:
    - join_domain
```

## Struttura del Ruolo

```
join_domain/
├── README.md                 # Questo file
├── defaults/
│   └── main.yml             # Variabili di default
├── handlers/
│   └── main.yml             # Handlers per riavvio servizi
├── meta/
│   └── main.yml             # Metadata e dipendenze
└── tasks/
    ├── main.yml             # Task principale
    ├── install_packages.yml # Installazione pacchetti
    ├── join_domain.yml      # Logica di join
    ├── configure.yml        # Configurazione post-join
    └── access_control.yml   # Controllo accessi e sudo
```

## Task Eseguiti

1. **Validazione** `[always, validate]`: Verifica che tutte le variabili richieste siano definite
2. **Installazione Pacchetti** `[packages, install]`:
   - Pacchetti base: realmd, sssd, adcli, samba-common-tools
   - RHEL/CentOS: oddjob, oddjob-mkhomedir, sssd-tools, krb5-workstation
   - Debian/Ubuntu: libnss-sss, libpam-sss, packagekit
3. **Check Status** `[join, domain]`: Verifica se il sistema è già joinato
4. **Domain Discovery** `[join, domain]`: Scopre il dominio AD tramite DNS
5. **Domain Join** `[join, domain]`:
   - Crea file temporaneo cifrato per la password (mode 0600)
   - Esegue il join al dominio con realm join
   - Rimuove il file temporaneo
6. **Configurazione** `[configure, config]`:
   - RHEL: Abilita e avvia oddjobd per creazione home directory
   - Debian: Configura PAM mkhomedir con parametri personalizzabili
   - Assicura che SSSD sia avviato e abilitato
7. **Controllo Accessi** `[access, security]` (opzionale):
   - Nega accesso a tutti gli utenti (realm deny --all)
   - Permette login solo ai gruppi AD specificati (realm permit -g)
   - Verifica configurazione accessi
8. **Configurazione Sudo** `[sudo, security]` (opzionale):
   - Crea file sudoers per gruppi AD
   - Configura privilegi sudo con validazione automatica (visudo -c)
   - Visualizza gruppi configurati
9. **Verifica** `[always, verify]`: Conferma che il join sia andato a buon fine

## Sicurezza

Il ruolo implementa diverse best practice di sicurezza:

### Gestione Password
- **Password Protection**: La password non viene mai esposta nella process list o nei log
- **File Temporaneo**: Uso di file temporaneo con permessi 0600 per passare la password
- **No Log**: Tutti i task sensibili usano `no_log: true`
- **Cleanup**: Il file temporaneo viene sempre rimosso, anche in caso di errore
- **Idempotenza**: Il join non viene rieseguito se il sistema è già nel dominio

### Controllo Accessi (IMPORTANTE)
- **Restrizione Login**: Con `ad_restrict_access: true`, solo i gruppi AD specificati possono accedere
- **Default Insicuro**: Senza restrizioni, TUTTI gli utenti del dominio possono fare login
- **Consiglio**: Abilitare sempre `ad_restrict_access: true` in produzione
- **Gruppi Multipli**: Supporto per più gruppi AD con diversi livelli di accesso

### Configurazione Sudo
- **Validazione Automatica**: Ogni modifica ai sudoers è validata con `visudo -c`
- **File Separati**: Usa `/etc/sudoers.d/` per configurazioni modulari
- **Permessi Corretti**: File sudoers con permessi 0440 (owner root)
- **Granularità**: Assegna sudo solo ai gruppi che ne hanno effettivamente bisogno

### Best Practice Consigliate
```yaml
# Configurazione MINIMA per produzione
ad_restrict_access: true
ad_login_groups: ["tuo-gruppo-specifico"]

# Configurazione COMPLETA per produzione
ad_restrict_access: true
ad_login_groups: ["sysadmins", "developers"]
ad_configure_sudo: true
ad_sudo_groups: ["sysadmins"]
```

## Handlers

- `restart sssd`: Riavvia il servizio SSSD
- `restart oddjobd`: Riavvia oddjobd (solo su RHEL/CentOS)

## Piattaforme Supportate

- **RHEL/CentOS/Rocky/Alma**: 7, 8, 9
- **Ubuntu**: 20.04 (Focal), 22.04 (Jammy), 24.04 (Noble)
- **Debian**: 10 (Buster), 11 (Bullseye), 12 (Bookworm)

## Troubleshooting

### Il dominio non viene scoperto

Verifica la risoluzione DNS:
```bash
nslookup corp.example.com
realm discover corp.example.com
```

### Join fallisce

Controlla i log:
```bash
journalctl -u realmd -n 50
tail -f /var/log/sssd/*.log
```

### Utenti AD non possono fare login

Verifica SSSD:
```bash
systemctl status sssd
id username@corp.example.com
```

### Home directory non viene creata

**RHEL/CentOS:**
```bash
systemctl status oddjobd
authselect current
```

**Ubuntu/Debian:**
```bash
cat /etc/pam.d/common-session | grep mkhomedir
```

## Licenza

MIT

## Autore

Alessio Leopardi

## Contribuire

Pull request e issue sono benvenuti!
