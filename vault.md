# Passwörter mit Ansible Vault verwalten

Alle Passwort-Variablen in `inventory/<env>/group_vars/all.yml` (aktuell mit
`"CHANGE-ME"` befüllt) dürfen nie im Klartext committet werden. Dieses Dokument
beschreibt, wie sie stattdessen per `ansible-vault` verschlüsselt werden und wie
die Playbooks anschließend damit aufgerufen werden.

## Betroffene Variablen

| Variable | Datei | Bedeutung |
| --- | --- | --- |
| `kafka_ssl_keystore_password` | kafka | Passwort des Kafka-Keystores |
| `kafka_ssl_key_password` | kafka | Passwort des Private-Keys im Keystore |
| `kafka_ssl_truststore_password` | kafka | Passwort des Kafka-Truststores |
| `zookeeper_ssl_keystore_password` | zookeeper | Passwort des ZooKeeper-Keystores |
| `zookeeper_ssl_truststore_password` | zookeeper | Passwort des ZooKeeper-Truststores |
| `kafka_inter_broker_password` | kafka | Passwort des Inter-Broker-SASL-Users (`kafka_inter_broker_username`) |
| `kafka_jaas_users` | kafka | Dict weiterer SASL-Client-User → Passwort (Producer/Consumer/Monitoring) |
| `zookeeper_kafka_sasl_password` | zookeeper | Passwort, mit dem sich der Kafka-Broker gegen ZooKeeper authentifiziert |

Jede der vier Umgebungen (`inventory/test`, `inventory/atu`, `inventory/entw`,
`inventory/prod`) hat ihre eigenen Werte und braucht daher auch ihre eigene
Vault-Verschlüsselung.

## Empfohlenes Vorgehen: separate `vault.yml` je Umgebung

