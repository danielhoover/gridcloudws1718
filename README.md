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

1. Installieren der Pakete mit yum install Paketname - diese sind aber bereits alle schon installiert worden

2. ist nur eine Feststellung - da ist nicht wirklich was zu tun. Wenn man dem Link zu Dokumentation folgt erfährt man etwas über das was wir jetzt ausführen sollen.

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

3. Zunächst sollen wir ins Verzeichnis /etc/grid-services/ wechseln und dort die verfügbaren Services auflisten.

```sh
cd /etc/grid-services/
ll (oder ls -l)
```

Wir sehen jetzt symbolische Links. Offenbar erzeugt der globus-gatekeeper-admin symbolische Links auf Skripte.
Wir können diese entfernen (wie in der Angabe gefordert) indem wir folgendes ausführen:

```sh
globus-gatekeeper-admin -d jobmanager-fork-poll
```

Führt man ll jetzt aus, so gibt es außer dem Verzeichnis available nichts mehr zu sehen. Enablen wir den Service nun unter anderem Namen

```sh
globus-gatekeeper-admin -e jobmanager-fork-poll -n jobmanager
```

Jetzt fehlt uns noch ein weiterer symbolischer Link damit wir mit der Angabe übereinstimmen. Legen wir diesen nun an:

```sh
(Syntax: ln -s Quelle Ziel bzw. ln -s Linkname Verweis)
ln -s jobmanager jobmanager-fork
```

Punkt 4 - wir sollen prüfen ob die Datei /etc/grid-services/available/jobmanager-fork-poll etwas enhtält:

```sh
cat /etc/grid-services/available/jobmanager-fork-poll
```

Das Ergebnis ist das gleiche wie auf dem Übungsblatt:

```sh
stderr_log,local_cred - /usr/sbin/globus-job-manager globus-job-manager -conf /etc/globus/globus-gram-job-manager.conf -type fork
```

5. Nach diesen Vorbereitungen dürfen wir nun den GRAM Gatekeeper starten, den Status prüfen und sehen, dass auf Port 2119 gehorcht wird. Wenig spektakulär die Befehle:

```sh
service globus-gatekeeper start
service globus-gatekeeper status

(Result: globus-gatekeeper is running (pid=14708))

netstat -anp |grep 2119

(Result: tcp6       0      0 :::2119                 :::*                    LISTEN      14708/globus-gateke)
```

Soweit also offenbar alles in Ordnung.

Jetzt sollen wir in Punkt 6 mal einen Job ausführen. Dazu brauchen wir zunächst ein Zertifikat. Wechsel zum eigenen Benutzer:

```sh
su username
myproxy-logon -s hostname
globus-job-run hostname /usr/bin/whoami
(Result: username)
```

Das erscheint erstmal wenig spektakulär - ist es aber schon. Wir können jetzt über unser Grid beliebige Jobs laufen lassen. Anmerkung: das tool globus-job-run ermöglicht die Ausführung von Jobs über Kommandozeilenparameter. Bei globusrun wird RSL verwendet.

Jetzt sollen wir noch die Beispiele der Dokumentation ausführen:

```sh
globus-job-run hostname /bin/hostname
globus-job-run hostname -np 4 /bin/hostname
globus-job-run hostname -stdin /tmp/testinput.txt /bin/cat
globus-job-run hostname -stdin -s /tmp/testinput.txt /bin/cat
globus-job-run hostname /bin/sleep 90
globus-job-run hostname -env TEST=1 -env GRID=1 /usr/bin/env
```

Jetzt wird es etwas lustiger. Wir sollen nun globus-job-submit verwenden:

```sh
globus-job-submit hostname /bin/hostname
```

Als Result erhalten wir eine URL (die merken - verwende ich jetzt exemplarisch):

```sh
https://hostname:52668/16650265051858224281/1801757980518004640/
```

Die nutzen wir jetzt um nach dem Status zu fragen:

```sh
globus-job-status https://hostname:52668/16650265051858224281/1801757980518004640/
```

(Ich bin nicht schnell genug mit dem Tippen von obiger Zeile um da PENDING oder ACTIVE zu erhalten. Ich habe als Status lediglich DONE erhalten.)

```sh
globus-job-get-output https://hostname:52668/16650265051858224281/1801757980518004640/

globus-job-clean -r hostname https://hostname:52668/16650265051858224281/1801757980518004640/
```

### globusrun

1. Aus der Dokumentation die Beispiele 2.12 und 2.13

-r steht für "Resource" - also worauf soll der Job laufen?

```sh
(geht) globusrun -a -r hostname
(geht) globusrun -a -r hostname/jobmanager
(fehler) globusrun -a -r hostname/jobmanager-doesntexist
(fehler) globusrun -j -r hostname/jobmanager:host@someother.host.example.org
(geht) globusrun -j -r hostname/jobmanager:host@hostname
```

2.

globusrun -s -r hostname/jobmanager "&(executable=/bin/hostname)(count=5)"

globusrun -b -r pcgridp19.gridp.lab.nm.ifi.lmu.de./jobmanager "&(executable=/bin/sleep)(arguments=500)"

hier kommt eine URL zurück die wir uns wieder merken müssen
https://hostname:48937/16650268353891826901/1801757980518054278/

globusrun -status https://hostname:48937/16650268353891826901/1801757980518054278/

ACTIVE

Jetzt killen wir den Job weil wir nicht wollen das das Ding 500 Sekunden schläft.

globusrun -k https://hostname:48937/16650268353891826901/1801757980518054278/

Erneutes Prüfen des Status

globusrun -status https://hostname:48937/16650268353891826901/1801757980518054278/

liefert DONE zurück.

Gut. Jetzt nutzen wir das erste Mal RSL in einer Datei. Dazu legen wir eine Datei an - eta rsltest.txt und schreiben das rein was auf dem Übungsblat zu finden ist:

```sh
&
(count=5)
(executable="/usr/bin/echo")
(arguments="Hello Globus")
```

Jetzt parsen wir das erstmal

```sh
globusrun -p -f ./rsltestjob.txt 
```

Alles ok. Jetzt können wir das ganze auf eine beliebige Art und Weise ausführen. Zum Beispiel direkt

```sh
globusrun -s -f ./rsltestjob.txt -r hostname/jobmanager
```

Oder als Batch (statt -s ein -b) mit entsprechender Prüfung des status (globusrun -status url oder globus-job-status url) und des results (globus-job-get-output) und einem clean.... globus-job-clean -r hostname url


