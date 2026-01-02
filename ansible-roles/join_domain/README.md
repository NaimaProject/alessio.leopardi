# Ansible Role: join_domain

Ruolo Ansible per joinare sistemi Linux ad Active Directory utilizzando `realmd`. Questo ruolo supporta distribuzioni RHEL/CentOS e Debian/Ubuntu e fornisce funzionalità complete di integrazione con il dominio.

## Caratteristiche

- ✅ Automatic domain discovery e join usando realmd
- ✅ Supporto multi-distribuzione (RHEL/CentOS/Rocky/Alma, Debian/Ubuntu)
- ✅ Operazioni idempotenti (sicuro eseguirlo più volte)
- ✅ Gestione sicura delle password via stdin
- ✅ Creazione automatica home directory per utenti del dominio
- ✅ Configurazione SSSD opzionale con template personalizzati
- ✅ Configurazione sudo opzionale per domain admins
- ✅ Configurazione accesso SSH opzionale
- ✅ Supporto per posizionamento in OU personalizzate
- ✅ Verifica completa e gestione errori

## Requisiti

- Ansible >= 2.9
- Privilegi sudo/root sui target
- Connettività di rete verso i Domain Controller
- DNS configurato correttamente per risolvere i nomi di dominio
- Firewall deve permettere Kerberos (TCP/UDP 88), LDAP (TCP/UDP 389) e porte correlate

## Distribuzioni Supportate

- RHEL/CentOS/Rocky/AlmaLinux 7, 8, 9
- Debian 10, 11, 12
- Ubuntu 18.04, 20.04, 22.04, 24.04

## Variabili del Ruolo

### Variabili Richieste

| Variabile | Descrizione | Esempio |
|-----------|-------------|---------|
| `domain_name` | Nome del dominio Active Directory | `corp.example.com` |
| `domain_user` | Utente del dominio con privilegi di join | `svc_domain_join` |
| `domain_password` | Password per l'utente del dominio | *Usare vault o --extra-vars* |

### Variabili Opzionali

| Variabile | Default | Descrizione |
|-----------|---------|-------------|
| `realm_join_options` | `""` | Opzioni aggiuntive per il comando realm join |
| `sssd_configure_sudo` | `false` | Abilita sudo per il gruppo domain admins |
| `sssd_configure_ssh` | `false` | Abilita autenticazione SSH con password |
| `sssd_configure_custom` | `false` | Usa template personalizzato per SSSD |
| `domain_admin_group` | `"Domain Admins"` | Gruppo del dominio per accesso sudo |
| `additional_sssd_domains` | `[]` | Domini trusted aggiuntivi da configurare |

### Opzioni Avanzate realm_join_options

```yaml
realm_join_options: "--computer-ou=OU=Linux,OU=Servers,DC=corp,DC=example,DC=com"
realm_join_options: "--automatic-id-mapping=no --user-principal=host/{{ ansible_fqdn }}@{{ domain_name | upper }}"
```

Opzioni comuni:
- `--computer-ou=OU_PATH` - Posiziona il computer in una OU specifica
- `--os-name=NAME` - Override del nome OS
- `--os-version=VERSION` - Override della versione OS
- `--automatic-id-mapping=yes/no` - Controlla il mapping automatico degli ID
- `--user-principal=UPN` - Imposta un user principal personalizzato

## Dipendenze

Nessuna.

## Esempi di Utilizzo

### Utilizzo Base

```yaml
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

### Utilizzo Avanzato con OU Personalizzata e Sudo

```yaml
---
- name: Join servers to AD with custom configuration
  hosts: linux_servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
        realm_join_options: "--computer-ou=OU=Linux,OU=Servers,DC=corp,DC=example,DC=com"
        sssd_configure_sudo: true
        sssd_configure_ssh: true
        domain_admin_group: "Linux Admins"
```

### Utilizzo con Configurazione SSSD Personalizzata

```yaml
---
- name: Join with custom SSSD config
  hosts: linux_servers
  become: yes

  roles:
    - role: join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
        sssd_configure_custom: true
        additional_sssd_domains:
          - trust1.example.com
          - trust2.example.com
```

## Best Practices di Sicurezza

### Non Hardcodare Mai le Password

**❌ Sbagliato:**
```yaml
vars:
  domain_password: "MyP@ssw0rd123"  # MAI FARE QUESTO
```

**✅ Corretto - Usare Ansible Vault:**

1. Crea un file vault criptato:
```bash
ansible-vault create vars/domain_credentials.yml
```

2. Aggiungi le credenziali:
```yaml
---
vault_domain_password: "MySecureP@ssw0rd"
```

3. Usa nel playbook:
```yaml
- hosts: servers
  vars_files:
    - vars/domain_credentials.yml
  roles:
    - role: join_domain
      vars:
        domain_password: "{{ vault_domain_password }}"
```

4. Esegui il playbook:
```bash
ansible-playbook site.yml --ask-vault-pass
```

**✅ Corretto - Usare Extra Vars:**
```bash
ansible-playbook site.yml --extra-vars "domain_password=$AD_PASSWORD"
```

**✅ Corretto - Usare Variabili d'Ambiente:**
```bash
export DOMAIN_PASSWORD="MySecureP@ssw0rd"
ansible-playbook site.yml --extra-vars "domain_password=$DOMAIN_PASSWORD"
```

## Esecuzione del Playbook

### Esecuzione Base
```bash
ansible-playbook -i inventory site.yml --ask-vault-pass
```

### Con Extra Variables
```bash
ansible-playbook -i inventory site.yml \
  --extra-vars "domain_name=corp.example.com domain_user=admin domain_password=$PASSWORD"
