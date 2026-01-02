# Join Domain using realmd - Ansible Playbook

Questo playbook Ansible permette di joinare sistemi Linux a un dominio Active Directory utilizzando `realmd`.

## Caratteristiche

- **Sicurezza migliorata**: Gestione password tramite file temporaneo cifrato (no esposizione in process list)
- **Idempotenza garantita**: Verifica se il sistema è già joinato prima di procedere
- **Multi-distribuzione**: Supporto completo per RHEL/CentOS/Rocky/Alma e Debian/Ubuntu
- **Tags per esecuzione selettiva**: Esegui solo i task necessari (packages, join, configure, verify)
- **Gestione errori robusta**: Validazione delle variabili e verifica del successo dell'operazione
- **Configurazione PAM idempotente**: Evita duplicazioni su esecuzioni multiple
- **Auto-creazione home directory**: Configurazione automatica per utenti del dominio

## Prerequisiti

- Ansible 2.9 o superiore
- Accesso con privilegi sudo/root sui target
- Connettività di rete verso i Domain Controller
- Credenziali di un utente con permessi di join al dominio

## Variabili richieste

- `domain_name`: Nome del dominio AD (es. `example.com`)
- `domain_user`: Username con permessi di join al dominio
- `domain_password`: Password dell'utente (deve essere fornita in modo sicuro)

## Utilizzo

### Metodo 1: Password via prompt (raccomandato)

```bash
ansible-playbook join-domain-realmd.yml --extra-vars "domain_password=" --ask-vault-pass
```

### Metodo 2: Password criptata con ansible-vault

1. Crea un file di variabili criptato:
```bash
ansible-vault create secrets.yml
```

2. Aggiungi le credenziali:
```yaml
domain_password: your_secure_password
```

3. Esegui il playbook:
```bash
ansible-playbook join-domain-realmd.yml --extra-vars "@secrets.yml" --ask-vault-pass
```

### Metodo 3: Password via variabile d'ambiente

```bash
export DOMAIN_PASSWORD="your_password"
ansible-playbook join-domain-realmd.yml --extra-vars "domain_password=$DOMAIN_PASSWORD"
```

### Personalizzare dominio e utente

Modifica le variabili nel file yml oppure passa via command line:

```bash
ansible-playbook join-domain-realmd.yml \
  --extra-vars "domain_name=corp.example.com domain_user=admin domain_password=$DOMAIN_PASSWORD"
```

## Utilizzo dei Tags

Il playbook supporta tags per esecuzione selettiva:

```bash
# Solo installazione pacchetti
ansible-playbook join-domain-realmd.yml --tags packages --extra-vars "domain_password=$PASS"

# Solo join al dominio (salta installazione)
ansible-playbook join-domain-realmd.yml --tags join --extra-vars "domain_password=$PASS"

# Solo configurazione
ansible-playbook join-domain-realmd.yml --tags configure --extra-vars "domain_password=$PASS"

# Solo verifica
ansible-playbook join-domain-realmd.yml --tags verify

# Salta la configurazione
ansible-playbook join-domain-realmd.yml --skip-tags configure --extra-vars "domain_password=$PASS"
```

**Tags disponibili:**
- `always`: Task eseguiti sempre (validazione e verifica)
- `validate`: Validazione variabili richieste
- `packages`, `install`: Installazione pacchetti
- `join`, `domain`: Operazioni di join al dominio
- `configure`, `config`: Configurazione post-join
- `verify`: Verifica finale

## Cosa fa il playbook

1. **Validazione** `[always, validate]`: Verifica che tutte le variabili richieste siano presenti
2. **Installazione Pacchetti** `[packages, install]`:
   - Pacchetti base: realmd, sssd, adcli, samba-common-tools
   - RHEL/CentOS: oddjob, oddjob-mkhomedir, sssd-tools, krb5-workstation
   - Debian/Ubuntu: libnss-sss, libpam-sss, packagekit
3. **Check Status** `[join, domain]`: Verifica se il sistema è già joinato al dominio
4. **Domain Discovery** `[join, domain]`: Scopre il dominio tramite `realm discover`
5. **Domain Join** `[join, domain]`:
   - Crea file temporaneo cifrato con permessi 0600
   - Esegue il join al dominio tramite file temporaneo
   - Rimuove automaticamente il file temporaneo
6. **Configurazione** `[configure, config]`:
   - RHEL: Abilita oddjobd per creazione home directory
   - Debian: Configura PAM mkhomedir (idempotente con regexp)
   - Avvia e abilita il servizio SSSD
7. **Verifica** `[always, verify]`: Conferma che il join sia andato a buon fine

## Note di sicurezza

- **Password Protection**: La password viene scritta in un file temporaneo con permessi 0600 (nessuna esposizione in process list)
- **No Log**: Tutti i task sensibili usano `no_log: true` per prevenire logging
- **Cleanup**: Il file temporaneo viene sempre rimosso dopo l'uso
- **Vault**: Si raccomanda fortemente di usare ansible-vault per le credenziali
- **Best Practice**: Non committare mai password in chiaro nel repository

## Utilizzo come Ruolo Ansible

Questo playbook è stato anche convertito in un ruolo Ansible riutilizzabile per facilitare l'integrazione in progetti più complessi.

Il ruolo si trova in una directory separata: `../../ansible-roles/join_domain/`

### Struttura del Progetto

```
alessio.leopardi/
├── ansible-playbook/
│   └── join-domain-realmd/      # Playbook standalone
│       ├── join-domain-realmd.yml
│       └── README.md
└── ansible-roles/
    └── join_domain/               # Ruolo riutilizzabile
        ├── README.md
        ├── defaults/
        ├── handlers/
        ├── meta/
        └── tasks/
```

### Esempio di Utilizzo del Ruolo

Crea un playbook che utilizza il ruolo:

```yaml
# site.yml
---
- name: Join servers to Active Directory
  hosts: linux_servers
  become: yes

  roles:
    - role: ../../ansible-roles/join_domain
      vars:
        domain_name: corp.example.com
        domain_user: svc_domain_join
        domain_password: "{{ vault_domain_password }}"
```

Oppure copia il ruolo nel tuo progetto:

```bash
cp -r ../../ansible-roles/join_domain /path/to/your/project/roles/
```

Esegui con:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Vedi [../../ansible-roles/join_domain/README.md](../../ansible-roles/join_domain/README.md) per la documentazione completa del ruolo.

### Vantaggi del Ruolo

- **Modularità**: Riutilizzabile in diversi playbook e progetti
- **Manutenibilità**: Codice organizzato e più facile da mantenere
- **Scalabilità**: Integrabile con altri ruoli in playbook complessi
- **Parametri configurabili**: Home directory, shell, SSSD settings personalizzabili
- **Condivisione**: Può essere pubblicato su Ansible Galaxy
- **Task separati**: Logica divisa in file distinti per chiarezza

Il ruolo include anche variabili aggiuntive come:
- `sssd_home_dir_template`: Template per home directory
- `sssd_default_shell`: Shell di default
- `pam_mkhomedir_umask`: Umask per home directory
- E molte altre opzioni di configurazione SSSD

Vedi la [documentazione completa del ruolo](../../ansible-roles/join_domain/README.md) per tutti i dettagli.

## Troubleshooting

Se il playbook fallisce:

1. Verifica la connettività di rete con il DC:
```bash
realm discover your-domain.com
```

2. Controlla i log SSSD:
```bash
tail -f /var/log/sssd/*.log
```

3. Verifica lo stato del realm:
```bash
realm list
```

4. Per fare leave del dominio:
```bash
realm leave
```
