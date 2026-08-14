# DevOps Lab — HAProxy (SSL) + 6× Nginx via Ansible & Docker

## 👥 Membres du groupe

| Nom | Matricule | Classe |
|---|---|---|
| **Tiavina Niaina** | 228 | L2 ARS |
| **Hariniaina** | 227 | L2 ARS |
| **Fanambinana** | 224 | L2 ARS |
| **Hery zo** | 242 | L2 ARS |

---

## 📋 Présentation du projet

Ce projet met en place une infrastructure web à haute disponibilité,
entièrement automatisée avec **Ansible** et **Docker**. Un unique point
d'entrée sécurisé en HTTPS (**HAProxy**) répartit le trafic vers **6
serveurs Nginx** identiques, chacun affichant une page thème "sport"
différente, ce qui permet de visualiser facilement la répartition de
charge.

L'objectif pédagogique est de démontrer la maîtrise des outils DevOps
suivants :
- Conteneurisation avec **Docker**
- Automatisation de déploiement avec **Ansible** (playbooks, rôles,
  templates Jinja2)
- Sécurisation des échanges avec **HTTPS/SSL**
- Répartition de charge et haute disponibilité avec **HAProxy**
  (load balancing + health checks)
- Tests et validation automatisés d'une infrastructure

## 🏗️ Architecture

```
Client (navigateur / curl)
        │  HTTPS :443
        ▼
   ┌─────────────┐
   │   HAProxy   │  ← reverse proxy + terminaison SSL + load balancer
   └─────────────┘
        │  HTTP :80 (interne, réseau Docker "webnet")
        ├──────┬──────┬──────┬──────┬──────┐
        ▼      ▼      ▼      ▼      ▼      ▼
      web1   web2   web3   web4   web5   web6
     Football Basket Tennis Rugby Volley  Boxe
     (Nginx)(Nginx)(Nginx)(Nginx)(Nginx)(Nginx)
```

**Principe clé** : le client ne communique **jamais** directement avec
les serveurs Nginx. Seul HAProxy expose des ports sur la machine hôte
(443, 80, 8404) ; les 6 conteneurs Nginx ne sont accessibles que depuis
le réseau Docker interne, ce qui réduit la surface d'attaque.

## 📁 Structure du projet

```
HAProxy/
├── ansible.cfg                  # réglages globaux d'Ansible
├── inventory.ini                # machines ciblées par Ansible
├── requirements.yml             # collection Ansible requise (community.docker)
├── playbook.yml                 # ORCHESTRATEUR PRINCIPAL (toutes les phases)
├── group_vars/
│   └── all.yml                  # variables du projet (ports, topologie, thème sport)
├── roles/
│   ├── certs/tasks/main.yml     # Phase 3 — génération du certificat SSL
│   ├── nginx/                   # Phase 4 & 5 — les 6 serveurs web
│   │   ├── tasks/main.yml
│   │   └── templates/
│   │       ├── nginx.conf.j2    # config Nginx (modèle commun aux 6 serveurs)
│   │       └── index.html.j2    # page web (modèle commun, thème sport)
│   ├── haproxy/                 # Phase 2, 3 & 6 — reverse proxy SSL + LB
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── templates/
│   │       └── haproxy.cfg.j2   # config HAProxy (SSL + load balancing + health checks)
│   └── tests/tasks/main.yml     # Phase 8 — tests et validation automatiques
└── .gitignore                   # exclut le dossier generated/ (certificats, configs générées)
```

> ⚠️ Le dossier `generated/` (clé privée SSL, certificats, fichiers de
> configuration rendus) **n'est jamais versionné** dans Git — il est
> produit automatiquement par Ansible à chaque exécution du playbook, et
> exclu via `.gitignore` pour ne jamais exposer de secret dans le dépôt.

## ⚙️ Prérequis

```bash
# Docker installé et démarré
docker --version

# Ansible + SDK Docker Python
pip install --break-system-packages ansible docker

# Collection Ansible Docker
ansible-galaxy collection install -r requirements.yml
```

## 🚀 Déploiement complet

