> ℹ️ Le playbook `wordpress_playbook.yml` évoqué ici est fourni, complet et testé, dans [`lab/wordpress.yml`](lab/wordpress.yml) : voir le [TP 5.5](TP5.5_Vault_WordPress.md) (mots de passe chiffrés avec Ansible Vault au lieu de `vars.yml` en clair).

# Exécuter le playbook
ansible-playbook -i hosts.ini wordpress_playbook.yml

# Avec des variables personnalisées
ansible-playbook -i hosts.ini wordpress_playbook.yml -e @vars.yml

# Avec demande de mot de passe sudo
ansible-playbook -i hosts.ini wordpress_playbook.yml --ask-become-pass



----------------------
nginx:
  image: nginx:alpine
  container_name: wordpress_nginx
  restart: always
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf:ro
    - wordpress_data:/var/www/html
  depends_on:
    - wordpress
  networks:
    - wordpress_network


-----------------------------
