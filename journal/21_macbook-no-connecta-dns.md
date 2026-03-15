# MacBook no es connecta amb el DNS

**Data:** 10 de març de 2026

---

## Problema

El MacBook no resolvia els dominis del lab (`*.devops.net`, `*.okd.devops.net`).

---

## Diagnòstic

### 1. Verificar quin DNS usa el MacBook

```bash
scutil --dns | grep nameserver
```

**Resultat:**

```
nameserver[0] : 192.168.1.2
nameserver[1] : 8.8.8.8
nameserver[2] : 1.1.1.1
```

✅ El DNS estava ben configurat. El MacBook ja apuntava al Synology (`192.168.1.2`).

### 2. Verificar resolució DNS directa al Synology

```bash
nslookup api.okd.devops.net 192.168.1.2
```

**Resultat:**

```
Server:         192.168.1.2
Address:        192.168.1.2#53

Name:   api.okd.devops.net
Address: 192.168.1.4
```

✅ Resol correctament a `192.168.1.4`.

---

## Causa arrel

El MacBook estava connectat pel **USB-C Dock Ethernet**, però el DNS `192.168.1.2` només estava configurat a la interfície **Wi-Fi**.

- `Wi-Fi` → DNS `192.168.1.2` ✅ (correcte però no usada)
- `USB-C Dock Ethernet` → DNS del router `192.168.1.1` ❌ (no coneix `devops.net`)

macOS usa la interfície amb prioritat més alta. El Dock Ethernet apareix primer a la llista de serveis, per tant les seves consultes DNS anaven al router i no resolien els dominis del lab.

---

## Solució aplicada

```bash
sudo networksetup -setdnsservers "USB-C Dock Ethernet" 192.168.1.2 8.8.8.8
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

✅ `ping gitea.devops.net` resol correctament.

---

## Regla per al futur

**Sempre que el MacBook estigui connectat pel Dock**, cal que l'Ethernet del Dock tingui el DNS configurat. Aquesta configuració és persistent (sobreviu als reinicis).

Si en algun moment es perd (p. ex. canvi de xarxa), re-aplicar:

```bash
sudo networksetup -setdnsservers "USB-C Dock Ethernet" 192.168.1.2 8.8.8.8
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

---

## Estat final

| Interfície | DNS configurat | Funciona |
|------------|---------------|----------|
| Wi-Fi | `192.168.1.2`, `8.8.8.8` | ✅ |
| USB-C Dock Ethernet | `192.168.1.2`, `8.8.8.8` | ✅ |
| `ping gitea.devops.net` | — | ✅ |
| `nslookup api.okd.devops.net` | — | ✅ |

---

## Actualització — Chrome no resol, Safari sí

**Data:** 10 de març de 2026

Després d'aplicar la solució anterior, Safari resol correctament els dominis del lab, però **Chrome segueix sense funcionar**.

### Causa investigada

**DNS-over-HTTPS (DNS segur) a Chrome** → ~~desactivat~~ ja estava desactivat. Descartada com a causa.

### Altres causes possibles i diagnòstic

#### 1. Caché DNS interna de Chrome

Chrome manté una caché DNS pròpia independent del sistema. Pot tenir entrades antigues o errònies.

**Solució:** Accedeix a `chrome://net-internals/#dns` i fes clic a **"Clear host cache"**.

També útil per veure les entrades actuals i detectar si hi ha errors de resolució.

#### 2. Caché de sockets de Chrome

De vegades les connexions antigues interfereixen.

**Solució:** A `chrome://net-internals/#sockets` → **"Flush socket pools"**.

#### 3. Chrome no agafa els canvis DNS del sistema en calent

macOS pot trigar a propagar els canvis DNS a Chrome si ja estava obert quan es va fer la configuració.

**Solució:** Tancar Chrome completament (Cmd+Q, no només la finestra) i tornar-lo a obrir.