```bash
git clone https://github.com/Miaro27/HAProxy.git
cd HAProxy
ansible-playbook playbook.yml
```

Ce que fait le playbook, dans l'ordre :

1. **Phase 1** — vérifie Docker/SDK, crée les répertoires de travail.
2. **Phase 3** — génère la clé privée, le certificat auto-signé et le
   bundle `haproxy.pem` requis par HAProxy.
3. **Phase 4 & 5** — génère la configuration et la page web de chacun
   des 6 serveurs à partir d'un **template Jinja2 unique**, puis
   démarre les 6 conteneurs Nginx sur le réseau Docker `webnet`.
4. **Phase 2, 3 & 6** — génère `haproxy.cfg` (SSL sur `:443`,
   redirection `:80 → :443`, `balance roundrobin`, health checks) et
   démarre le conteneur HAProxy.
5. **Phase 8** — exécute automatiquement une série de tests de
   validation (accès HTTPS, certificat SSL, répartition de charge,
   health checks, logs).

## 🎯 Déploiement par phase (tags)

```bash
ansible-playbook playbook.yml --tags prepare      # Phase 1
ansible-playbook playbook.yml --tags certs        # Phase 3
ansible-playbook playbook.yml --tags nginx        # Phase 4 & 5
ansible-playbook playbook.yml --tags haproxy      # Phase 2/3/6
ansible-playbook playbook.yml --tags tests        # Phase 8
```

## ✅ Tests et validation (Phase 8)

| # | Test | Ce qui est vérifié |
|---|---|---|
| 1 | Accès HTTPS via HAProxy | `curl -k https://127.0.0.1:443/` renvoie 200 |
| 2 | Certificat SSL | Le certificat auto-signé est bien présenté par HAProxy |
| 3 | Répartition de charge | 12 requêtes successives passent bien par les 6 serveurs en round-robin |
| 4 | Health checks HAProxy | La page de stats (`:8404`) est accessible |
| 5 | Résilience (failover) | Le service reste disponible après l'arrêt d'un serveur Nginx |
| 6 | Logs | Les logs HAProxy et Nginx sont exploitables |

Pour inclure le test de failover (coupure volontaire d'un serveur) :
```bash
ansible-playbook playbook.yml --tags tests -e run_failover_test=true
```

Vérifications manuelles utiles :
```bash
# Voir la répartition sur les 6 serveurs
for i in $(seq 1 12); do curl -sk https://127.0.0.1/ | grep -o 'web[1-6]'; done

# Page de statistiques HAProxy (état de chaque backend)
curl http://127.0.0.1:8404/

# Simuler une panne
docker stop web3
curl -sk -o /dev/null -w "%{http_code}\n" https://127.0.0.1/
docker start web3
```

## 🔒 Sécurité

- Le certificat SSL est **auto-signé** : `curl` doit être appelé avec
  `-k` (en production, on utiliserait un certificat signé par une
  autorité, ex. Let's Encrypt, sans changer le reste de l'architecture).
- Les 6 conteneurs Nginx ne publient **aucun port** sur l'hôte : seul
  HAProxy est exposé, ce qui centralise la sécurité en un seul point
  d'entrée.
- La clé privée SSL n'est **jamais versionnée** dans Git (voir
  `.gitignore`).

## 🧩 Répartition du travail (Git)

| Membre | Rôle technique | Fichiers |
|---|---|---|
| **Hariniaina** | Architecture & Automatisation Ansible | `playbook.yml`, `group_vars/`, `ansible.cfg`, `inventory.ini`, `roles/certs/` |
| **Tiavina Niaina** | Serveurs Web (Nginx) | `roles/nginx/` |
| **Fanambinana** | Reverse Proxy & Tests (HAProxy) | `roles/haproxy/`, `roles/tests/` |
**Hery Zo** |  Tests (HAProxy) | `, `roles/tests/` |mais sans compte github a planté a la dernière minute du copu fanambianana a envoyé ca partie mais il a contribué au projet

## 🧹 Nettoyage

```bash
docker rm -f haproxy-lb web1 web2 web3 web4 web5 web6
docker network rm webnet
rm -rf generated/
```
