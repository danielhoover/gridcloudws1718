# gridcloudws1718

Nachfolgend die Installationsschritte für die Virtuelle Maschine in der Vorlesung Grid und Cloud Computing an der LMU im Wintersemester 2017/2018

## Übungblatt 2

Login auf die Virtuelle Maschine mit

```sh
ssh root@IPAdresse
```
Die IP Adresse und das Passwort für die VM findet sich in der Email.

Jetzt müssen 2 Dinge gemacht werden:

1. Benutzer anlegen und ein Passwort vergeben

```sh
useradd username
passwd username
```

2. Für jeden angelegten Nutzer eine Ordnerstruktur im Homeverzeichnis anlegen und dem Nutzer die Rechte auf die Verzeichnisse geben

```sh
cd /home/username/
mkdir cert
mkdir .globus
chown username.username cert
chown username.username .globus
```

Im weiteren Verlauf des Übungsblattes ist von einem Bug die Rede. Um diesem Bug Rechnung zu tragen muss zunächst eine Ordnersturkur angelegt werden:

```sh
mkdir /etc/zypp/
mkdir /etc/zypp/repos.d/
```

Jetzt laden wir die RPM Pakete herunter und installieren beide anschließend. Am besten wechseln wir hierfür in das Homeverzeichnis des Benutzers root:

```sh
cd
wget http://scientificlinux.physik.uni-muenchen.de/mirror/epel/7/x86_64/Packages/e/epel-release-7-11.noarch.rpm
wget http://www.globus.org/ftppub/gt6/installers/repo/globus-toolkit-repo-latest.noarch.rpm
rpm -UvH epel-release-7-11.noarch.rpm
rpm -Uvh globus-toolkit-repo-latest.noarch.rpm
```
Abschließend müssen wegen des Bugs die Repos in das richtige Verzeichnis kopiert werden:

```sh
cp /etc/zypp/repos.d/* /etc/yum.repos.d/
```
Nachdem jetzt die richtigen Repositories bekannt sind können mit yum install alle erforderlichen Pakete installiert werden. Entweder einzeln nacheinander oder in einem kompletten Befehl:

Einzeln:

```sh
yum install globus-gridftp 
yum install globus-gram5
yum install globus-gsi
[...]
yum install myproxy-admin
```

Kompletter Befehl:

```sh
yum install globus-gridftp globus-gram5 globus-gsi globus-data-management-server globus-data-management-client globus-resource-management-server globus-resource-management-client myproxy myproxy-server myproxy-admin
```

## Übungblatt 4

Ins Verzeichnis /etc/grid-security wechseln. Dort finden sich die hostcert.pem und hostkey.pem. (Auflisten von Dateien mittels ls -l oder dem Alias ll)

```sh
cd /etc/grid-security/
ls -l (alternativ: ll)
```

Wir sollen nun erstmal sicherstellen, dass die Zertifikate Valide sind. Das geht grundsätzlich mit dem Befehl openssl verify

```sh
openssl verify -CApath /etc/grid-security/certificates/ -purpose sslserver /etc/grid-security/hostcert.pem
```
Nachdem alles ok ist können wir nun die Zertifikate installieren. (Im Prinzip kopieren und entsprechende Rechte vergeben - hierfür gibt es einen Shortcut den wir praktischerweise nutzen:)

```sh
install -o myproxy -m 644 /etc/grid-security/hostcert.pem /etc/grid-security/myproxy/hostcert.pem
install -o myproxy -m 600 /etc/grid-security/hostkey.pem /etc/grid-security/myproxy/hostkey.pem
```

Mit einem beliebigen Texteditor der Wahl muss nun die Datei /etc/myproxy-server.config editiert werden. Ich nutze gerne Joe weil ich mit vi oder vim nicht zurechtkomme. Sollte der fehlen schafft (yum install joe) Abhilfe.

Editieren des Files dann mit
```sh
joe /path/to/file
```

In unserem Beispiel

```sh
joe /etc/myproxy-server.config
```

In der Sektion #1 alle Kommentare entfernen. Speichern+Beenden mit joe über CTRL + K + X

Jetzt muss der Nutzer myproxy der Gruppe simpleca hinzugefügt werden. (damit anschließend der Befehl myproxy-admin-adduser funktioniert)

```sh
usermod -a -G simpleca myproxy
```

Wir sollen jetzt den myproxy-server starten, den Status prüfen und zusätzlich überprüfen ob der auf auf einem gewissen Port horcht.

```sh
service myproxy-server start
service myproxy-server status
netstat -anp |grep 7512
```

(falls netstat nicht geht: yum install net-tools)

Nachdem der Nutzer myproxy in der /etc/passwd am Ende /sbin/nologin eingetragen hat, kann nicht per su zu diesem Nutzer gewechselt werden. Zwei Lösungsmöglichkeiten: A. Beim wechsel mit su eine Bash mit angeben. B. Dauerhaft in der /etc/passwd eine Shell eintragen.

Möglichkeit A:

```sh
su –s /bin/bash myproxy
```

Möglichkeit B:
```sh
chsh -s /bin/bash myproxy
su myproxy
```

```
whoami
```

Sollte jetzt myproxy sein.

Für jeden Nutzer der später das Grid benutzen können soll müssen wir nun myproxy-admin-adduser ausführen. Wichtig: merken wie der distinguished Name lautet der vom Befehl myproxy-admin-adduser im Erfolgsfall zurückgegeben wird. (am besten kopieren)

```sh
myproxy-admin-adduser -l username -c "Full Name"
```

Das Fullsubject (distunguished Name) sieht zum Beispiel wie folgt aus:

```sh
/O=Grid/OU=GlobusTest/OU=simpleCA-gridsl/OU=Globus Simple CA/CN=Example User
```

Wichtig ist dabei der Teil nach OU=simpleCA- bis zum / also folgender:

/O=Grid/OU=GlobusTest/OU=simpleCA-**gridsl**/OU=Globus Simple CA/CN=Example User

Den brauchen wir später, da das unser hostname ist! Also irgendwie merken. (am besten kopieren oder notieren)

Jetzt müssen wir wieder root werden. Es reicht wahrscheinlich ein exit:
```sh
exit
```

Jetzt fügen wir den Nutzer unserem gridmapfile hinzu:

```sh
grid-mapfile-add-entry -dn "/O=Grid/OU=GlobusTest/OU=simpleCA-gridsl/OU=Globus Simple CA/CN=Example User" -ln username
```

Das muss für jeden Nutzer gemacht werden. Anschließend starten wir (sofern er nicht laufen sollte) den gridftp server

```sh
service globus-gridftp-server start
```

Für den folgenden Kopiertest wechseln wir zu einem beliebigen Nutzerkonto:

```sh
su username
```

Hilfreich zu wissen wäre wie der hostname so aussieht. Wir erinnern uns vorher haben wir uns den merken müssen. Gut, dass wir diesen noch haben:

```sh
myproxy-logon -s gridsl
globus-url-copy gsiftp://gridsl/etc/group /tmp/copytest
```
gridsl steht hier für den vollen hostname. Der muss korrekt sein sonst gibt es Fehlermeldungen!

## Übungsblatt 6

globus-gatekeeper-admin -l

Listet alle Skripte auf die enabled sind.

Weitere kann man wie folgt enablen:

```sh
globus-gatekeeper-admin [-e scriptname -n shortaliasname]
```

Disable eines Skriptes mit:

```sh
globus-gatekeeper-admin -d scriptname
```

