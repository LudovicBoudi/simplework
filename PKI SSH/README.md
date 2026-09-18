# PKI SSH — Rôle Ansible `ssh_cert_manager`

Automatise la génération et la distribution de certificats SSH (hôtes et
utilisateurs) via une autorité de certification (CA) SSH.

## Architecture

```
[ssh_ca]  ca01            # nœud dédié qui héberge la clé de la CA root
[ssh_servers] server*     # hôtes recevant un certificat d'hôte
[ssh_workstations] ws*    # postes recevant les certificats utilisateurs
```

- **CA root** : clé générée sur le nœud `ca01` (`/etc/ssh/ca_ssh_ca`), jamais
  distribuée.
- **Certificats d'hôte** : la clé publique de chaque serveur est signée par la
  CA (`-h`), puis déployée (`HostCertificate` + `TrustedUserCAKeys`).
- **Certificats utilisateur** : les paires de clés sont générées par le rôle,
  signées par la CA, puis déployées dans `~<user>/.ssh/` sur le poste cible.
- **Confiance** : les postes ajoutent la CA comme `@cert-authority` dans
  `known_hosts`, les serveurs déclarent la CA dans `sshd_config`.

## Prérequis

- Ansible >= 2.9
- `ssh-keygen` (OpenSSH) sur le nœud de signature et le contrôleur
- Un inventaire déclarant les groupes `ssh_ca`, `ssh_servers`,
  `ssh_workstations`

## Utilisation

```bash
cd "PKI SSH"
ansible-playbook playbooks/pki.yml
```

Le playbook cible `hosts: all` ; le rôle détermine lui-même quoi faire sur
chaque hôte en fonction de son groupe.

## Configuration

Les valeurs par défaut sont dans
`roles/ssh_cert_manager/defaults/main.yml`. Les points de configuration
principaux sont exposés dans `inventory/hosts.yml` (`all.vars`) :

| Variable | Description |
| --- | --- |
| `ssh_ca_hostname` | Nom d'hôte du nœud CA (ex. `ca01`). **Si vide, le rôle génère une CA temporaire de test.** |
| `ssh_users` | Liste d'utilisateurs `{ name, host }` à certifier. |
| `ssh_principal_domain` | Domaine ajouté aux principaux des certificats d'hôte. |
| `ssh_host_cert_validity` / `ssh_user_cert_validity` | Durée de validité des certificats émis. |

### CA temporaire (tests avant mise en production)

Quand `ssh_ca_hostname` est vide (ou que le groupe `ssh_ca` est absent), le
rôle génère une **CA temporaire** sur le contrôleur (`localhost`) dans
`/tmp/ssh_ca_temp`, avec une validité courte (`+24h`). Cela permet de valider
le déroulé sans CA définitive :

```yaml
# inventory/hosts.yml
all:
  children:
    ssh_servers: { ... }
    ssh_workstations: { ... }
  vars:
    ssh_ca_hostname: ""   # => CA temporaire
    ssh_users:
      - name: alice
        host: workstation01
```

Pour la production, renseigner `ssh_ca_hostname` et déclarer le groupe
`ssh_ca` ; la CA persistante sera créée sur ce nœud.

## Déroulé du rôle

1. `tasks/ca.yml` — résout la config de la CA, génère la clé (persistante ou
   temporaire) et expose la clé publique en fact.
2. `tasks/host_certs.yml` — signe et déploie les certificats d'hôte sur les
   serveurs.
3. `tasks/user_certs.yml` — génère, signe et distribue les certificats
   utilisateurs.
4. `tasks/trust.yml` — déploie la clé publique de la CA, configure `sshd`
   (serveurs) et `known_hosts` (postes).

## Notes

- La clé privée de la CA ne quitte jamais le nœud de signature : la signature
  est effectuée à distance via `delegate_to`.
- Les artefacts de staging sont stockés dans `/var/tmp/ssh_pki_staging` (nœud
  de signature) et `/tmp/ssh_pki_local` (contrôleur).
- Le rôle recharge `sshd` après toute modification de configuration
  (`handlers/main.yml`).
