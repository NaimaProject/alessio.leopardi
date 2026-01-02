# Ansible Role: join_domain

Ruolo Ansible per joinare sistemi Linux ad Active Directory utilizzando `realmd`.

## Descrizione

Questo ruolo automatizza il processo di join di sistemi Linux (RHEL, CentOS, Rocky, Alma, Ubuntu, Debian) ad un dominio Active Directory. Utilizza `realmd` e `sssd` per una gestione moderna e affidabile dell'integrazione con AD.

## Caratteristiche

- Installazione automatica di tutti i pacchetti necessari
- Supporto multi-distribuzione (RHEL/CentOS/Rocky/Alma/Ubuntu/Debian)
- Idempotenza: controlla se il sistema è già joinato
- Gestione sicura delle credenziali (no password in chiaro nei log)
- Configurazione automatica home directory per utenti AD
- Validazione completa e gestione errori
- Handlers per il riavvio dei servizi quando necessario

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

# Configurazioni SSSD (future implementazioni)
sssd_configure_sudo: false
sssd_configure_ssh: false
```

## Dipendenze

Nessuna dipendenza esterna.

## Utilizzo

### 1. Installazione del Ruolo

#### Metodo A: Copia locale

Copia la directory `roles/join_domain` nel tuo progetto Ansible:

```bash
cp -r roles/join_domain /path/to/your/ansible/project/roles/
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

### 4. Esempi Avanzati

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
├── tasks/
│   ├── main.yml             # Task principale
│   ├── install_packages.yml # Installazione pacchetti
│   ├── join_domain.yml      # Logica di join
│   └── configure.yml        # Configurazione post-join
└── vars/
    └── main.yml             # Variabili (vuoto per ora)
```

## Task Eseguiti

1. **Validazione**: Verifica che tutte le variabili richieste siano definite
2. **Installazione Pacchetti**: Installa realmd, sssd, adcli, samba-common-tools e pacchetti specifici per distribuzione
3. **Check Status**: Verifica se il sistema è già joinato
4. **Domain Discovery**: Scopre il dominio AD tramite DNS
5. **Domain Join**: Esegue il join al dominio (se non già fatto)
6. **Configurazione**: Abilita la creazione automatica delle home directory
7. **Verifica**: Conferma che il join sia andato a buon fine

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
