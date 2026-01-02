# Join Domain using realmd - Ansible Playbook

Questo playbook Ansible permette di joinare sistemi Linux a un dominio Active Directory utilizzando `realmd`.

## Caratteristiche

- **Sicurezza migliorata**: La password non viene esposta nei log grazie a `no_log: true`
- **Idempotenza**: Verifica se il sistema è già joinato prima di procedere
- **Multi-distribuzione**: Supporta RHEL/CentOS/Rocky/Alma e Debian/Ubuntu
- **Gestione errori**: Validazione delle variabili e verifica del successo dell'operazione
- **Auto-creazione home directory**: Configurazione automatica per creare le home directory degli utenti del dominio

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
ansible-playbook join-domain-realid.yml --extra-vars "domain_password=" --ask-vault-pass
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
ansible-playbook join-domain-realid.yml --extra-vars "@secrets.yml" --ask-vault-pass
```

### Metodo 3: Password via variabile d'ambiente

```bash
export DOMAIN_PASSWORD="your_password"
ansible-playbook join-domain-realid.yml --extra-vars "domain_password=$DOMAIN_PASSWORD"
```

### Personalizzare dominio e utente

Modifica le variabili nel file yml oppure passa via command line:

```bash
ansible-playbook join-domain-realid.yml \
  --extra-vars "domain_name=corp.example.com domain_user=admin domain_password=$DOMAIN_PASSWORD"
```

## Cosa fa il playbook

1. Valida che tutte le variabili richieste siano presenti
2. Installa i pacchetti necessari (realmd, sssd, adcli, ecc.)
3. Installa pacchetti specifici per la distribuzione in uso
4. Verifica se il sistema è già joinato al dominio
5. Scopre il dominio tramite `realm discover`
6. Esegue il join al dominio (solo se non già joinato)
7. Configura la creazione automatica delle home directory
8. Riavvia il servizio SSSD
9. Verifica che il join sia andato a buon fine

## Note di sicurezza

- La password viene passata via stdin per evitare esposizione nella command line
- `no_log: true` previene il logging della password
- Si raccomanda fortemente di usare ansible-vault per le credenziali
- Non committare mai password in chiaro nel repository

## Utilizzo come Ruolo Ansible

Questo playbook è stato anche convertito in un ruolo Ansible riutilizzabile per facilitare l'integrazione in progetti più complessi.

Il ruolo si trova in una directory separata: `../../ansible-roles/join_domain/`

### Struttura del Progetto

```
alessio.leopardi/
├── ansible-playbook/
│   └── join-domain-realmid/      # Playbook standalone
│       ├── join-domain-realid.yml
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
- **Condivisione**: Può essere pubblicato su Ansible Galaxy
- **Task separati**: Logica divisa in file distinti per chiarezza

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
