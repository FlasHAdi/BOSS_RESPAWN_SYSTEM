# Boss Respawn System - Sistem de Respawnare Sefi dupa Mentenanta

## Descriere
Sistem automat de salvare si respawnare a sefilor importanti dupa restart/mentenanta.
- Salveaza pozitiile sefilor in baza de date
- Suporta canale multiple
- Respawnare automata la pornirea serverului
- Comenzi GM pentru management usor

## Instalare

### 1. Baza de Date
Ruleaza fisierul SQL pentru a crea tabelul:

### 2. Compilare
Compileaza game folosind:
```bash
cd game/src
gmake clean
gmake -j4
```

## Configurare

### Adaugare Sefi in Lista de Tracking

#### Metoda 1: Prin Comanda GM (Recomandat)
```
/boss_track_add <vnum>
```
Exemplu pentru sefi comuni:
- `/boss_track_add 2493` - Beran-Setaou
- `/boss_track_add 6091` - Razador
- `/boss_track_add 2597` - Charon
- `/boss_track_add 2191` - Testoasa Gigant
- `/boss_track_add 2492` - General Yonghan

#### Metoda 2: (Automat la boot)
Modifica in `game/src/input_db.cpp` dupa LoadBossRespawnData():
```cpp
// Adauga automat sefii importanti la tracking
CMobManager::instance().AddTrackedBossVnum(2493); // Beran-Setaou
CMobManager::instance().AddTrackedBossVnum(6091); // Razador
CMobManager::instance().AddTrackedBossVnum(2597); // Charon
CMobManager::instance().AddTrackedBossVnum(2191); // Testoasa Gigant
CMobManager::instance().AddTrackedBossVnum(2492); // General Yonghan
// Adauga aici alti sefi...
```

## Comenzi GM

### `/boss_track_add <vnum>`
Adauga un boss in lista de tracking.
- **Nivel GM:** IMPLEMENTOR
- **Exemplu:** `/boss_track_add 2493`

### `/boss_track_remove <vnum>`
Elimina un boss din lista de tracking.
- **Nivel GM:** IMPLEMENTOR
- **Exemplu:** `/boss_track_remove 2493`

### `/boss_track_list`
Afiseaza lista tuturor sefilor urmariti.
- **Nivel GM:** LOW_WIZARD
- **Exemplu:** `/boss_track_list`

### `/boss_respawn_reload`
Reincarca datele din baza de date si respawneaza sefii salvati.
- **Nivel GM:** IMPLEMENTOR
- **Utilizare:** Dupa o eroare sau pentru testare
- **Exemplu:** `/boss_respawn_reload`

### `/boss_respawn_clear`
Sterge toate datele de respawn din baza de date pentru canalul curent.
- **Nivel GM:** IMPLEMENTOR
- **Atentie:** Aceasta comanda sterge permanent datele!
- **Exemplu:** `/boss_respawn_clear`

## Functionare

### La Spawn-ul Sefilor
Cand un boss din lista de tracking este spawnat:
1. Sistemul inregistreaza automat pozitia (X, Y)
2. Salveaza map_index si canalul
3. Scrie in baza de date
4. **Suporta multiple instante** - acelasi boss poate fi salvat in mai multe locatii diferite
### La Moartea Sefilor
Cand un boss moare:
1. Sistemul elimina intrarea din baza de date
2. Opreste tracking-ul pentru acel boss pana la urmatorul spawn

### La Repornirea Serverului
La boot-ul serverului:
1. Se incarca toate datele de respawn pentru canalul curent
2. Sefii sunt respawnati automat la pozitiile salvate
3. Tracking-ul continua pentru noii sefi


**Constraint Important:** `unique_boss_position` permite salvarea aceluiasi boss (acelasi VNUM) in **multiple locatii diferite**. Fiecare pozitie unica (x, y) pe aceeasi harta si canal este tratata ca o instanta separata.

## Exemple de Utilizare

### Exemplu 1: Adaugare Boss Nou
```
GM: /boss_track_add 6091
Server: Boss VNUM 6091 (Razador) added to tracking list.

[Boss-ul este spawnat in joc]
Server Log: BOSS_RESPAWN: Registered boss vnum=6091, vid=150001, map=351, pos=(1234,5678), channel=1
Server Log: BOSS_RESPAWN: Saved boss data vnum=6091...
```

