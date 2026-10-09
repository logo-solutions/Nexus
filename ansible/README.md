# Déploiement Ansible de Nexus

Même squelette que `maisonnettev2/ansible` : rôle générique `compose_deploy`
(copie de `maisonnettev2/ansible/roles/compose_deploy`, à garder synchronisée).

Nexus tourne sur le Mac mini, depuis ce dépôt cloné (`Projects/Nexus`, projet
compose `nexus`). Le rôle est donc utilisé **en place** :

- pas d'archive ni de synchronisation (`compose_deploy_archive` vide) ;
- pas de `.env` (`compose_deploy_env_file` vide) ;
- pas de `pull` : l'image `latest` ne change que sur décision explicite ;
- aucune suppression d'image (`compose_deploy_prune_images: false`), le Mac
  mini héberge d'autres piles.

Tant que `docker-compose.yml` ne change pas, un passage ne recrée pas le
conteneur (vérifié le 2026-10-09 : même identifiant de conteneur avant et
après, `changed=0`).

```bash
cd ansible
ansible-playbook deploy.yml --check --diff   # contrôle sans rien toucher
ansible-playbook deploy.yml                  # applique docker-compose.yml
```

Contrôles : conteneur `nexus` en marche, `http://127.0.0.1:8081/` → 200.