#### 4. Perfil de Chrome corrupte o extensions interferint

Una extensió de VPN, proxy o ad-blocker pot redirigir el DNS.

**Comprovació:** Prova en mode incògnit amb extensions desactivades: `Cmd + Shift + N`.

#### 5. Certificat SSL no reconegut (error de connexió, no de DNS)

Si el domini resol però el certificat és autosignat, Chrome mostra error de connexió que es pot confondre amb un error DNS.

**Comprovació:** Mira si Chrome mostra `ERR_NAME_NOT_RESOLVED` (problema DNS) o `NET::ERR_CERT_AUTHORITY_INVALID` (problema de certificat).

---

### Ordre de diagnòstic recomanat

1. Comprova l'error exacte a Chrome: `ERR_NAME_NOT_RESOLVED` o un altre
2. `chrome://net-internals/#dns` → Clear host cache
3. Tanca Chrome completament i torna a obrir
4. Prova en mode incògnit sense extensions
5. Si tot falla → problema de certificat, no de DNS

| Navegador | Resol `*.devops.net` |
|-----------|----------------------|
| Safari | ✅ |
| Chrome | ❌ (causa pendent de confirmar) |

---

## Anàlisi: `.net` vs `.local` vs altres TLDs privats

### Per què `.local` NO és una bona opció

`.local` és un domini **reservat per mDNS/Bonjour** (RFC 6762). En macOS:

- Les consultes `.local` **no van al servidor DNS** (Synology), sinó que es resolen per **multicast DNS** directament a la xarxa local.
- Això significa que el Synology DNS **no pot gestionar** dominis `.local` de manera fiable.
- Chrome i Safari tindrien el mateix problema o pitjor.

### Per què `.net` tampoc és ideal

`.net` és un TLD públic real. Quan el DNS segur de Chrome no troba resposta local, pot intentar consultar servidors externs, que evidentment no coneixen el teu `devops.net` privat.

### Recomanació: usar un TLD reservat per xarxes privades

| Domini | Recomanació |
|--------|-------------|
| `.local` | ❌ Reservat per mDNS/Bonjour, no usar per DNS clàssic |
| `.net` / `.com` | ⚠️ TLDs públics, poden generar conflictes |
| `.internal` | ✅ Reservat per ICANN per xarxes privades (recomanat) |
| `.lan` | ✅ Àmpliament usat per labs privats, no assignat públicament |
| `.lab` | ✅ No assignat públicament, popular en entorns de laboratori |
| `.home.arpa` | ✅ RFC 8375, estàndard per xarxes domèstiques |

**Opcions concretes per al nostre lab:**

```
# Opció A — .internal (recomanat per estàndard ICANN)
gitea.devops.internal
api.okd.devops.internal

# Opció B — .lab (semànticament clar per a un lab)
gitea.devops.lab
api.okd.devops.lab
```

`.lab` és una opció perfectament vàlida: no és un TLD públic assignat, és descriptiu i fàcil de recordar. La diferència respecte a `.internal` és purament de convenció — `.internal` té el suport explícit de l'ICANN mentre que `.lab` és ús per costum.

### ✅ Decisió: migrar a `devops.lan`

`devops.lan` és clar, curt i universalment entès com a xarxa local privada. No té conflictes amb TLDs públics ni amb mDNS.

**Nous dominis:**

```
# Serveis generals
gitea.devops.lan
traefik.devops.lan

# OKD
api.okd.devops.lan
console.okd.devops.lan
*.apps.okd.devops.lan
```

**Passos per a la migració:**

1. Actualitzar les zones DNS al Synology (`devops.net` → `devops.lan`)
2. Actualitzar la configuració de Traefik (routers i certificats)
3. Actualitzar les referències als serveis (OKD, Gitea, etc.)
4. Actualitzar el DNS del MacBook (si cal, ja apunta al Synology)
5. Flush caché DNS: `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`

> Veure document de migració per als detalls tècnics.