### Exemplu 2: Dupa Restart Server
```
Server Log: BOOT: Loading boss respawn system...
Server Log: BOSS_RESPAWN: Loading boss respawn data from database for channel 1...
Server Log: BOSS_RESPAWN: Loaded boss vnum=6091, map=351, pos=(1234,5678), channel=1
Server Log: BOSS_RESPAWN: Loaded 1 boss(es) for channel 1
Server Log: BOSS_RESPAWN: Starting respawn of 1 boss(es) on channel 1...
Server Log: BOSS_RESPAWN: Respawned boss vnum=6091 at map=351, pos=(1234,5678), channel=1 [NEW VID=150002]
Server Log: BOSS_RESPAWN: Successfully respawned 1/1 boss(es) on channel 1
```

### Exemplu 3: Verificare Sefi Urmariti
```
GM: /boss_track_list
Server: === Tracked Boss VNUMs ===
Server: - VNUM 2191: Beran-Setaou
Server: - VNUM 2493: Dragon Lacau
Server: - VNUM 6091: Razador
Server: - VNUM 8057: Grotto Orc Boss
Server: Total: 4 boss(es)
```
### Exemplu 4: Multiple Instante Acelasi Boss
```
# Spawn acelasi boss in 3 locatii diferite
GM: /mob 6091   # Locatia 1
GM: /warp 100 200
GM: /mob 6091   # Locatia 2
GM: /warp 300 400
GM: /mob 6091   # Locatia 3

# Verificare in DB
SELECT mob_vnum, x, y FROM boss_respawn_data WHERE mob_vnum=6091;
+----------+--------+--------+
| mob_vnum | x      | y      |
+----------+--------+--------+
|     6091 | 123450 | 678900 |
|     6091 | 234560 | 789010 |
|     6091 | 345670 | 890120 |
+----------+--------+--------+

# Dupa restart, toate 3 instante vor fi respawnate in locatiile lor
```
## Logging

Toate actiunile sistemului sunt inregistrate in syslog cu prefixul `BOSS_RESPAWN`:
- Inregistrari de sefi
- Salvari in baza de date
- Incarcari la boot
- Respawn-uri

## Compatibilitate cu Multiple Canale

Sistemul functioneaza independent pe fiecare canal:
- Fiecare canal isi salveaza propriii sefi
- La boot, se incarca doar sefii pentru canalul respectiv
- Comenzile GM opereaza doar pe canalul curent

## Note Importante

1. **Performanta:** Sistemul foloseste query-uri optimizate cu indexuri
2. **Siguranta:** Datele sunt salvate cu `ON DUPLICATE KEY UPDATE`
3. **Cleanup:** Datele sunt sterse automat cand sefii mor

## Debugging

Pentru a activa logging-ul detaliat, verifica syslog:
```bash
tail -f /usr/metin2/log/syslog | grep BOSS_RESPAWN
```

## Extindere

Pentru a adauga functionalitati noi:
1. Modifica structura `TBossRespawnData` in `mob_manager.h`
2. Actualizeaza functiile de salvare/incarcare in `mob_manager.cpp`
3. Adauga coloane noi in tabelul SQL daca este necesar

## Troubleshooting

### Sefii nu se respawneaza dupa restart
- Verifica daca tabelul SQL exista si are date: `SELECT * FROM boss_respawn_data;`
- Verifica syslog-ul pentru erori
- Ruleaza `/boss_respawn_reload` ca test

### Sefii se respawneaza de multiple ori
- Verifica daca exista duplicate in baza de date
- Ruleaza `/boss_respawn_clear` si readauga sefii

### Date vechi in baza de date
- Ruleaza periodic: `DELETE FROM boss_respawn_data WHERE last_update < DATE_SUB(NOW(), INTERVAL 30 DAY);`

## Suport

Pentru probleme sau intrebari:
- Verifica logging-ul in syslog
- Testeaza cu `/boss_respawn_reload`
- Asigura-te ca toate fisierele au fost compilate corect

---
**Versiune:** 1.0.0
**Ultima actualizare:** 2026