Statt einzelne Werte inline in `all.yml` zu verschlüsseln (siehe
[Alternative](#alternative-einzelne-werte-inline-verschlüsseln) unten), ist es
übersichtlicher, `group_vars/all.yml` in ein Verzeichnis mit zwei Dateien
umzuwandeln – eine unverschlüsselte mit Verweisen und eine komplett
verschlüsselte mit den echten Passwörtern:

```
inventory/<env>/group_vars/all/vars.yml    # unverschlüsselt, wie bisher
inventory/<env>/group_vars/all/vault.yml   # komplett mit ansible-vault verschlüsselt
```

Ansible lädt bei einem Verzeichnis `group_vars/all/` automatisch alle darin
liegenden Dateien, daher genügt es, `all.yml` in diesen Ordner mit zwei Dateien
aufzuteilen.

### 1. `vars.yml` anpassen

In `vars.yml` bleiben alle unkritischen Werte wie bisher stehen. Die
Passwort-Variablen werden durch `vault_`-Referenzen ersetzt:

```yaml
# inventory/test/group_vars/all/vars.yml
kafka_ssl_keystore_password: "{{ vault_kafka_ssl_keystore_password }}"
kafka_ssl_key_password: "{{ vault_kafka_ssl_key_password }}"
kafka_ssl_truststore_password: "{{ vault_kafka_ssl_truststore_password }}"

zookeeper_ssl_keystore_password: "{{ vault_zookeeper_ssl_keystore_password }}"
zookeeper_ssl_truststore_password: "{{ vault_zookeeper_ssl_truststore_password }}"

kafka_inter_broker_password: "{{ vault_kafka_inter_broker_password }}"
kafka_jaas_users:
  monitoring: "{{ vault_kafka_jaas_password_monitoring }}"
  client: "{{ vault_kafka_jaas_password_client }}"

zookeeper_kafka_sasl_password: "{{ vault_zookeeper_kafka_sasl_password }}"
```

### 2. `vault.yml` anlegen und mit den echten Passwörtern befüllen

```bash
ansible-vault create inventory/test/group_vars/all/vault.yml
```

Öffnet den Editor (`$EDITOR`) mit einer leeren, zu verschlüsselnden Datei.
Inhalt:

```yaml
vault_kafka_ssl_keystore_password: "ein-echtes-passwort"
vault_kafka_ssl_key_password: "ein-echtes-passwort"
vault_kafka_ssl_truststore_password: "ein-echtes-passwort"

vault_zookeeper_ssl_keystore_password: "ein-echtes-passwort"
vault_zookeeper_ssl_truststore_password: "ein-echtes-passwort"

vault_kafka_inter_broker_password: "ein-echtes-passwort"
vault_kafka_jaas_password_monitoring: "ein-echtes-passwort"
vault_kafka_jaas_password_client: "ein-echtes-passwort"

vault_zookeeper_kafka_sasl_password: "ein-echtes-passwort"
```

Nach dem Speichern liegt die Datei komplett verschlüsselt auf der Platte und
kann gefahrlos committet werden. Diesen Schritt für jede Umgebung wiederholen
(`inventory/atu/...`, `inventory/entw/...`, `inventory/prod/...`).

### Vorhandene `vault.yml` bearbeiten

```bash
ansible-vault edit inventory/test/group_vars/all/vault.yml
```

### Vault-Passwort ändern (Rekey)

```bash
ansible-vault rekey inventory/test/group_vars/all/vault.yml
```

## Alternative: einzelne Werte inline verschlüsseln

Wenn keine Trennung `vars.yml`/`vault.yml` gewünscht ist, können einzelne Werte
direkt in `group_vars/all.yml` verschlüsselt werden, der Rest der Datei bleibt
Klartext (so wie bereits in der README beschrieben):

```bash
ansible-vault encrypt_string 'ein-echtes-passwort' --name 'kafka_inter_broker_password'
```

Die Ausgabe (ein `!vault |`-Block) ersetzt dann den `"CHANGE-ME"`-Wert 1:1 in
`inventory/<env>/group_vars/all.yml`, z. B.:

```yaml
kafka_inter_broker_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          66386439653...
```

Das funktioniert für jede der oben gelisteten Variablen einzeln, auch für die
Werte im `kafka_jaas_users`-Dict (Name entsprechend anpassen, z. B.
`--name 'dummy'` und den Key danach manuell in `kafka_jaas_users:` einsetzen,
da `encrypt_string` keine verschachtelten Dict-Keys kennt).

Vorteil: kleinere Diffs, keine neue Dateistruktur. Nachteil: Datei bleibt ein
Mix aus Klartext und Vault-Blöcken, schwerer auf einen Blick zu prüfen, ob alle
Geheimnisse erfasst sind – deshalb oben die separate `vault.yml` als
Empfehlung.

## Vault-Passwort bereitstellen

Für beide Varianten braucht Ansible beim Playbook-Lauf Zugriff auf das
Vault-Passwort. Zwei Möglichkeiten:

### a) Interaktiv abfragen

```bash
ansible-playbook -i inventory/test/hosts.ini install-zookeeper.yml --ask-vault-pass
ansible-playbook -i inventory/test/hosts.ini install-kafka.yml --ask-vault-pass
```

### b) Passwort-Datei (z. B. für CI/CD)

```bash
echo 'das-vault-master-passwort' > ~/.vault_pass_kafka-install.txt
chmod 600 ~/.vault_pass_kafka-install.txt
```

```bash
ansible-playbook -i inventory/test/hosts.ini install-zookeeper.yml \
  --vault-password-file ~/.vault_pass_kafka-install.txt
ansible-playbook -i inventory/test/hosts.ini install-kafka.yml \
  --vault-password-file ~/.vault_pass_kafka-install.txt
```

**Wichtig:** Diese Passwort-Datei niemals im Repo ablegen bzw. committen. Sie
liegt außerhalb von `kafka-install/` oder wird über `.gitignore`
ausgeschlossen; in CI/CD kommt sie aus einem Secret-Store (Vault, Jenkins
Credentials, GitLab CI Variable etc.) und wird zur Laufzeit erzeugt.

Alternativ kann das Passwort in `ansible.cfg` bzw. per Umgebungsvariable
hinterlegt werden, um `--vault-password-file` nicht bei jedem Aufruf angeben
zu müssen:

```ini
# ansible.cfg
[defaults]
vault_password_file = ~/.vault_pass_kafka-install.txt
```

oder:

```bash
export ANSIBLE_VAULT_PASSWORD_FILE=~/.vault_pass_kafka-install.txt
```

## Kompletter Ablauf pro Umgebung

```bash
# 1. Vault-Datei für die Umgebung anlegen/befüllen (einmalig bzw. bei Passwort-Änderung)
ansible-vault create inventory/prod/group_vars/all/vault.yml

# 2. ZooKeeper installieren
ansible-playbook -i inventory/prod/hosts.ini install-zookeeper.yml --ask-vault-pass

# 3. Kafka installieren
ansible-playbook -i inventory/prod/hosts.ini install-kafka.yml --ask-vault-pass
```

## Hinweise

- `no_log: true` ist in `install-kafka.yml` (JAAS-Rendering) und
  `install-zookeeper.yml` (SASL-User-Fact, JAAS-Rendering) bereits gesetzt –
  die Passwörter tauchen dadurch nicht im Ansible-Output/-Log auf, auch wenn
  sie aus der Vault entschlüsselt wurden.
- Verschlüsselte `vault.yml`-Dateien können bedenkenlos ins Git-Repo
  committet werden – der Inhalt ist ohne Vault-Passwort nicht lesbar. Nur das
  Vault-Passwort selbst (bzw. die `--vault-password-file`) darf niemals ins
  Repo.
- Bei mehreren Teams/Umgebungen mit unterschiedlichen Zugriffsrechten kann
  pro Umgebung ein eigenes Vault-Passwort verwendet werden (z. B. `prod`
  getrennt von `test`/`atu`/`entw`), damit nicht jeder mit Zugriff auf ein
  Test-Vault-Passwort auch die Prod-Passwörter entschlüsseln kann.
- Passwort-Rotation: neuen Wert in der jeweiligen `vault.yml` setzen
  (`ansible-vault edit ...`) und Playbook erneut laufen lassen – die
  `template`-Tasks für `kafka_server_jaas.conf` / `zookeeper_jaas.conf` /
  `server.properties` / `zoo.cfg` lösen bei Änderung automatisch den
  `notify`-Handler aus und starten den jeweiligen Service neu.
