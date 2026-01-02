# Domain Join Playbook (Standalone)

Questo è un playbook standalone per joinare sistemi Linux ad Active Directory usando realmd.

## File Inclusi

- `join-domain-realid.yml` - Playbook principale
- `inventory.example.ini` - Esempio di file inventory
- `vault.example.yml` - Esempio di file vault per credenziali
- `README.md` - Questa documentazione

## Utilizzo Rapido

### 1. Preparazione

Copia e personalizza l'inventory di esempio:
```bash
cp inventory.example.ini inventory.ini
# Modifica inventory.ini con i tuoi server e domini
```

### 2. Configurazione Credenziali (Opzione A: Ansible Vault)

Crea un file vault per le credenziali:
```bash
ansible-vault create vault.yml
```

Aggiungi la password del dominio:
```yaml
---
vault_domain_password: "YourSecurePassword123"
```

### 3. Esecuzione

Con vault:
```bash
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --extra-vars "@vault.yml" \
  --ask-vault-pass
```

Con password da environment variable:
```bash
export AD_PASSWORD="YourPassword"
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --extra-vars "domain_password=$AD_PASSWORD"
```

Con prompt interattivo:
```bash
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --extra-vars "domain_password=" \
  --ask-pass
```

## Opzioni Disponibili

### Variabili Richieste

| Variabile | Descrizione | Esempio |
|-----------|-------------|---------|
| `domain_name` | Nome FQDN del dominio | `corp.example.com` |
| `domain_user` | Utente con privilegi di join | `svc_domain_join` |
| `domain_password` | Password utente dominio | Passare via vault/extra-vars |

### Variabili Opzionali

| Variabile | Default | Descrizione |
|-----------|---------|-------------|
| `realm_join_options` | `""` | Opzioni aggiuntive per realm join |
| `sssd_configure_sudo` | `false` | Abilita sudo per domain admins |
| `sssd_configure_ssh` | `false` | Abilita autenticazione SSH |
| `domain_admin_group` | `"Domain Admins"` | Gruppo per sudo access |

## Esempi Avanzati

### Join con OU Personalizzata

```bash
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --extra-vars "domain_password=$AD_PASSWORD" \
  --extra-vars "realm_join_options='--computer-ou=OU=Linux,OU=Servers,DC=corp,DC=example,DC=com'"
```

### Abilitare Sudo per Domain Admins

```bash
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --extra-vars "@vault.yml" \
  --extra-vars "sssd_configure_sudo=true domain_admin_group='Linux Admins'" \
  --ask-vault-pass
```

### Esecuzione su Host Specifici

```bash
# Solo server RHEL
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --limit rhel_servers \
  --extra-vars "domain_password=$AD_PASSWORD"

# Solo webservers
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --limit webservers \
  --extra-vars "@vault.yml" \
  --ask-vault-pass
```

### Dry Run (Check Mode)

```bash
ansible-playbook -i inventory.ini join-domain-realid.yml \
  --check --diff \
  --extra-vars "domain_password=$AD_PASSWORD"
```

## Verifica

Dopo l'esecuzione, verifica il join:

```bash
# Connettiti al server
ssh user@server.corp.example.com

# Verifica realm
sudo realm list

# Verifica SSSD
sudo systemctl status sssd

# Test autenticazione utente dominio
id domain_user@corp.example.com

# Test login SSH (se configurato)
ssh domain_user@corp.example.com@server.corp.example.com
```

## Troubleshooting

### Discovery del Dominio Fallisce

Verifica DNS:
```bash
nslookup corp.example.com
dig _ldap._tcp.corp.example.com SRV
```

### Join Fallisce

Controlla i log:
```bash
sudo journalctl -u realmd -n 50
sudo tail -f /var/log/sssd/*.log
```

### Verifica Connettività

```bash
# Test Kerberos
kinit username@CORP.EXAMPLE.COM

# Test LDAP
ldapsearch -H ldap://dc01.corp.example.com -b "dc=corp,dc=example,dc=com"
```

## Note di Sicurezza

- ⚠️ **MAI** hardcodare password nel playbook o inventory
- ✅ Usare sempre ansible-vault per le credenziali
- ✅ Usare `no_log: true` per task sensibili
- ✅ Limitare i permessi sui file vault (chmod 600)
- ✅ Usare utenti di servizio dedicati per il join al dominio

## Alternativa: Usare il Role

Per un uso più modulare e riusabile, considera di usare il role `join_domain` invece del playbook standalone:

```yaml
---
- name: Join servers using role
  hosts: linux_servers
  become: yes

  roles:
    - role: ../../ansible-roles/join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
```

Vedi `../../ansible-roles/join_domain/README.md` per la documentazione completa del role.
