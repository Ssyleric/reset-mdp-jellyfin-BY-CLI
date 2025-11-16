# Reset du mot de passe d’un utilisateur Jellyfin via CLI (LXC)

## 🎯 Objectif

Réinitialiser le mot de passe d’un utilisateur Jellyfin (`USER$` dans cet exemple) **sans recréer le compte**, directement depuis le conteneur LXC, quand l’interface web affiche *Connection Failure* ou que le mot de passe est perdu.

Environnement :

- Jellyfin installé dans un LXC Ubuntu
- Base SQLite : `/var/lib/jellyfin/data/jellyfin.db`
- Accès root au LXC

---

## 1. Se connecter au conteneur Jellyfin

Depuis l’hôte Proxmox (PVE), en root :

```bash
pct enter <CTID_JELLYFIN>
```

(Le prompt devient alors `root@jellyfin:/#`.)

Vérifier que Jellyfin tourne et écoute sur le port 8096 :

```bash
systemctl status jellyfin.service
ss -lntp | grep -E '8096|8920' || echo "Rien n'écoute sur 8096/8920"
```

---

## 2. Arrêter proprement le service Jellyfin

```bash
systemctl stop jellyfin.service
```

---

## 3. Sauvegarder la base de données Jellyfin

Se placer dans le répertoire des données :

```bash
cd /var/lib/jellyfin/data
ls jellyfin.db
```

Créer une sauvegarde horodatée :

```bash
cp jellyfin.db jellyfin.db.bak_$(date +%F_%H-%M-%S)
```

> En cas de problème, il suffira de restaurer :
>
> ```bash
> systemctl stop jellyfin.service
> cp jellyfin.db.bak_YYYY-MM-DD_HH-MM-SS jellyfin.db
> systemctl start jellyfin.service
> ```

---

## 4. Vérifier la liste des utilisateurs

Installer `sqlite3` si nécessaire :

```bash
apt update
apt install -y sqlite3
```

Lister les utilisateurs Jellyfin :

```bash
sqlite3 jellyfin.db "SELECT Id, Username FROM Users;"
```

Repérer l’utilisateur cible, par exemple : USER$.

---

## 5. Réinitialiser le mot de passe de l’utilisateur

Mettre le champ `Password` à `NULL` pour l’utilisateur :

```bash
sqlite3 jellyfin.db "UPDATE Users SET Password=NULL WHERE Username='USER$';"
```

Vérifier :

```bash
sqlite3 jellyfin.db "SELECT Username, Password FROM Users WHERE Username='USER$';"
```

Résultat attendu :

```text
USER$|
```

(Colonne `Password` vide)

---

## 6. Redémarrer Jellyfin

```bash
systemctl start jellyfin.service
systemctl status jellyfin.service
```

Le service doit repasser en `active (running)`.

---

## 7. Connexion et changement de mot de passe

1. Ouvrir l’interface Jellyfin :`http://<IP_LXC_JELLYFIN>:8096`
2. S’authentifier avec :
   - **Utilisateur** : USER$
   - **Mot de passe** : **laisser vide**
3. Une fois connecté, aller dans :
   - **Tableau de bord → Utilisateurs → USER$ → Mot de passe**
   - Définir un **nouveau mot de passe** sécurisé.

---

## 8. Récapitulatif des commandes utilisées

```bash
# Dans le LXC Jellyfin
systemctl stop jellyfin.service

cd /var/lib/jellyfin/data
cp jellyfin.db jellyfin.db.bak_$(date +%F_%H-%M-%S)

sqlite3 jellyfin.db "SELECT Id, Username FROM Users;"
sqlite3 jellyfin.db "UPDATE Users SET Password=NULL WHERE Username='USER$';"
sqlite3 jellyfin.db "SELECT Username, Password FROM Users WHERE Username='USER$';"

systemctl start jellyfin.service
systemctl status jellyfin.service
```

---

## Notes

- Cette méthode ne supprime **aucun utilisateur** ni profils.
- Toujours conserver au moins **une sauvegarde** de `jellyfin.db` avant modification.
- La procédure est réutilisable pour n’importe quel utilisateur en changeant simplement :
  ```sql
  WHERE Username='nom_utilisateur';
  ```
