# Ansible Active Directory Integration

Repository contenente playbook e ruoli Ansible per l'integrazione di sistemi Linux con Active Directory.

## Struttura del Repository

```
.
├── ansible-playbook/              # Playbook standalone pronti all'uso
│   └── join-domain-realmid/      # Playbook per domain join con realmd
│       ├── join-domain-realid.yml
│       ├── inventory.example.ini
│       ├── vault.example.yml
│       └── README.md
│
└── ansible-roles/                 # Ruoli Ansible riusabili
    └── join_domain/               # Ruolo per domain join
        ├── defaults/              # Variabili di default
        ├── handlers/              # Service handlers
        ├── meta/                  # Role metadata
        ├── tasks/                 # Task files
        ├── templates/             # Jinja2 templates
        ├── example-playbook.yml   # Esempio di utilizzo
        └── README.md              # Documentazione completa
```

## Quick Start

### Opzione 1: Usare il Playbook Standalone

Per un utilizzo rapido e diretto:

```bash
cd ansible-playbook/join-domain-realmid
cp inventory.example.ini inventory.ini
# Modifica inventory.ini con i tuoi server

# Esegui il playbook
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --extra-vars "domain_password=$AD_PASSWORD"
```

### Opzione 2: Usare il Role

Per un'integrazione più modulare nei tuoi playbook esistenti:

```yaml
---
- name: Join servers to Active Directory
  hosts: linux_servers
  become: yes

  roles:
    - role: ansible-roles/join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
```

## Caratteristiche Principali

### ✅ Sicurezza
- Gestione sicura delle password via stdin (no esposizione in /proc)
- Supporto per Ansible Vault
- Validazione delle credenziali
- `no_log` per operazioni sensibili

### ✅ Idempotenza
- Controllo dello stato prima delle operazioni
- Configurazione PAM idempotente
- Sicuro eseguirlo più volte

### ✅ Multi-distribuzione
- RHEL/CentOS/Rocky/AlmaLinux 7, 8, 9
- Debian 10, 11, 12
- Ubuntu 18.04, 20.04, 22.04, 24.04

### ✅ Funzionalità Avanzate
- Posizionamento in OU personalizzate
- Configurazione SSSD custom
- Sudo per domain admins
- Accesso SSH configurabile
- Home directory automatiche

## Documentazione Dettagliata

- **Playbook Standalone**: [ansible-playbook/join-domain-realmid/README.md](ansible-playbook/join-domain-realmid/README.md)
- **Role join_domain**: [ansible-roles/join_domain/README.md](ansible-roles/join_domain/README.md)

## Requisiti

- Ansible >= 2.9
- Accesso root/sudo ai server target
- Connettività di rete verso Domain Controller
- DNS configurato correttamente
- Credenziali di un account con privilegi di domain join

## Esempi d'Uso

### Join Base

```bash
ansible-playbook -i inventory site.yml \
  --extra-vars "domain_password=$AD_PASSWORD"
```

### Join con OU Personalizzata

```bash
ansible-playbook -i inventory site.yml \
  --extra-vars "domain_password=$AD_PASSWORD" \
  --extra-vars "realm_join_options='--computer-ou=OU=Linux,OU=Servers,DC=corp,DC=example,DC=com'"
```

### Join con Sudo per Domain Admins

```bash
ansible-playbook -i inventory site.yml \
  --extra-vars "@vault.yml" \
  --extra-vars "sssd_configure_sudo=true" \
  --ask-vault-pass
```

## Best Practices di Sicurezza

### ✅ Fare

- Usare Ansible Vault per le credenziali
- Passare password via environment variables o --extra-vars
- Limitare i permessi sui file di configurazione
- Usare utenti di servizio dedicati per il domain join
- Testare in ambiente di sviluppo prima

### ❌ Non Fare

- Hardcodare password nei playbook o inventory
- Committare file vault.yml non criptati
- Usare account personali per il domain join
- Eseguire in produzione senza test

## Verifica del Join

Dopo l'esecuzione, verifica lo stato:

```bash
# Connettiti al server
ssh user@server.example.com

# Verifica realm
sudo realm list

# Verifica SSSD
sudo systemctl status sssd

# Test autenticazione
id domain_user@corp.example.com

# Verifica home directory
ls -la /home/
```

## Troubleshooting

### Problema: Domain Discovery Fallisce

**Causa**: DNS non configurato correttamente

**Soluzione**:
```bash
# Verifica DNS
nslookup corp.example.com
dig _ldap._tcp.corp.example.com SRV

# Aggiungi nameserver se necessario
echo "nameserver 10.0.0.10" | sudo tee -a /etc/resolv.conf
```

### Problema: Join Fallisce con Errore di Autenticazione

**Causa**: Credenziali errate o account senza privilegi

**Soluzione**:
- Verifica password
- Controlla che l'utente abbia privilegi di join
- Verifica che l'account non sia bloccato/scaduto

### Problema: SSSD Non Si Avvia

**Causa**: Configurazione o permessi errati

**Soluzione**:
```bash
# Verifica configurazione
sudo sssctl config-check

# Verifica permessi
sudo chmod 600 /etc/sssd/sssd.conf

# Controlla log
sudo journalctl -u sssd -n 50
```

## Contribuire

Contributi benvenuti! Per favore:

1. Fai un fork del repository
2. Crea un branch per la tua feature (`git checkout -b feature/nuova-funzionalita`)
3. Committa le modifiche (`git commit -am 'Aggiunta nuova funzionalità'`)
4. Pusha il branch (`git push origin feature/nuova-funzionalita`)
5. Apri una Pull Request

## Changelog

### v2.0.0 (2026-01-02)

**Miglioramenti di Sicurezza:**
- ✅ Risolto problema di password exposure in `/proc`
- ✅ Migliore gestione stdin per credenziali
- ✅ Validazione migliorata delle variabili

**Nuove Funzionalità:**
- ✅ Supporto completo per `realm_join_options`
- ✅ Configurazione SSSD personalizzabile
- ✅ Supporto sudo per domain admins
- ✅ Configurazione SSH opzionale
- ✅ Template per configurazioni avanzate

**Miglioramenti:**
- ✅ Idempotenza migliorata per configurazione PAM
- ✅ Handler corretti senza condizioni
- ✅ Documentazione completa e dettagliata
- ✅ Esempi di inventory e vault
- ✅ README multi-livello

**Bug Fix:**
- ✅ Risolto problema di configurazione PAM duplicata
- ✅ Fixati handler con condizioni problematiche
- ✅ Migliorata gestione errori

### v1.0.0

- Release iniziale
- Funzionalità base di domain join
- Supporto RHEL/Debian

## Licenza

MIT License - Vedi file LICENSE per dettagli

## Autore

**Alessio Leopardi**

## Supporto

Per problemi o domande:
- Apri un issue su GitHub
- Consulta la documentazione dettagliata nei README specifici

## Riferimenti Utili

- [Documentazione realmd](https://www.freedesktop.org/software/realmd/)
- [Documentazione SSSD](https://sssd.io/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [Active Directory Integration Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/integrating_rhel_systems_directly_with_windows_active_directory/index)