```

### Check Mode (Dry Run)
```bash
ansible-playbook -i inventory site.yml --check --diff
```

### Limitare a Host Specifici
```bash
ansible-playbook -i inventory site.yml --limit "webservers"
```

## Verifica

Dopo aver eseguito il ruolo, verifica lo stato del join al dominio:

```bash
# Verifica stato realm
realm list

# Verifica che SSSD sia in esecuzione
systemctl status sssd

# Test autenticazione utente del dominio
id domain_user@corp.example.com

# Test login utente del dominio (se SSH configurato)
ssh domain_user@corp.example.com@hostname
```

## Troubleshooting

### Domain Discovery Fallisce

**Problema:** `realm discover` non trova il dominio

**Soluzioni:**
- Verifica la configurazione DNS: `nslookup corp.example.com`
- Controlla i record SRV: `dig _ldap._tcp.corp.example.com SRV`
- Assicurati che il firewall permetta DNS (UDP/TCP 53)

### Join Fallisce con Errore di Autenticazione

**Problema:** Credenziali non valide

**Soluzioni:**
- Verifica che l'utente del dominio abbia privilegi di join
- Controlla che la password sia corretta (nessun problema di escape di caratteri speciali)
- Assicurati che l'account utente del dominio non sia bloccato o scaduto
- Prova il join manualmente: `realm join -U admin corp.example.com`

### SSSD Non Si Avvia

**Problema:** Il servizio SSSD non si avvia

**Soluzioni:**
- Controlla i log di SSSD: `journalctl -u sssd -n 50`
- Verifica che i permessi di `/etc/sssd/sssd.conf` siano 0600
- Testa la configurazione: `sssctl config-check`
- Controlla il contesto SELinux: `restorecon -Rv /etc/sssd`

### Gli Utenti del Dominio Non Possono Fare Login

**Problema:** L'autenticazione del dominio fallisce

**Soluzioni:**
- Verifica che SSSD sia in esecuzione: `systemctl status sssd`
- Controlla la configurazione PAM per il supporto del dominio
- Testa NSS: `getent passwd domain_user@corp.example.com`
- Rivedi i log di SSSD: `/var/log/sssd/sssd_*.log`

### Home Directory Non Vengono Create

**Problema:** Gli utenti possono fare login ma la home directory non viene creata

**Soluzioni:**
- RHEL/CentOS: Assicurati che `oddjobd` sia in esecuzione: `systemctl status oddjobd`
- Debian/Ubuntu: Controlla la configurazione PAM: `grep pam_mkhomedir /etc/pam.d/common-session`
- Crea manualmente per test: `mkhomedir_helper testuser`

## Struttura del Ruolo

```
join_domain/
├── README.md
├── defaults/
│   └── main.yml              # Variabili di default
├── handlers/
│   └── main.yml              # Handler per riavvio servizi
├── meta/
│   └── main.yml              # Metadata del ruolo
├── tasks/
│   ├── main.yml              # Orchestrazione task principale
│   ├── install_packages.yml  # Installazione pacchetti
│   ├── join_domain.yml       # Logica di join al dominio
│   └── configure.yml         # Configurazione post-join
├── templates/
│   ├── sssd.conf.j2          # Template configurazione SSSD
│   └── domain_admins_sudo.j2 # Template configurazione sudo
└── example-playbook.yml      # Esempio di utilizzo
```

## Task Eseguiti

1. **Validazione**: Verifica che tutte le variabili richieste siano definite
2. **Installazione Pacchetti**: Installa realmd, sssd, adcli, samba-common-tools e pacchetti specifici per distribuzione
3. **Verifica Status**: Controlla se il sistema è già joinato al dominio
4. **Domain Discovery**: Scopre il dominio AD tramite DNS
5. **Domain Join**: Esegue il join al dominio (se non già fatto)
6. **Configurazione**: Abilita la creazione automatica delle home directory e configurazioni opzionali
7. **Verifica**: Conferma che il join sia andato a buon fine

## Handlers

- `restart sssd`: Riavvia il servizio SSSD
- `restart oddjobd`: Riavvia oddjobd (solo su RHEL/CentOS)
- `restart sshd`: Riavvia il servizio SSH (quando configurato)

## Licenza

MIT

## Autore

Alessio Leopardi

## Contribuire

Contributi benvenuti! Assicurati che:
- Tutte le variabili siano documentate
- Vengano forniti esempi per le nuove funzionalità
- I cambiamenti mantengano l'idempotenza
- Vengano seguite le best practices di sicurezza

## Changelog

### Version 2.0
- ✅ Risolto problema di sicurezza con l'esposizione della password
- ✅ Migliorata idempotenza per la configurazione PAM
- ✅ Implementato supporto per realm_join_options
- ✅ Aggiunto supporto per configurazione SSSD personalizzata
- ✅ Aggiunte opzioni di configurazione sudo e SSH
- ✅ Risolti problemi con le condizioni negli handler
- ✅ Aggiunta documentazione completa
- ✅ Miglioramenti per supporto multi-distribuzione

### Version 1.0
- Release iniziale
- Funzionalità base di domain join
- Supporto RHEL/Debian
